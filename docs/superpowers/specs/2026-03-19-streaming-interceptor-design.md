# Streaming Interceptor for pi Extension System

**Date**: 2026-03-19
**Status**: Approved
**Scope**: pi-mono (3 files), tallow (0 files for V1)

## Problem

Streaming is opaque in the pi/tallow stack. Extensions can modify what goes into the model (context, system prompt) and block tool calls, but cannot intercept token-by-token output mid-stream. `turn_end` fires after generation completes. There is no way to filter, modify, or suppress Claude's text output before the user sees it.

## Requirements

- Extensions can buffer streaming tokens, then decide to pass, modify, suppress, or abort
- Extension controls flush timing (sentence boundary, token count, custom logic)
- Timeout safety valve prevents frozen UI if extension never flushes
- Existing observe-only `message_update` handlers remain unaffected
- Single interceptor in V1, API designed to not preclude future chaining
- Abort triggers stream kill and retry with reason injected via context

## Use Cases

- "Don't guess" enforcement: buffer sentences, detect hallucination patterns, abort and retry
- PII redaction: buffer chunks, redact sensitive content before TUI renders
- Output filtering: suppress unwanted content from display while preserving model context
- Streaming audit: observe tokens mid-stream (already possible, but now with interception option)

## Design

### Types

Added to `packages/coding-agent/src/core/extensions/types.ts`:

```typescript
/** Decision returned by a message_update interceptor after evaluating buffered tokens. */
export type StreamDecision =
  | { action: "pass" }
  | { action: "modify"; text: string }
  | { action: "suppress" }
  | { action: "abort"; reason: string };

/** Result type for message_update handlers that opt into interception. */
export type MessageUpdateEventResult = StreamDecision | void;
```

The `message_update` handler signature changes from:

```typescript
on(event: "message_update", handler: ExtensionHandler<MessageUpdateEvent>): void;
```

to:

```typescript
on(event: "message_update", handler: ExtensionHandler<MessageUpdateEvent, MessageUpdateEventResult>): void;
```

Handlers returning `void` remain pure observers (backwards compatible). Handlers returning a `StreamDecision` become interceptors.

### Buffer + Flush Logic in ExtensionRunner

New `emitMessageUpdate()` method on `ExtensionRunner` in `packages/coding-agent/src/core/extensions/runner.ts`, following the existing `emitToolResult()` pattern.

**Buffer state** (created on `message_start`, cleared on `message_end`):

```typescript
private _streamBuffer: {
  tokens: string;
  events: MessageUpdateEvent[];
  flushTimer: ReturnType<typeof setTimeout> | null;
  maxHoldMs: number;  // default 500ms
} | null = null;
```

**Flow per token:**

1. `emitMessageUpdate(event)` called from `_handleAgentEvent`
2. If no interceptor registered: call observers, return event unchanged (fast path)
3. If interceptor exists:
   - `text_delta` events: append delta to buffer, call interceptor with buffered state
   - Non-text events (`thinking_delta`, `toolcall_delta`): pass through immediately, no buffering
   - Interceptor returns:
     - `void` / `undefined`: hold (keep buffering)
     - `{ action: "pass" }`: flush buffer as-is to `_emit()`
     - `{ action: "modify", text }`: flush with rewritten delta to `_emit()`
     - `{ action: "suppress" }`: skip `_emit()`, clear buffer
     - `{ action: "abort", reason }`: call `abortFn()`, stash reason, clear buffer

4. **Timeout safety valve**: on first buffered token, start timer (`maxHoldMs`). If fired before interceptor flushes, auto-flush with `pass` and log warning. Timer resets on each flush.

**Return value**: `emitMessageUpdate()` returns `{ suppressed: boolean; aborted: boolean; modifiedEvent?: MessageUpdateEvent }` so `_handleAgentEvent` knows whether to call `_emit()`.

**Lifecycle**: Buffer created on first `message_update` after `message_start`, cleared on `message_end` (force-flushes remaining tokens).

### Wiring in AgentSession

`_handleAgentEvent` at `agent-session.ts` currently does:

```typescript
await this._emitExtensionEvent(event);  // observers see it
this._emit(event);                       // TUI sees it
```

For `message_update`, `_emitExtensionEvent` routes through `emitMessageUpdate()`:

```typescript
} else if (event.type === "message_update") {
    const extensionEvent: MessageUpdateEvent = {
        type: "message_update",
        message: event.message,
        assistantMessageEvent: event.assistantMessageEvent,
    };
    const result = await this._extensionRunner.emitMessageUpdate(extensionEvent);

    if (result.suppressed || result.aborted) {
        return;
    }
    if (result.modifiedEvent) {
        this._emit({
            ...event,
            assistantMessageEvent: result.modifiedEvent.assistantMessageEvent,
        });
        return;
    }
    // No interceptor or "pass" - fall through to normal _emit
}
```

### Abort + Retry Flow

1. Runner clears buffer, cancels flush timer, calls `abortFn()`
2. `abortFn()` calls `AgentSession.abort()` which kills model stream and waits for idle
3. Runner stashes reason: `_pendingAbortReason = reason`
4. Existing retry logic handles the aborted message
5. On retry, `emitContext()` injects stashed reason as system instruction, clears it
6. Model generates new response, interceptor runs again on the new stream
7. Retry limit: handled by existing `_retryAttempt` counter and `maxRetries`

Partial display: users may see already-flushed tokens before the abort. TUI already handles aborted messages (shows partial text then retry). No TUI changes needed.

### Extension Author API

```typescript
export default function myExtension(pi: ExtensionAPI) {
    let buffer = "";

    pi.on("message_update", async (event) => {
        if (event.assistantMessageEvent.type !== "text_delta") {
            return; // void = pass through non-text events
        }

        buffer += event.assistantMessageEvent.delta;

        // Not enough context yet
        if (!endsWithSentence(buffer)) {
            return; // void = keep buffering
        }

        const sentence = buffer;
        buffer = "";

        if (looksLikeGuessing(sentence)) {
            return { action: "abort", reason: "Model guessed. Retry with stronger instructions." };
        }

        if (containsPII(sentence)) {
            return { action: "modify", text: redact(sentence) };
        }

        return { action: "pass" };
    });

    pi.on("message_start", () => { buffer = ""; });
}
```

Returning `void` means "hold" (when buffering) or "observe only" (when no interception intended). Existing handlers returning void are unaffected.

V1: first extension returning a non-void `StreamDecision` is the interceptor. Others returning void remain observers.

Optional: `pi.setStreamInterceptorTimeout(ms)` to configure `maxHoldMs`.

### Performance

**No interceptor (hot path):** one boolean check (`_hasInterceptor`) per token above current baseline. Flag set on first `emitMessageUpdate()` call by scanning handlers.

**Interceptor active (warm path):** string append to buffer, one function call to interceptor, check return value. On flush: construct modified event, single `_emit()`. Buffered tokens coalesce into one delta, cheaper than per-token emit.

**Abort (cold path):** `abort()` + `waitForIdle()` + retry. Interceptor adds no meaningful overhead on top of existing retry cost.

**Memory:** buffer holds ~500ms of tokens (50-100 tokens, few hundred bytes). Cleared on every flush and on `message_end`.

## Files Changed

| File | Change |
|------|--------|
| `pi-mono: packages/coding-agent/src/core/extensions/types.ts` | Add `StreamDecision`, `MessageUpdateEventResult`. Update `message_update` handler signature. Remove `MessageUpdateEvent` from `RunnerEmitEvent`. |
| `pi-mono: packages/coding-agent/src/core/extensions/runner.ts` | Add `_streamBuffer`, `_hasInterceptor`, `_pendingAbortReason`. New `emitMessageUpdate()` method (~60-80 lines). Inject abort reason in `emitContext()`. Buffer lifecycle on `message_start`/`message_end`. |
| `pi-mono: packages/coding-agent/src/core/agent-session.ts` | Replace `emit()` with `emitMessageUpdate()` for `message_update` in `_emitExtensionEvent`. Handle suppressed/modified/aborted before `_emit()`. |

## Not Changed

- `packages/ai/` - provider layer untouched
- `packages/agent/` - agent loop and Agent class untouched
- Existing `subscribe()` listeners - see final (possibly modified) event
- Existing `message_update` observers returning void - no behavior change
- tallow - no changes for V1 (extensions use pi API directly)

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

/** Result returned by emitMessageUpdate() to the caller. */
export type EmitMessageUpdateResult =
  | { outcome: "emit"; event: MessageUpdateEvent }
  | { outcome: "emit_modified"; event: MessageUpdateEvent }
  | { outcome: "hold" }
  | { outcome: "suppressed" }
  | { outcome: "aborted" };
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

### AssistantMessageEvent Subtype Routing

`AssistantMessageEvent` has 12 subtypes. Each `message_update` wraps one. The interceptor only buffers `text_delta`; all others pass through immediately. Complete routing table:

| `assistantMessageEvent.type` | Behavior |
|------------------------------|----------|
| `text_delta` | **Buffered.** Appended to buffer, interceptor called. |
| `text_start` | Pass through immediately. No buffering. |
| `text_end` | **Force-flush** any buffered text first, then pass through. |
| `thinking_start` | Pass through immediately. |
| `thinking_delta` | Pass through immediately. |
| `thinking_end` | Pass through immediately. |
| `toolcall_start` | Pass through immediately. |
| `toolcall_delta` | Pass through immediately. |
| `toolcall_end` | Pass through immediately. |
| `start` | Pass through immediately. |
| `done` | **Force-flush** any buffered text first, then pass through. |
| `error` | **Force-flush** any buffered text first, then pass through. |

Force-flush on `text_end`, `done`, and `error` guarantees no tokens are lost at content block or stream boundaries. The force-flush uses `{ action: "pass" }` — the interceptor does not get a decision on forced flushes. Note: `done` carries `{ message: AssistantMessage }` and `error` carries `{ error: AssistantMessage }` — neither has `delta` or `partial`. Extension observers matching on the inner event type should guard accordingly.

Buffer cleanup also happens on `message_end`, which is an `AgentEvent` (not an `AssistantMessageEvent` subtype) and therefore not routed through `emitMessageUpdate()`. The `_handleAgentEvent` method calls `this._extensionRunner.flushAndClearBuffer()` when it sees `message_end`, before emitting the end event to extensions and subscribers. If `done`/`error` arrives after an abort (buffer already cleared), force-flush is a no-op — safe by construction since `_handleAgentEvent` serializes events.

### Buffer + Flush Logic in ExtensionRunner

New `emitMessageUpdate()` method on `ExtensionRunner` in `packages/coding-agent/src/core/extensions/runner.ts`, following the existing `emitToolResult()` pattern.

**Buffer state** (created lazily on first `text_delta` when an interceptor is registered, cleared on `message_end`):

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
2. If no interceptor registered: call observers, return `{ outcome: "emit", event }` (fast path)
3. If interceptor exists:
   - Check subtype routing table above. Non-buffered types: call observers, return `{ outcome: "emit", event }`
   - Force-flush types (`text_end`, `done`, `error`): if buffer has content, flush with `pass`, then return `{ outcome: "emit", event }`
   - `text_delta`: append delta to buffer, call interceptor with event
   - Interceptor returns:
     - `void` / `undefined`: hold (keep buffering), return `{ outcome: "hold" }`
     - `{ action: "pass" }`: flush buffer as-is, return `{ outcome: "emit", event: flushedEvent }`
     - `{ action: "modify", text }`: flush with rewritten delta, return `{ outcome: "emit_modified", event: modifiedEvent }`
     - `{ action: "suppress" }`: clear buffer, return `{ outcome: "suppressed" }`
     - `{ action: "abort", reason }`: clear buffer, initiate abort, return `{ outcome: "aborted" }`

4. **Timeout safety valve**: timer starts when the first `text_delta` is appended to an empty buffer. Timer counts down continuously and **only resets when the buffer is flushed** (by interceptor decision, force-flush, or timeout itself). New tokens arriving while the timer is running do not reset it. If the timer fires, auto-flush with `pass` and log a warning.

**Backpressure model**: `_handleAgentEvent` is async and the agent's `subscribe()` mechanism serializes event delivery — each handler call must resolve before the next event fires. When the interceptor holds tokens (returns `void`), subsequent `text_delta` events queue behind the previous `emitMessageUpdate()` call. This is expected and safe: the buffer absorbs the backpressure, and the timeout valve prevents indefinite stall. At typical streaming speeds (~60 tokens/sec, ~15ms between tokens), an interceptor that takes <15ms per call introduces no observable delay.

**Lifecycle**: Buffer created lazily on first `text_delta` with an active interceptor (not on `message_start`). Cleared on `message_end`, which force-flushes any remaining tokens before emitting the end event.

### `modify` and `suppress` Semantics

**`modify`**: When the interceptor returns `{ action: "modify", text }`, the runner synthesizes a new `text_delta` event with `delta` set to the modified text. The `partial: AssistantMessage` field is **not patched** — it reflects the model's actual output, not the modified text. This is a deliberate trade-off: patching `partial` would require deep-cloning and rewriting the `AssistantMessage.content` array, which is expensive per-flush and fragile. Downstream `subscribe()` listeners that use `partial` for rendering will see the original model text. The TUI in tallow renders from deltas, not from `partial`, so the modification is visible to the user. Extensions that need the modified text should track it themselves.

**`suppress`**: When the interceptor returns `{ action: "suppress" }`, the buffer is cleared and `_emit()` is not called. The suppressed tokens remain in `partial: AssistantMessage` and in `context.messages` — the model remembers them on subsequent turns. This is by design (see "Abort + Retry Flow" for the escalation path when context divergence is unacceptable). Known limitation: any `subscribe()` listener that reconstructs display from `partial.content` rather than accumulated deltas may show suppressed text.

### Wiring in AgentSession

The interception logic lives in `_handleAgentEvent` (not in `_emitExtensionEvent`), as a special case before the unconditional `_emit()` at line 341. This follows the pattern of existing special handling in `_handleAgentEvent` (e.g., steering queue cleanup at lines 320-335).

```typescript
// In _handleAgentEvent, BEFORE the existing lines 337-341:
if (event.type === "message_update") {
    const extensionEvent: MessageUpdateEvent = {
        type: "message_update",
        message: event.message,
        assistantMessageEvent: event.assistantMessageEvent,
    };
    const result = await this._extensionRunner.emitMessageUpdate(extensionEvent);

    switch (result.outcome) {
        case "hold":
        case "suppressed":
        case "aborted":
            return; // do not call _emit
        case "emit":
            this._emit(event);
            return;
        case "emit_modified":
            this._emit({
                ...event,
                assistantMessageEvent: result.event.assistantMessageEvent,
            });
            return;
    }
}

// Existing code for all other event types:
await this._emitExtensionEvent(event);
this._emit(event);
```

`_emitExtensionEvent` no longer handles `message_update` — it is routed entirely through `emitMessageUpdate()` which calls both observers and the interceptor.

### Abort + Retry Flow

The existing `_isRetryableError` only handles API errors (overloaded, rate limit, 500s). An interceptor abort sets `stopReason: "aborted"`, which is not matched by `_isRetryableError`. A new retry path is needed.

1. Runner clears buffer, cancels flush timer, calls `abortFn()`
2. `abortFn()` calls `AgentSession.abort()` which kills model stream and waits for idle
3. Runner stashes reason: `_pendingAbortReason = reason`
4. **New**: `_handleAgentEvent` checks `this._extensionRunner.consumePendingAbortReason()` on `agent_end`. The runner exposes this as a public method that returns the stashed reason and clears it (returns `string | undefined`). If a reason is returned, the session initiates a retry by calling `this.agent.continue()` (the same mechanism used by the existing retry system after `_handleRetryableError` resolves). This bypasses `_isRetryableError` entirely — the retry is triggered by the interceptor, not by an API error.
5. On the retry's `context` event, `emitContext()` receives the reason via a second field: `_pendingContextInjection`. This is set by `_handleAgentEvent` at the same time it consumes the abort reason, keeping the cross-boundary API to two methods: `consumePendingAbortReason()` and `injectContextMessage(text: string)`. In `emitContext()`, if `_pendingContextInjection` is set, it appends the injection as a system instruction and clears it.
6. Model generates new response, interceptor runs again on the new stream
7. **Retry limit**: new counter `_interceptorAbortCount`, incremented on each interceptor abort, reset on successful `message_end`. Capped at 3 (configurable). If exceeded, the abort is downgraded to `suppress` and a warning is logged. This prevents infinite abort loops independently of the existing `_retryAttempt` counter.

Partial display: users may see already-flushed tokens before the abort. TUI already handles aborted messages (shows partial text then retry). No TUI changes needed.

### Extension Author API

```typescript
export default function myExtension(pi: ExtensionAPI) {
    let buffer = "";

    pi.on("message_update", async (event) => {
        // Only intercept text deltas
        if (event.assistantMessageEvent.type !== "text_delta") {
            return; // void = observe only, pass through
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

    // Reset buffer state between messages
    pi.on("message_start", () => { buffer = ""; });
}
```

**Key behaviors:**
- Returning `void` means "hold" (when the interceptor is buffering) or "observe only" (when no interception is intended). Existing handlers returning void are unaffected.
- V1: first extension returning a non-void `StreamDecision` is the interceptor. Others returning void remain observers.
- Extension state (the `buffer` variable) is per-extension-instance, and each session gets its own extension instances. No shared state concern.
- `pi.setStreamInterceptorTimeout(ms)` is deferred to post-V1. V1 uses a hardcoded 500ms default.

### Performance

**No interceptor (hot path):** one boolean check (`_hasInterceptor`) per token above current baseline. Flag set on first `emitMessageUpdate()` call by scanning handlers for any that have previously returned non-void.

**Interceptor active (warm path):** string append to buffer, one function call to interceptor, check return value. On flush: construct modified event, single `_emit()`. Buffered tokens coalesce into one delta, cheaper than per-token emit.

**Abort (cold path):** `abort()` + `waitForIdle()` + retry. Interceptor adds no meaningful overhead on top of existing retry cost.

**Memory:** buffer holds at most `maxHoldMs` worth of tokens (~500ms at ~60 tokens/sec = ~30-50 tokens, few hundred bytes). Cleared on every flush and on `message_end`.

## Files Changed

| File | Change |
|------|--------|
| `pi-mono: packages/coding-agent/src/core/extensions/types.ts` | Add `StreamDecision`, `MessageUpdateEventResult`, `EmitMessageUpdateResult`. Update `message_update` handler signature. Remove `MessageUpdateEvent` from `RunnerEmitEvent`. |
| `pi-mono: packages/coding-agent/src/core/extensions/runner.ts` | Add `_streamBuffer`, `_hasInterceptor`, `_pendingAbortReason`, `_interceptorAbortCount`. New `emitMessageUpdate()` method (~80-100 lines). Inject abort reason in `emitContext()`. Buffer lifecycle tied to `text_delta` arrival and `message_end`. |
| `pi-mono: packages/coding-agent/src/core/agent-session.ts` | Add `message_update` special case in `_handleAgentEvent` before `_emitExtensionEvent`/`_emit`. Route through `emitMessageUpdate()`, handle `EmitMessageUpdateResult` outcomes. Add interceptor-abort retry trigger on `agent_end`. |

## Not Changed

- `packages/ai/` - provider layer untouched
- `packages/agent/` - agent loop and Agent class untouched
- Existing `subscribe()` listeners - see final (possibly modified) event, with known limitation that `partial` field is not patched for modify/suppress
- Existing `message_update` observers returning void - no behavior change
- tallow - no changes for V1 (extensions use pi API directly)

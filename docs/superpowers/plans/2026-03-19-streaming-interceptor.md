# Streaming Interceptor Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan, one task at a time. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Enable extensions to intercept, buffer, modify, suppress, or abort streaming tokens before they reach the TUI.

**Architecture:** New `emitMessageUpdate()` method on `ExtensionRunner` buffers `text_delta` tokens and routes them through a single interceptor (V1). `AgentSession._handleAgentEvent` gains a `message_update` special case that routes through `emitMessageUpdate()` instead of the generic `_emitExtensionEvent`/`_emit` path. Abort triggers `agent.abort()` + retry via `agent.continue()` with reason injected into the next `context` event.

**Tech Stack:** TypeScript, pi-mono extension system (`@mariozechner/pi-coding-agent`)

---

## File Structure

| File | Responsibility |
|------|----------------|
| `pi-mono: packages/coding-agent/src/core/extensions/types.ts` | New types: `StreamDecision`, `MessageUpdateEventResult`, `EmitMessageUpdateResult`. Update `message_update` handler signature. |
| `pi-mono: packages/coding-agent/src/core/extensions/runner.ts` | Exclude `MessageUpdateEvent` from `RunnerEmitEvent`. Buffer state, `emitMessageUpdate()`, `flushAndClearBuffer()`, `consumePendingAbortReason()`, `injectContextMessage()`. Timeout safety valve. |
| `pi-mono: packages/coding-agent/src/core/agent-session.ts` | `message_update` routing in `_handleAgentEvent`, interceptor-abort retry on `agent_end`, `message_end` buffer cleanup, auto-flush callback wiring. |
| `pi-mono: packages/coding-agent/test/streaming-interceptor.test.ts` | Unit tests for buffer, flush, timeout, abort, modify, suppress, observer compatibility. |

---

## Task 1: Add Types to `types.ts`

**Files:**
- Modify: `pi-mono: packages/coding-agent/src/core/extensions/types.ts:840-844` (event results area)
- Modify: `pi-mono: packages/coding-agent/src/core/extensions/types.ts:945` (handler signature)

- [ ] **Step 1: Add `StreamDecision`, `MessageUpdateEventResult`, and `EmitMessageUpdateResult` types**

In `types.ts`, after the existing `ToolResultEventResult` interface (line 844), add the new types:

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
	| { outcome: "emit"; event: MessageUpdateEvent; flushedEvent?: MessageUpdateEvent }
	| { outcome: "emit_modified"; event: MessageUpdateEvent }
	| { outcome: "hold" }
	| { outcome: "suppressed" }
	| { outcome: "aborted" };
```

- [ ] **Step 2: Update `message_update` handler signature in `ExtensionAPI`**

Change line 945 from:

```typescript
on(event: "message_update", handler: ExtensionHandler<MessageUpdateEvent>): void;
```

to:

```typescript
on(event: "message_update", handler: ExtensionHandler<MessageUpdateEvent, MessageUpdateEventResult>): void;
```

This is backwards compatible: handlers returning `void` remain pure observers. Handlers returning a `StreamDecision` become interceptors.

- [ ] **Step 3: Exclude `MessageUpdateEvent` from `RunnerEmitEvent` in `runner.ts`**

In `runner.ts` at the `RunnerEmitEvent` type (line 102-111), add `MessageUpdateEvent` to the exclusion list. It will get its own dedicated `emitMessageUpdate()` method, matching the pattern of `ToolCallEvent`, `ToolResultEvent`, etc.

Change:

```typescript
type RunnerEmitEvent = Exclude<
	ExtensionEvent,
	| ToolCallEvent
	| ToolResultEvent
	| UserBashEvent
	| ContextEvent
	| BeforeAgentStartEvent
	| ResourcesDiscoverEvent
	| InputEvent
>;
```

to:

```typescript
type RunnerEmitEvent = Exclude<
	ExtensionEvent,
	| ToolCallEvent
	| ToolResultEvent
	| UserBashEvent
	| ContextEvent
	| BeforeAgentStartEvent
	| ResourcesDiscoverEvent
	| InputEvent
	| MessageUpdateEvent
>;
```

Also add `MessageUpdateEvent` to the import block at the top of `runner.ts` (lines 13-50).

- [ ] **Step 4: Verify the build status**

Run:
```bash
cd /Users/savusilviu/pi-mono && npx tsc --noEmit -p packages/coding-agent/tsconfig.json
```

Expected: Build errors in `agent-session.ts` (line 465 calls `emit()` with a `MessageUpdateEvent`, which is now excluded from `RunnerEmitEvent`). This is expected and will be fixed in Task 3.

- [ ] **Step 5: Commit**

```bash
cd /Users/savusilviu/pi-mono
git add packages/coding-agent/src/core/extensions/types.ts packages/coding-agent/src/core/extensions/runner.ts
git commit -m "feat(extensions): add streaming interceptor types

Add StreamDecision, MessageUpdateEventResult, EmitMessageUpdateResult.
Update message_update handler signature to accept StreamDecision returns.
Exclude MessageUpdateEvent from RunnerEmitEvent (dedicated method next)."
```

---

## Task 2: Implement `emitMessageUpdate()` in `ExtensionRunner`

**Files:**
- Modify: `pi-mono: packages/coding-agent/src/core/extensions/runner.ts:196-234` (class properties)
- Modify: `pi-mono: packages/coding-agent/src/core/extensions/runner.ts` (after `emitToolResult`, add new method)
- Modify: `pi-mono: packages/coding-agent/src/core/extensions/runner.ts:675-705` (context injection in `emitContext`)
- Create: `pi-mono: packages/coding-agent/test/streaming-interceptor.test.ts`

- [ ] **Step 1: Write the test file for buffer and flush logic**

Create `pi-mono: packages/coding-agent/test/streaming-interceptor.test.ts`.

Test structure (follow the pattern in `extensions-runner.test.ts` for setup/teardown):

```
describe("Streaming Interceptor")
  describe("no interceptor (fast path)")
    - passes through text_delta when no handlers registered
    - passes through and calls observer returning void
  describe("interceptor active")
    - holds tokens when interceptor returns void (after being marked)
    - flushes buffer on pass decision
    - modifies buffer on modify decision (check delta equals "REDACTED")
    - suppresses buffer on suppress decision
    - aborts on abort decision (check abortFn called, consumePendingAbortReason returns reason)
  describe("subtype routing")
    - passes through thinking_delta without buffering
    - force-flushes buffer on text_end
    - force-flushes buffer on done
  describe("timeout safety valve")
    - auto-flushes after maxHoldMs (use vi.useFakeTimers)
  describe("buffer lifecycle")
    - flushAndClearBuffer clears state
    - consumePendingAbortReason returns and clears reason
  describe("abort retry limit")
    - downgrades abort to suppress after 3 aborts
  describe("context injection")
    - injectContextMessage adds text that emitContext picks up
```

Helpers needed:
- `makeTextDelta(delta)` creates a `MessageUpdateEvent` with `assistantMessageEvent.type` of `"text_delta"`
- `makeTextEnd()` creates a `text_end` subtype event
- `makeThinkingDelta(delta)` creates a `thinking_delta` subtype event
- `makeDone()` creates a `done` subtype event
- `createExtension(handler)` creates an `Extension` with a single `message_update` handler
- `bindCoreDefaults(runner, overrides?)` calls `runner.bindCore()` with no-op defaults and optional `abort` override

Use `SessionManager.inMemory()` for the session manager, and follow the `ModelRegistry` setup from `extensions-runner.test.ts`.

- [ ] **Step 2: Run the tests to verify they fail**

Run:
```bash
cd /Users/savusilviu/pi-mono && npx vitest run packages/coding-agent/test/streaming-interceptor.test.ts
```

Expected: FAIL with errors that `emitMessageUpdate`, `flushAndClearBuffer`, `consumePendingAbortReason`, `injectContextMessage` do not exist on `ExtensionRunner`.

- [ ] **Step 3: Add buffer state and new properties to `ExtensionRunner`**

In `runner.ts`, add new private fields after line 218 (`private commandDiagnostics`):

```typescript
	// Streaming interceptor state
	private _streamBuffer: {
		tokens: string;
		events: MessageUpdateEvent[];
		flushTimer: ReturnType<typeof setTimeout> | null;
		maxHoldMs: number;
	} | null = null;
	private _hasInterceptor = false;
	private _pendingAbortReason: string | undefined = undefined;
	private _interceptorAbortCount = 0;
	private _pendingContextInjection: string | undefined = undefined;
	private _onAutoFlush: ((event: MessageUpdateEvent) => void) | undefined = undefined;
```

- [ ] **Step 4: Add new type imports to the import block**

In `runner.ts`, add to the import from `"./types.js"` (lines 13-50):

```typescript
	EmitMessageUpdateResult,
	MessageUpdateEvent,
	MessageUpdateEventResult,
	StreamDecision,
```

These go in alphabetical order within the existing import block.

- [ ] **Step 5: Implement `emitMessageUpdate()` method**

Add after `emitToolResult()` (after line 621). The method is approximately 80-100 lines and follows this flow:

1. **Fast path** (no interceptor): iterate all `message_update` handlers. If all return `void`, return `{ outcome: "emit", event }`. If any returns a `StreamDecision`, set `_hasInterceptor = true` and process the decision.

2. **Interceptor active, subtype routing**:
   - Non-buffered types (thinking_\*, toolcall_\*, text_start, start): call observers, return `{ outcome: "emit", event }`
   - Force-flush types (`text_end`, `done`, `error`): flush buffer with `pass` if non-empty, call observers, return `{ outcome: "emit", event }`
   - `text_delta`: append delta to `_streamBuffer.tokens`, push event to `_streamBuffer.events`, start flush timer if not running, call interceptor

3. **Process interceptor decision** via `_processInterceptorDecision(event, decision)`:
   - `pass`: flush buffer as coalesced event, return `{ outcome: "emit", event: flushedEvent }`
   - `modify`: flush buffer with rewritten delta, return `{ outcome: "emit_modified", event: modifiedEvent }`
   - `suppress`: clear buffer, return `{ outcome: "suppressed" }`
   - `abort`: check `_interceptorAbortCount >= 3` (downgrade to suppress if exceeded), increment counter, clear buffer, set `_pendingAbortReason`, call `this.abortFn()`, return `{ outcome: "aborted" }`

```typescript
	async emitMessageUpdate(event: MessageUpdateEvent): Promise<EmitMessageUpdateResult> {
		const subtype = event.assistantMessageEvent.type;

		// Fast path: no interceptor registered yet
		if (!this._hasInterceptor) {
			const ctx = this.createContext();
			for (const ext of this.extensions) {
				const handlers = ext.handlers.get("message_update");
				if (!handlers || handlers.length === 0) continue;
				for (const handler of handlers) {
					try {
						const result = await handler(event, ctx);
						if (result !== undefined && result !== null) {
							this._hasInterceptor = true;
							return this._processInterceptorDecision(event, result as StreamDecision);
						}
					} catch (err) {
						this.emitError({
							extensionPath: ext.path,
							event: "message_update",
							error: err instanceof Error ? err.message : String(err),
							stack: err instanceof Error ? err.stack : undefined,
						});
					}
				}
			}
			return { outcome: "emit", event };
		}

		// Subtype routing
		const forceFlushTypes = new Set(["text_end", "done", "error"]);
		if (subtype !== "text_delta" && !forceFlushTypes.has(subtype)) {
			await this._callObservers(event);
			return { outcome: "emit", event };
		}
		if (forceFlushTypes.has(subtype)) {
			const flushedEvent = this._forceFlush();
			await this._callObservers(event);
			return { outcome: "emit", event, flushedEvent };
		}

		// text_delta: buffer and call interceptor
		if (!this._streamBuffer) {
			this._streamBuffer = { tokens: "", events: [], flushTimer: null, maxHoldMs: 500 };
		}
		const delta = (event.assistantMessageEvent as { delta: string }).delta;
		this._streamBuffer.tokens += delta;
		this._streamBuffer.events.push(event);
		if (this._streamBuffer.flushTimer === null) {
			this._streamBuffer.flushTimer = setTimeout(
				() => this._timeoutFlush(),
				this._streamBuffer.maxHoldMs,
			);
		}

		// Call interceptor (V1: first non-void return wins)
		const ctx = this.createContext();
		let decision: StreamDecision | undefined;
		for (const ext of this.extensions) {
			const handlers = ext.handlers.get("message_update");
			if (!handlers || handlers.length === 0) continue;
			for (const handler of handlers) {
				try {
					const result = await handler(event, ctx);
					if (result !== undefined && result !== null) {
						decision = result as StreamDecision;
						break;
					}
				} catch (err) {
					this.emitError({
						extensionPath: ext.path,
						event: "message_update",
						error: err instanceof Error ? err.message : String(err),
						stack: err instanceof Error ? err.stack : undefined,
					});
				}
			}
			if (decision) break;
		}

		if (!decision) return { outcome: "hold" };
		return this._processInterceptorDecision(event, decision);
	}
```

- [ ] **Step 6: Implement private helper methods**

Add these private methods to `ExtensionRunner`:

```typescript
	private _processInterceptorDecision(
		event: MessageUpdateEvent,
		decision: StreamDecision,
	): EmitMessageUpdateResult {
		switch (decision.action) {
			case "pass": {
				const flushedEvent = this._flushBuffer(event);
				return { outcome: "emit", event: flushedEvent };
			}
			case "modify": {
				const modifiedEvent = this._flushBufferModified(event, decision.text);
				return { outcome: "emit_modified", event: modifiedEvent };
			}
			case "suppress": {
				this._clearBuffer();
				return { outcome: "suppressed" };
			}
			case "abort": {
				if (this._interceptorAbortCount >= 3) {
					console.warn(
						"[extensions] Interceptor abort limit (3) reached, downgrading to suppress",
					);
					this._clearBuffer();
					return { outcome: "suppressed" };
				}
				this._interceptorAbortCount++;
				this._clearBuffer();
				this._pendingAbortReason = decision.reason;
				this.abortFn();
				return { outcome: "aborted" };
			}
		}
	}

	private _flushBuffer(latestEvent: MessageUpdateEvent): MessageUpdateEvent {
		const tokens = this._streamBuffer?.tokens ?? "";
		this._resetBuffer();
		if (!tokens) return latestEvent;
		return {
			...latestEvent,
			assistantMessageEvent: {
				...latestEvent.assistantMessageEvent,
				delta: tokens,
			} as MessageUpdateEvent["assistantMessageEvent"],
		};
	}

	private _flushBufferModified(
		latestEvent: MessageUpdateEvent,
		text: string,
	): MessageUpdateEvent {
		this._resetBuffer();
		return {
			...latestEvent,
			assistantMessageEvent: {
				...latestEvent.assistantMessageEvent,
				delta: text,
			} as MessageUpdateEvent["assistantMessageEvent"],
		};
	}

	private _forceFlush(): MessageUpdateEvent | undefined {
		if (!this._streamBuffer?.tokens) return undefined;
		const tokens = this._streamBuffer.tokens;
		const lastEvent =
			this._streamBuffer.events[this._streamBuffer.events.length - 1];
		this._resetBuffer();
		if (!lastEvent) return undefined;
		return {
			...lastEvent,
			assistantMessageEvent: {
				...lastEvent.assistantMessageEvent,
				delta: tokens,
			} as MessageUpdateEvent["assistantMessageEvent"],
		};
	}

	private _timeoutFlush(): void {
		if (!this._streamBuffer?.tokens) return;
		console.warn(
			"[extensions] Streaming interceptor timeout - auto-flushing buffer",
		);
		const tokens = this._streamBuffer.tokens;
		const lastEvent =
			this._streamBuffer.events[this._streamBuffer.events.length - 1];
		this._resetBuffer();
		if (lastEvent && this._onAutoFlush) {
			this._onAutoFlush({
				...lastEvent,
				assistantMessageEvent: {
					...lastEvent.assistantMessageEvent,
					delta: tokens,
				} as MessageUpdateEvent["assistantMessageEvent"],
			});
		}
	}

	private _clearBuffer(): void {
		if (this._streamBuffer) {
			if (this._streamBuffer.flushTimer !== null) {
				clearTimeout(this._streamBuffer.flushTimer);
			}
			this._streamBuffer = null;
		}
	}

	private _resetBuffer(): void {
		if (this._streamBuffer) {
			if (this._streamBuffer.flushTimer !== null) {
				clearTimeout(this._streamBuffer.flushTimer);
			}
			this._streamBuffer.tokens = "";
			this._streamBuffer.events = [];
			this._streamBuffer.flushTimer = null;
		}
	}

	private async _callObservers(event: MessageUpdateEvent): Promise<void> {
		const ctx = this.createContext();
		for (const ext of this.extensions) {
			const handlers = ext.handlers.get("message_update");
			if (!handlers || handlers.length === 0) continue;
			for (const handler of handlers) {
				try {
					await handler(event, ctx);
				} catch (err) {
					this.emitError({
						extensionPath: ext.path,
						event: "message_update",
						error: err instanceof Error ? err.message : String(err),
						stack: err instanceof Error ? err.stack : undefined,
					});
				}
			}
		}
	}
```

- [ ] **Step 7: Implement public API methods**

Add these public methods to `ExtensionRunner`:

```typescript
	/** Flush remaining buffer and clear all interceptor state. Called on message_end. */
	flushAndClearBuffer(): MessageUpdateEvent | undefined {
		if (!this._streamBuffer?.tokens) {
			this._clearBuffer();
			return undefined;
		}
		const tokens = this._streamBuffer.tokens;
		const lastEvent =
			this._streamBuffer.events[this._streamBuffer.events.length - 1];
		this._clearBuffer();
		if (!lastEvent) return undefined;
		return {
			...lastEvent,
			assistantMessageEvent: {
				...lastEvent.assistantMessageEvent,
				delta: tokens,
			} as MessageUpdateEvent["assistantMessageEvent"],
		};
	}

	/** Consume and clear the pending abort reason. Returns undefined if none. */
	consumePendingAbortReason(): string | undefined {
		const reason = this._pendingAbortReason;
		this._pendingAbortReason = undefined;
		return reason;
	}

	/** Inject a context message to be appended on the next emitContext() call. */
	injectContextMessage(text: string): void {
		this._pendingContextInjection = text;
	}

	/** Reset interceptor abort counter. Called on successful message_end. */
	resetInterceptorAbortCount(): void {
		this._interceptorAbortCount = 0;
	}

	/** Set callback for timeout auto-flush. Called by AgentSession to wire _emit. */
	setAutoFlushCallback(cb: (event: MessageUpdateEvent) => void): void {
		this._onAutoFlush = cb;
	}
```

- [ ] **Step 8: Modify `emitContext()` to inject pending context message**

In `emitContext()` (line 675), add context injection logic after the handler loop (line 702), before `return currentMessages;` (line 704):

```typescript
		// Inject pending context message from interceptor abort
		if (this._pendingContextInjection) {
			const injectionText = this._pendingContextInjection;
			this._pendingContextInjection = undefined;
			currentMessages = [
				...currentMessages,
				{
					role: "user",
					content: [{ type: "text", text: `[System: ${injectionText}]` }],
				} as AgentMessage,
			];
		}
```

- [ ] **Step 9: Run tests to verify they pass**

Run:
```bash
cd /Users/savusilviu/pi-mono && npx vitest run packages/coding-agent/test/streaming-interceptor.test.ts
```

Expected: Most tests PASS. Fix any failures.

- [ ] **Step 10: Commit**

```bash
cd /Users/savusilviu/pi-mono
git add packages/coding-agent/src/core/extensions/runner.ts packages/coding-agent/test/streaming-interceptor.test.ts
git commit -m "feat(extensions): implement emitMessageUpdate with buffer and interceptor

Add streaming buffer, text_delta interception, subtype routing table,
timeout safety valve (500ms), abort/modify/suppress/pass decisions,
force-flush on text_end/done/error, context injection for abort retry."
```

---

## Task 3: Wire `AgentSession._handleAgentEvent` to `emitMessageUpdate()`

**Files:**
- Modify: `pi-mono: packages/coding-agent/src/core/agent-session.ts:317-396` (_handleAgentEvent)
- Modify: `pi-mono: packages/coding-agent/src/core/agent-session.ts:429-499` (_emitExtensionEvent)

- [ ] **Step 1: Add `message_update` special case in `_handleAgentEvent`**

In `_handleAgentEvent` (line 317), add a new block **before** lines 337-341 (`await this._emitExtensionEvent(event)` / `this._emit(event)`). Replace lines 337-341:

```typescript
		// Emit to extensions first
		await this._emitExtensionEvent(event);

		// Notify all listeners
		this._emit(event);
```

with:

```typescript
		// Special case: message_update routes through streaming interceptor
		if (event.type === "message_update" && this._extensionRunner) {
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
					// Emit force-flushed buffer tokens before the boundary event
					if (result.flushedEvent) {
						this._emit({
							...event,
							assistantMessageEvent: result.flushedEvent.assistantMessageEvent,
						});
					}
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

		// Emit to extensions first
		await this._emitExtensionEvent(event);

		// Notify all listeners
		this._emit(event);
```

- [ ] **Step 2: Add `message_end` buffer cleanup in `_handleAgentEvent`**

In `_handleAgentEvent`, inside the `event.type === "message_end"` block (line 344), add buffer flush as the first thing after the `if` condition:

```typescript
			// Flush any remaining interceptor buffer before processing message_end
			if (this._extensionRunner) {
				const flushedEvent = this._extensionRunner.flushAndClearBuffer();
				if (flushedEvent) {
					this._emit({
						type: "message_update",
						message: event.message,
						assistantMessageEvent: flushedEvent.assistantMessageEvent,
					} as AgentEvent);
				}
				// Only reset abort counter on successful (non-aborted) message_end
				const msg = event.message as AssistantMessage;
				if (msg.role === "assistant" && msg.stopReason !== "aborted") {
					this._extensionRunner.resetInterceptorAbortCount();
				}
			}
```

- [ ] **Step 3: Add interceptor-abort retry trigger on `agent_end`**

In `_handleAgentEvent`, inside the `event.type === "agent_end"` block (line 384), add interceptor abort check **after** `this._lastAssistantMessage = undefined;` (line 386) and **before** the existing `_isRetryableError` check (line 389):

```typescript
			// Check for interceptor abort - retry with injected reason
			if (this._extensionRunner) {
				const abortReason = this._extensionRunner.consumePendingAbortReason();
				if (abortReason) {
					this._extensionRunner.injectContextMessage(abortReason);
					const messages = this.agent.state.messages;
					if (messages.length > 0 && messages[messages.length - 1].role === "assistant") {
						this.agent.replaceMessages(messages.slice(0, -1));
					}
					setTimeout(() => {
						this.agent.continue().catch(() => {});
					}, 0);
					return;
				}
			}
```

- [ ] **Step 4: Remove `message_update` from `_emitExtensionEvent`**

In `_emitExtensionEvent` (line 429), remove the `message_update` branch (lines 459-465). It is now handled by the special case in `_handleAgentEvent`. Delete:

```typescript
		} else if (event.type === "message_update") {
			const extensionEvent: MessageUpdateEvent = {
				type: "message_update",
				message: event.message,
				assistantMessageEvent: event.assistantMessageEvent,
			};
			await this._extensionRunner.emit(extensionEvent);
```

- [ ] **Step 5: Add `MessageUpdateEvent` import to `agent-session.ts`**

Add `MessageUpdateEvent` to the import from extensions types (around line 46-50). Check if it is already imported; if not:

```typescript
import type { MessageUpdateEvent } from "./extensions/types.js";
```

Or add it to the existing destructured import if one exists.

- [ ] **Step 6: Wire the auto-flush callback for timeout safety valve**

In the method that sets up the extension runner (where `bindCore` is called, around the `_initExtensions` area), add the auto-flush callback so the timeout safety valve can emit flushed tokens to subscribers:

```typescript
this._extensionRunner.setAutoFlushCallback((flushedEvent) => {
	this._emit({
		type: "message_update",
		message: flushedEvent.message,
		assistantMessageEvent: flushedEvent.assistantMessageEvent,
	} as AgentEvent);
});
```

Search for where `bindCore` is called on `this._extensionRunner` in `agent-session.ts` (around line 1935-1958) and add this call immediately after.

- [ ] **Step 7: Verify the build compiles**

Run:
```bash
cd /Users/savusilviu/pi-mono && npx tsc --noEmit -p packages/coding-agent/tsconfig.json
```

Expected: PASS (no type errors).

- [ ] **Step 8: Run all interceptor tests**

Run:
```bash
cd /Users/savusilviu/pi-mono && npx vitest run packages/coding-agent/test/streaming-interceptor.test.ts
```

Expected: PASS

- [ ] **Step 9: Commit**

```bash
cd /Users/savusilviu/pi-mono
git add packages/coding-agent/src/core/agent-session.ts
git commit -m "feat(extensions): wire streaming interceptor into AgentSession

Route message_update through emitMessageUpdate() with hold/suppress/abort
short-circuiting. Flush buffer on message_end. Trigger interceptor-abort
retry on agent_end with context injection."
```

---

## Task 4: Run Full Test Suite and Fix Regressions

**Files:**
- Possibly modify any of the three target files if regressions are found

- [ ] **Step 1: Run the full coding-agent test suite**

Run:
```bash
cd /Users/savusilviu/pi-mono && npx vitest run packages/coding-agent/test/
```

Expected: All existing tests PASS. Key regression risks:
- `extensions-runner.test.ts` may have tests that call `emit()` with `MessageUpdateEvent`, which will now fail because `MessageUpdateEvent` is excluded from `RunnerEmitEvent`. Update these to call `emitMessageUpdate()` instead.
- `agent-session-*.test.ts` should be unaffected since the event flow is preserved.

- [ ] **Step 2: Fix any regressions**

If `extensions-runner.test.ts` has tests that emit `message_update` via `emit()`, update them to use `emitMessageUpdate()`.

If any `agent-session` tests mock `_emitExtensionEvent` and expect `message_update` to flow through it, update the mocks.

- [ ] **Step 3: Run full suite again to confirm green**

Run:
```bash
cd /Users/savusilviu/pi-mono && npx vitest run packages/coding-agent/test/
```

Expected: All tests PASS.

- [ ] **Step 4: Commit any fixes**

```bash
cd /Users/savusilviu/pi-mono
git add -u packages/coding-agent/
git commit -m "fix(extensions): update existing tests for streaming interceptor changes

Update tests that used emit() with message_update to use emitMessageUpdate()."
```

---

## Task 5: Type-check and Build Verification

**Files:**
- No new files

- [ ] **Step 1: Full type-check**

Run:
```bash
cd /Users/savusilviu/pi-mono && npx tsc --noEmit -p packages/coding-agent/tsconfig.json
```

Expected: PASS (zero errors).

- [ ] **Step 2: Build the package**

Run:
```bash
cd /Users/savusilviu/pi-mono && npm run build -w packages/coding-agent
```

Expected: Build succeeds.

- [ ] **Step 3: Verify tallow still builds against the updated pi-mono**

Run:
```bash
cd /Users/savusilviu/tallow && npm run build
```

Expected: Build succeeds. tallow has no code changes for V1, but imports from `@mariozechner/pi-coding-agent` must still resolve.

- [ ] **Step 4: Commit (if any build config changes were needed)**

Only if changes were required. Otherwise skip.

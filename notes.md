# Bug Reproduction Notes: "Should not already be working" in Firefox after a breakpoint/alert

## Steps to reproduce

1. Use a React application with React 16.11 (or an unpatched scheduler) in Firefox (or Chrome 68 on Windows 7).
2. Trigger a situation where the browser's native re-entrant event loop fires a pending `MessageChannel` message while a React scheduler task callback is still on the call stack. This happens when:
   - A component or lifecycle method calls `window.alert()`, `window.confirm()`, or `window.prompt()` during rendering, **or**
   - A debugger breakpoint is hit in Firefox while a React scheduler task is executing.
3. When the native dialog is dismissed (or the debugger is resumed), the second `MessageChannel` callback re-enters `performWorkUntilDeadline`, calls `flushWork` again, and tries to start another React render while `executionContext` already has `RenderContext` set.

To confirm programmatically, run the regression test added to `packages/scheduler/src/__tests__/Scheduler-test.js`:

```
node ./scripts/jest/jest-cli.js --testPathPattern Scheduler-test -t "does not process tasks re-entrantly"
```

Without the fix to `performWorkUntilDeadline` in `Scheduler.js` the test throws `Error: Message event already scheduled` (because the re-entrant call incorrectly schedules a second browser message while one is already pending) and/or sets `taskBExecutedDuringTaskA = true` (Task B runs inside Task A's callback), demonstrating the re-entrancy.

## Observed

Without the fix, the scheduler's `performWorkUntilDeadline` function has no re-entrancy guard. When Firefox (or a similar browser) fires a queued `MessageChannel` message during a native modal dialog while a task callback is already on the call stack:

- `performWorkUntilDeadline` is entered a second time while `isMessageLoopRunning = true`.
- The function falls through into `flushWork`, which calls `workLoop` again.
- `workLoop` executes the next pending React task (for a different root, or the same root after continuation), calling `performWorkOnRoot`.
- `performWorkOnRoot` checks `executionContext` and finds `RenderContext` still set from the outer render, throwing:

```
Error: Should not already be working.
```

Even when the thrown error does not surface directly, the re-entrant `workLoop` call disrupts task ordering (Task B executes inside Task A's callback), corrupts `isMessageLoopRunning` state, and can silently drop work.

## Expected

`performWorkUntilDeadline` should detect re-entrant calls and exit early without doing any work. When `isPerformingWork` is `true`, a second invocation of `performWorkUntilDeadline` indicates it was called from within a running scheduler task (e.g., via a native browser dialog's nested event loop). The fix adds this guard:

```js
if (isPerformingWork) {
  return; // skip re-entrant call
}
```

With this guard in place:

- The re-entrant `MessageChannel` callback is a no-op.
- The outer `performWorkUntilDeadline` resumes normally after the dialog/breakpoint is cleared.
- Tasks execute in the correct order without re-entrancy.
- The "Should not already be working" error no longer occurs in Firefox after an `alert()` or breakpoint.

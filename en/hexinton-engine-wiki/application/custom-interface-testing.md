# Custom Interface Testing

Status: current author-facing test checklist.

Test a custom surface at both the protocol and visual layers.

## Protocol Cases

- Valid bootstrap with empty and populated state.
- Unknown command and malformed parameters.
- Correlated success and structured failure.
- Denied capability and unavailable process session.
- Duplicate request IDs and repeated action requests.
- Stale, future, and out-of-scope feed revisions.
- Service startup failure and disable cleanup.

## UI Cases

- Loading, ready, stale, unavailable, and error states.
- Keyboard and accessibility behavior for every control.
- Long values, missing optional fields, and narrow surfaces.
- Reconnect or repeated bootstrap without duplicated subscriptions.

Run the actual authored HTML/entry module with the supplied SDK and a real DOM/browser plus
MessageChannel. Keep commands disabled after `ready()` until an `active` lifecycle message;
hold that interval in the test instead of immediately activating. Exercise the real input/click
handlers, repeated commands, pause/resume and failed-command draft preservation. Helper-only tests
do not check those handlers. Labels and errors follow [Engine Essentials](../engine/engine-essentials.md)
and describe the player's task; technical details stay in diagnostics.

Pair the UI check with a native [surface command test](../engine/testing.md#custom-surface-command-steps)
using the same argument object. A Node command case uses `node --test --test-reporter=tap` with
the existing prepared DOM dependencies; do not install a separate test runtime in every package.

Never use a successful render as evidence that native memory work succeeded. Assert command results,
feed revisions, cleanup, and error codes separately.

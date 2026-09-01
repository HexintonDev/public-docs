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

Never use a successful render as evidence that native memory work succeeded. Assert command results,
feed revisions, cleanup, and error codes separately.

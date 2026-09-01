# Custom Interface API

Status: current protocol outline.

## Lifecycle

A hosted surface receives a bootstrap payload after validation. The payload identifies the package,
surface, available commands/queries, and initial ViewModel state. The host may send state updates and
errors after bootstrap. The surface should tolerate reconnect and repeated bootstrap.

## Action Message

```json
{
  "type": "command",
  "requestId": "req-1",
  "runnableId": "set-health",
  "parameters": { "value": 120 }
}
```

The host returns a correlated completion with success or structured error. Do not infer error type
from message text. Use the documented error code.

## State Message

Feed updates contain a view model identifier, revision, and serializable value. Apply only updates
that belong to the current package/surface and are newer than the last accepted revision.

The protocol is intentionally capability-limited: the page cannot call Lua, read process memory, or
invoke arbitrary host methods.

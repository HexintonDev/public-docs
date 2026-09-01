# Application State Synchronization

Status: current desktop WebView schema-2 behavior. Remote cloud relay is future work.

The C# application is the authority for projected application state. React receives snapshots and
versioned deltas through the application-state channel; it does not reconstruct committed state from
individual command responses.

## Versions

- `schemaVersion` identifies the shape of the projected ViewModel.
- `stateVersion` monotonically identifies one committed application-state history.

A snapshot at version `V` contains all mutations through `V`. A delta at `V` advances a contiguous
mirror from `V - 1` to `V`. Deltas use absolute `upsert` or `delete`, not `toggle`, `increment`, or
`append` operations.

## Initial Synchronization

1. Subscribe before requesting a snapshot.
2. Enter `awaiting-snapshot`.
3. Send a unique request ID.
4. Buffer newer deltas while the snapshot is in flight.
5. Apply the matching snapshot.
6. Reconcile buffered deltas in order.
7. Enter `live` only after the mirror is contiguous.

A stale or duplicate delta is ignored. A future delta is buffered until the missing version arrives.
An incompatible schema enters an explicit incompatible state instead of retrying forever.

## Message Shapes

```json
{
  "type": "application-view-model-changed",
  "payload": {
    "schemaVersion": 2,
    "stateVersion": 106,
    "path": "/sessions/demo",
    "operation": "upsert",
    "value": {}
  }
}
```

A checkpoint confirms the version posted to the WebView. It is transport acknowledgement, not a
second authority. Commands use their own correlated command-result channel.

## Hosted Trainer State

Committed widget values live under the session projection. Input drafts and command progress may be
view-local. Continuously changing values are published by a package service and feed, not simulated
by client-side polling.

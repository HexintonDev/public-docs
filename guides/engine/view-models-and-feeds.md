# View Models and State Feeds

Status: current public trainer integration reference.

## Why View Models Exist

A command answers one request. A view model represents authoritative changing state. Use a
service-backed view model when a trainer surface must stay synchronized with values that can change
outside the current button click.

## Data Flow

```text
Lua service
  -> publishEvent(event, value)
  -> host state feed with package scope and revision
  -> hosted view model
  -> declared surface binding
  -> UI render
```

The service should publish a small initial value and then publish only meaningful changes. The host
adds scope and a monotonically increasing revision so the UI can reject stale updates.

## Manifest Shape

```json
{
  "frontend": {
    "hosted": {
      "schemaVersion": 1,
      "viewModels": [
        { "id": "health", "service": "watch-health", "autoStart": true }
      ],
      "groups": []
    }
  }
}
```

A surface binds to the declared view model rather than reaching into the Lua runtime:

```json
{
  "bindings": {
    "health.current": {
      "type": "viewModel",
      "viewModel": "health"
    }
  }
}
```

## Query Versus Feed

- Use a query for a one-shot search, lookup, or pagination request.
- Use a command for an explicit user operation.
- Use a state feed and view model for changing authoritative state.

Do not poll a query as a replacement for a service-backed feed unless the feature explicitly needs
snapshot polling.

## Lifecycle

A service-backed view model starts only when its package and surface policy allow it. On disable,
the service stops, timers are destroyed, and the feed is no longer updated. UI code must handle
loading, stale, unavailable, and error states.

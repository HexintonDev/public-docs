# Trainer Controls and UI Bindings

Status: current public technical guide.

Lua functions become usable trainer features through package declarations. The runtime defines the
operation; the hosted trainer view defines how that operation is presented to a user.

## Actions, Queries, and Services

- Use an `action` for a button, toggle write, patch, or explicit refresh.
- Use a `query` for a one-shot search or read requested by the user.
- Use a `service` for changing state that should be displayed continuously.

An action result means the operation reached a terminal result. It does not automatically mean that
a later state feed has updated. Keep command completion and authoritative state separate.

## Hosted View Models

Declare a service-backed view model in `frontend.hosted`:

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

The service should publish a small initial JSON payload and later publish only meaningful changes.
The host adds package scope and revision metadata. Do not use a query as a replacement for a live
service.

## Binding an Action

```json
"bindings": {
  "health.current": {
    "type": "viewModel",
    "viewModel": "health"
  },
  "health.set": {
    "type": "command",
    "action": "set-health",
    "executionPolicy": "write"
  }
}
```

The binding alias is local to the surface. The command's arguments are validated against the action
`parameterSchema` in the package manifest.

## Queries and Feeds

```json
"inventory.search": {
  "type": "query",
  "query": "search-inventory"
},
"inventory.state": {
  "type": "feed",
  "stateFeed": "resources"
}
```

Queries are appropriate for search and pagination. A feed connects a surface to a service-backed
state feed and carries a scoped value plus a monotonically increasing revision.

## Custom Surface Declaration

```json
{
  "frontend": {
    "surfaces": [
      {
        "id": "health-panel",
        "kind": "web-inline",
        "label": "Health",
        "entry": "frontend/health/index.html",
        "presentation": { "mode": "panel", "size": "large" },
        "bindings": {
          "health": { "type": "viewModel", "viewModel": "health" },
          "setHealth": { "type": "command", "action": "set-health", "executionPolicy": "write" }
        }
      }
    ]
  }
}
```

Supported placements are `web-inline` with `compact` or `panel` mode, and `web-app` with `app` mode.
The current host loads one package-relative HTML document and enforces a 1 MiB document limit.

Surface code runs in a sandbox. It receives only declared bindings and must use the provided surface
SDK. It must not call the raw WebView bridge, access credentials, or assume unrestricted filesystem,
network, or native access.

## UI Design Rules

Use a toggle for a reversible enable/disable feature, a command button for a one-time action, a
query for transient results, and a read-only view model value for authoritative state. Keep local
draft input separate from committed service state.
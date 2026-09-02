# Trainer Controls and UI Bindings

Status: current public technical guide.

Lua functions become usable trainer features through package declarations. The runtime defines the
operation; the hosted trainer view defines how that operation is presented to a user. The live value
path is explained in [View Models and State Feeds](view-models-and-feeds.md).

## Choose the Contract

| User need | Package runnable | UI binding | What comes back |
| --- | --- | --- | --- |
| Perform one operation | `action` | `command` | One correlated command result |
| Read once | `query` | `query` | One response for that request |
| Keep observing a value | `service` | `viewModel` | Repeated host state updates |

Use an `action` for a button, toggle write, patch, or explicit refresh. Use a `query` for a
one-shot search or read. Use a `service` when the value can change without the current UI action.

An action result answers “did this operation finish?”. It is not a live value and does not replace a
later Service observation. If another source can change the value, display the ViewModel updated by
the Service rather than the last value returned by the action.

## Hosted View Models

Declare a ViewModel in `frontend.hosted` and connect it to a Service:

```json
{
  "frontend": {
    "hosted": {
      "schemaVersion": 1,
      "viewModels": [{ "id": "health", "service": "watch-health", "autoStart": true }],
      "groups": []
    }
  }
}
```

The Service calls `publishEvent`; it does not update a control directly. The host matches the event
to the declared Service, stores the payload as Feed state, updates the ViewModel, and sends a state
update to the UI. Publish a small initial JSON payload and later publish only meaningful changes.
The host adds package scope and revision metadata. Do not use a query as a replacement for a live
Service.

## Binding an Action

The complete bindings object for a surface can contain both the current ViewModel value and the
Command that changes it:

```json
{
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
}
```

The binding alias is local to the surface. The command's arguments are validated against the action
`parameterSchema` in the package manifest.

## Commands, Queries, and Feeds

These bindings are shown as one complete object so the JSON can be copied and then nested under a
surface declaration:

```json
{
  "bindings": {
    "inventory.search": {
      "type": "query",
      "query": "search-inventory"
    },
    "inventory.state": {
      "type": "viewModel",
      "viewModel": "resources"
    }
  }
}
```

Queries are appropriate for search and pagination. A Command is a request such as `set-health`. A
Feed is the host route for values published by a Service. The ViewModel is the current host
projection that controls bind to. The UI receives a command result separately from a ViewModel state
update.

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
query for transient results, and a read-only ViewModel value for the current value observed by the
Service. Keep local draft input separate from the committed ViewModel value. A delta is only the
transport message that changes the ViewModel; it is not the ViewModel itself.
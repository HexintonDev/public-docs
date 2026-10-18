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

## Complete hosted toggle

Create `example.toggle/package.json`:

```json
{
  "id": "example.toggle",
  "version": "1.0.0",
  "displayName": "Example Toggle",
  "runtime": {
    "files": [{ "path": "lua/main.lua", "runtime": "lua" }],
    "runnables": [
      { "id": "enable", "kind": "enable", "runtime": "lua", "entryFile": "lua/main.lua", "entrySymbol": "enable" },
      { "id": "disable", "kind": "disable", "runtime": "lua", "entryFile": "lua/main.lua", "entrySymbol": "disable" },
      { "id": "turn-on", "kind": "action", "runtime": "lua", "entryFile": "lua/main.lua", "entrySymbol": "turnOn" },
      { "id": "turn-off", "kind": "action", "runtime": "lua", "entryFile": "lua/main.lua", "entrySymbol": "turnOff" },
      { "id": "state", "kind": "query", "runtime": "lua", "entryFile": "lua/main.lua", "entrySymbol": "readState" }
    ]
  },
  "frontend": {
    "hosted": {
      "schemaVersion": 1,
      "groups": [{
        "id": "player",
        "label": "Player",
        "widgets": [{
          "id": "example-mode",
          "kind": "toggle",
          "label": "Example Mode",
          "description": "Demonstrates a control and package state; it has no gameplay effect.",
          "write": { "enableAction": "turn-on", "disableAction": "turn-off" },
          "read": { "query": "state", "resultPointer": "/enabled" },
          "tags": [{ "id": "example", "label": "Example" }]
        }]
      }]
    }
  }
}
```

Create `example.toggle/lua/main.lua`:

```lua
local state = { enabled = false }

function enable()
    -- Prepare package resources only; displaying or reading this control must not turn it on.
end

function turnOn()
    state.enabled = true
end

function turnOff()
    state.enabled = false
end

function readState()
    return { enabled = state.enabled }
end

function disable()
    turnOff()
end
```

The write IDs must name `action` runnables, not the package's `enable`/`disable` hooks. The read
ID must name a `query`. Package preparation happens before an action or query when needed:
put a user-selected gameplay change in the action, because even the initial state query can
prepare the package. Keep cleanup of owned resources in the lifecycle `disable` hook.

A Lua query must return exactly one detached JSON-compatible value; objects and arrays are
supported. Here that value is `{ "enabled": false }`. Native transport wraps it in a `result`
field, but the hosted `resultPointer` is relative to the query value, so use `/enabled`, not
`/result/enabled`. Use an empty pointer when reading a scalar query value.

An action's Lua return values are discarded. Returning `false` or `{ ok = false }` does not
fail the command. Raise an error when a required check or cleanup fails; otherwise a toggle
may commit the requested state despite an unsuccessful operation. For a non-service-backed
toggle, command success can commit that requested boolean; it does not establish an independently
observed game value. Use a service/feed when other code can change the actual state.

## Groups and control tags

Author groups live in `frontend.hosted.groups`. Each group has a stable `id`, a readable `label`,
and `widgets`. Choose groups around the user's task, such as Player, Combat or Inventory. Group
IDs must not change when labels are translated.

Each widget can additionally declare an optional `tags` array:

```json
"tags": [{ "id": "health", "label": "Health" }, { "id": "survival", "label": "Survival" }]
```

This is a fragment inside a widget, not a top-level manifest field. The current compiler accepts
at most 32 tags per widget, with unique IDs and IDs/labels no longer than 64 characters. Repeated
tag IDs across widgets in one package must use the same label. The host prefixes tag identity
with the package ID. These control tags are separate from the backend package page's string tags.
Personal trainer organization may override layout without rewriting the author manifest.

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
      "action": "set-health"
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
          "setHealth": { "type": "command", "action": "set-health" }
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

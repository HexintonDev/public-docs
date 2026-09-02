# Custom Interface Quickstart

Status: current authoring workflow.

This quickstart builds one surface with a button that sends a Command and a value that stays current
through a Service, Feed, and ViewModel. These are different parts of the same path.

For a package that can be copied without assembling snippets from this page, use the
[Complete Custom Interface Example](custom-interface-health-example.md). The example includes the
manifest, Lua runtime file, HTML surface, expected message flow, and cleanup steps. The snippets below
explain the same package section by section.

## 1. Decide What the Control Needs

Use a `command` when the user requests one operation. Use a `query` when the user requests one
read. Use a `service` plus a `viewModel` when the value can change without the current UI action.

For a health control:

```text
set-health Command: user asks to write 100
watch-health Service: repeatedly reads the target health
health Feed: carries health_changed observations into the host
health ViewModel: current host value bound to the control
```

The Command result answers “did this write request finish?”. The ViewModel answers “what value has
the Service most recently observed and the host accepted?”.

## 2. Declare the ViewModel and Surface

Put the ViewModel under `frontend.hosted` and the custom HTML surface under `frontend.surfaces`.
This is the `frontend` section for the health example. It is a manifest fragment; use the complete
example linked above when creating a package:

```json
{
  "frontend": {
    "hosted": {
      "schemaVersion": 1,
      "viewModels": [
        {
          "id": "health",
          "service": "watch-health",
          "autoStart": true
        }
      ],
      "groups": []
    },
    "surfaces": [
      {
        "id": "health-panel",
        "kind": "web-inline",
        "label": "Health",
        "entry": "ui/health.html",
        "presentation": { "mode": "panel", "size": "large" },
        "bindings": {
          "health": { "type": "viewModel", "viewModel": "health" },
          "setHealth": {
            "type": "command",
            "action": "set-health",
            "executionPolicy": "write"
          }
        }
      }
    ]
  }
}
```

The `ui/health.html` file is the package-relative entry document for the surface. Its binding names
are capabilities supplied by the host: `health` is a current ViewModel projection and `setHealth`
is a one-shot Command. The page does not call Lua functions directly.

The runnable declarations belong alongside this `frontend` object in the same `package.json`. This is
also a manifest fragment:

```json
{
  "runtime": {
    "runnables": [
      {
        "id": "watch-health",
        "kind": "service",
        "runtime": "lua",
        "entryFile": "lua/main.lua",
        "entrySymbol": "watchHealth"
      },
      {
        "id": "set-health",
        "kind": "action",
        "runtime": "lua",
        "entryFile": "lua/main.lua",
        "entrySymbol": "setHealth"
      }
    ]
  }
}
```

For the complete manifest including lifecycle entries, parameter schemas, and the HTML entry file, copy
the [complete Custom Interface example](custom-interface-health-example.md).

## 3. Expose Engine Work

Declare a Lua `action`, `query`, or `service`. The Lua entry points and complete manifest are shown in
the [complete Custom Interface example](custom-interface-health-example.md).
Do not expose raw memory operations to the page. Bind a button to the Command and bind a changing
value to the ViewModel.

The Service calls `publishEvent("health_changed", { health = value })`. The page never calls
`publishEvent` or `readInteger` itself.

## 4. Follow the Data Path

```text
UI click -> command binding -> set-health action -> command result

watch-health service -> publishEvent -> health Feed -> health ViewModel -> state delta -> UI value
```

The two lines are related but independent. A click can succeed while the next Service observation is
still pending. A Service can publish a new value even when no button was clicked.

## 5. Handle States

Render loading, unavailable, stale, error, and ready states. A successful button response means the
command completed; it does not guarantee that a service feed has already published a new value.

## 6. Test

Validate malformed manifests, missing commands, denied capabilities, disconnected sessions, repeated
clicks, stale feed revisions, and cleanup after disable. See [Custom Interface Testing](custom-interface-testing.md).

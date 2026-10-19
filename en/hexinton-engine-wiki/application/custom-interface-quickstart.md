# Custom Interface Quickstart

Status: current authoring workflow.

This quickstart builds one surface with a button that sends a Command and a value that stays current
through a Service, Feed, and ViewModel. These are different parts of the same path.

For a package that can be copied without assembling snippets from this page, use the
[Complete Custom Interface Example](custom-interface-health-example.md). The example includes the
manifest, Lua runtime file, HTML surface, expected message flow, and cleanup steps. The snippets below
explain the same package section by section.

## Reuse the supplied SDK

The client and worker author-reference bundles provide these files under `docs/_code/`:

- `surface-sdk.js`: the existing host-tested `SurfaceSdkClient` bundled as a browser ES module.
- `surface-ui-kit.css`: the existing public `hm-*` design classes, also injected by the host.
- `surface-sdk.manifest.json`: internal build/source hashes; keep this metadata outside your package.

Copy `surface-sdk.js` into the package's UI directory using ordinary file tools, then import its
exports. This reuses the client that owns request IDs, pending replies, errors and subscriptions;
do not implement another MessagePort client. The conversion worker's `get_engine_info` reports the
module path. The in-client agent reads the same files from its installed reference bundle.

```js
import { SurfaceSdkClient, isSurfaceBootstrapMessageV1 } from "./surface-sdk.js";

let sdk;
window.addEventListener("message", event => {
  if (sdk || !isSurfaceBootstrapMessageV1(event.data) || !event.ports[0]) return;
  sdk = new SurfaceSdkClient(event.ports[0], event.data.payload);
  sdk.lifecycle.onStateChange(state => {
    document.getElementById("apply").disabled = state !== "active";
  });
  sdk.lifecycle.ready();
});
window.addEventListener("pagehide", () => sdk?.dispose(), { once: true });
```

Start the button disabled and run `sdk.commands.execute("setHealth", { health: 100 })` only after
the host reports `active`. The alias `setHealth` is the declared surface binding below; the page does
not call a Lua symbol. Handle command failure in the UI and keep the entered value for another try.
Use `sdk.queries.execute` for queries and `sdk.viewModels.subscribe` for live values.

Use `hm-surface`, `hm-input`, `hm-button`, `hm-cluster`, `hm-list-row` and the other provided classes
for the layout. The stylesheet is available for standalone previews; installed surfaces receive it
from the host. A copied custom CSP that blocks package-relative modules will also block this SDK;
use the host-owned asset policy rather than copying an older inline-only CSP example.

## Transparent documents and aligned fields

The host supplies the page background. Keep the document, main container and form transparent;
`hm-surface` already does this. Use `hm-stack` for the page rather than wrapping everything in
`hm-card`. A card is for a purposeful subsection, not an extra page background.

Use the kit's input, select, button, label and description classes for their colors, size and focus
states. Inherit the placement's control sizing and use its spacing tokens; keep custom CSS to the
layout the task needs. This layout-only example uses the kit's responsive two-column grid:

```html
<style>
  .item-fields { align-items: start; }
</style>
<body class="hm-surface">
  <main class="hm-stack">
    <div class="hm-grid hm-grid-2 item-fields">
      <div class="hm-stack-sm">
        <label class="hm-label" for="item-id">Item ID (hex)</label>
        <input id="item-id" class="hm-input" type="text" value="00061A80"
               aria-describedby="item-name">
        <span id="item-name" class="hm-description">Standard Arrow</span>
      </div>
      <div class="hm-stack-sm">
        <label class="hm-label" for="quantity">Quantity</label>
        <input id="quantity" class="hm-input" type="number" min="1" step="1" value="30">
      </div>
    </div>
    <div class="hm-cluster">
      <button id="spawn" class="hm-button hm-button-primary" type="button" disabled>Spawn Item</button>
    </div>
  </main>
</body>
```

`hm-stack-sm` keeps each field's contents at the top, so optional helper text below one input does
not stretch a neighbouring label row or move its control down. `hm-grid-2` collapses to one column
on a narrow surface. Do not replace these field stacks with stretching automatic grid rows.
The button still waits for the SDK's active lifecycle before it can execute a declared command.

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
        "entry": "frontend/health-panel/index.html",
        "presentation": { "mode": "panel", "size": "large" },
        "bindings": {
          "health": { "type": "viewModel", "viewModel": "health" },
          "setHealth": {
            "type": "command",
            "action": "set-health"
          }
        }
      }
    ]
  }
}
```

The `frontend/health-panel/index.html` file is the package-relative entry document for the surface. Its binding names
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

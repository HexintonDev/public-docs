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
- `surface-ui-kit.css`: the public `hm-*` design classes for standalone previews; the host injects them.
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

A copied custom CSP that blocks package-relative modules will also block this SDK; use the
host-owned asset policy rather than copying an older inline-only CSP example.

## Transparent documents and aligned fields

The host injects the `hm-*` UI kit into every surface page, before the page's own CSS. Do not copy or
link `surface-ui-kit.css` in a package; the `docs/_code` copy is only for standalone previews. A
copied file is loaded after the host kit and can pin an older version of its rules.

- **Root class.** Base font, text color and `box-sizing: border-box` apply to the whole document
  with or without a root class. Put `class="hm-surface"` or its identical alias `class="hm-root"` on
  `<body>`; it also removes default heading and paragraph margins. Keep the page, main container
  and forms transparent: the host owns the background.
- **Your CSS wins.** Kit base and native-control rules have zero specificity, so write only the
  layout the task needs. Do not resize or restyle native controls by hand (no
  `input[type=checkbox] { width: 20px }`, no fixed widths on inputs).
- **Native controls.** Raw checkboxes, radios, ranges, selects, buttons and text/number inputs are
  given the kit look automatically, but prefer the classes below.

| Purpose | Classes |
| --- | --- |
| Text | `hm-heading`, `hm-heading-sm`, `hm-body`, `hm-description`, `hm-label`, `hm-meta` |
| Layout | `hm-stack`, `hm-stack-sm` (vertical), `hm-cluster` (wrapping row), `hm-toolbar` (row, ends apart), `hm-grid` with `hm-grid-2`/`hm-grid-3` (one column below 620px), `hm-layout-panel` (content + 180-220px side column), `hm-control-group` (input + button row) |
| Fields | `hm-field` (label text or `hm-label`, control, optional `hm-description` in one stack; the control fills the field), `hm-field-grid` (wrapping grid of `hm-field`, columns at least 160px; change with `style="--hm-field-min: 220px"`) |
| Controls | `hm-input`, `hm-select`, `hm-textarea`; `hm-check` (`<label class="hm-check"><input type="checkbox"><span>Text</span></label>`, also for radios); `hm-toggle` (switch: same markup with `role="switch"` on the checkbox) |
| Buttons | `hm-button` plus `hm-button-primary`/`-secondary`/`-ghost`/`-danger`, sizes `hm-button-sm`/`-md`/`-lg`, `hm-button-icon` |
| Content | `hm-card` (a purposeful subsection, not a page background), `hm-divider`, `hm-chip` (`aria-pressed`), `hm-list` + `hm-list-row` (`aria-selected`), `hm-badge`, `hm-metric` + `hm-metric-value`, `hm-status` + `hm-status-success`/`-warning`/`-error`, `hm-empty`, `hm-selected` |
| Sizing | `hm-control-sm`/`-md`/`-lg` on a container or control selects 32/40/48px control geometry |

Layout rules:

- No fixed heights, `100vh` layouts or inner scroll areas (`overflow: auto` panels). The host sizes
  the frame and the document scrolls as a whole.
- Put every label + input set in `hm-field` inside `hm-field-grid`. Never put `width: 100%` inputs
  in hand-made flex rows or fixed-column grids; that is what makes inputs overlap.
- Checkbox or switch rows use `hm-check`/`hm-toggle`, not a bare input next to a block `hm-label`.
- Check the page at about 420px and 900px wide: nothing may overlap or scroll sideways.

```html
<body class="hm-root">
  <main class="hm-stack">
    <section class="hm-card hm-stack-sm">
      <div class="hm-toolbar">
        <label class="hm-toggle"><input id="damage-on" type="checkbox" role="switch"><span>Damage tuning</span></label>
        <button id="damage-reset" class="hm-button hm-button-ghost hm-button-sm" type="button">Reset values</button>
      </div>
      <div class="hm-field-grid">
        <label class="hm-field">Additional
          <input id="damage-add" class="hm-input" type="number" step="any" value="0"></label>
        <label class="hm-field">Multiplier
          <input id="damage-mul" class="hm-input" type="number" step="any" value="1">
          <span class="hm-description">1 keeps the game value.</span></label>
      </div>
      <p id="damage-error" class="hm-status hm-status-error" role="status"></p>
    </section>
    <label class="hm-check"><input id="flares" type="checkbox"><span>Infinite flares</span></label>
  </main>
</body>
```

The button or switch still waits for the SDK's active lifecycle before it can execute a declared command.

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

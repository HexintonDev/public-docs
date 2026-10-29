# Custom Interface Manifest

Status: current public manifest subset.

Hosted trainer values live under `frontend.hosted`. A custom HTML surface lives under
`frontend.surfaces`. Paths are package-relative and use forward slashes. IDs must be unique within
their declaration scope.

```json
{
  "frontend": {
    "hosted": {
      "schemaVersion": 1,
      "viewModels": [{ "id": "health", "service": "watch-health", "autoStart": true }],
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
          "setHealth": { "type": "command", "action": "set-health" }
        }
      }
    ]
  }
}
```

The host validates schema version, package-relative assets, duplicate IDs, runnable references, and
binding targets before hosting a surface. `viewModels[].service` connects a long-lived Service to a
host Feed and current ViewModel projection. A `surfaces[].bindings` entry gives the page a declared
ViewModel or one-shot command; it does not expose Lua or native memory.

`frontend.hosted.groups` describes host-owned trainer widgets. It is separate from
`frontend.surfaces`, which describes a package-owned HTML document. Do not put a surface entry under
`frontend.hosted`.

## Isolated Save Mods recording

Each binding can optionally declare `recording: { "result": ... }`. The result is a finite JSON
value supplied by the recording host instead of executing the binding's runnable or subscribing to
its live Service. This applies to `command`, `query`, and `viewModel` bindings:

```json
{
  "bindings": {
    "health.set": {
      "type": "command", "action": "set-health",
      "recording": { "result": null }
    },
    "inventory.search": {
      "type": "query", "query": "search-inventory",
      "recording": { "result": [{ "id": "potion", "name": "Potion" }] }
    },
    "health.current": {
      "type": "viewModel", "viewModel": "health",
      "recording": { "result": { "health": 100 } }
    }
  }
}
```

Omitting `recording` means no recording response was declared. It is different from
`"recording": { "result": null }`, which explicitly declares a successful null response.
`"recording": {}`, `"recording": null`, and non-object declarations are invalid. Existing packages
without this metadata keep their normal live behavior; declaring it does not change live execution.
Save Mods opens the interface in its isolated editor when every declared binding has a recording
response. Otherwise it shows a recording-support notice for that interface. Local preparation before
Start is not saved; commands issued after Start become ordered replay steps. Queries and ViewModel
reads never become steps. Loading resolves each saved command alias against the current package.

These responses are authored initial preview data, not current game state or a simulation of the
runnable. A query receives the same declared response regardless of its arguments. Stateful
interfaces should use [`environment.mode === "recording"`](custom-interface-api.md#recording-mode)
to maintain local UI state and filter packaged catalogue data. Arbitrary workflows that depend on
live game queries are not automatically simulated. Recording does not execute game commands,
queries, or live Service subscriptions.

## Entry document and folder

Each surface owns one folder, `frontend/<surface-id>/`, and its `entry` must be exactly
`frontend/<surface-id>/index.html`: the folder name equals the surface `id`, and the single HTML
file in that folder is always `index.html`, the surface's stable entrance. An entry such as
`frontend/<surface-id>/main.html`, or a folder named differently from the `id`, is rejected. Two
surfaces never share a folder.

- The host maps only the entry's folder into the surface frame. Scripts, styles, images, fonts and
  catalogue modules the page uses must be inside that folder; the page cannot read `lua/`, `data/`
  or another surface's folder.
- The document is UTF-8, at most 1 MiB, and begins with an optional `<!doctype html>`, an optional
  `<html>`, then `<head>`. The host inserts its content security policy at the start of `<head>`
  and replaces a document without one with an error page.
- References are plain relative paths inside the folder. Inline event handlers (`onclick=`),
  `<base>`, `<iframe>`, `<object>`, `<embed>`, `srcset` and CSS `@import` are not supported. Use
  `addEventListener` and one static stylesheet.
- `presentation.size` is one of `control`, `small`, `medium`, `large`, `standard` or `wide`.

Package publication applies the same folder and document rules, so a package that breaks them
cannot be published even if a local build happens to load it.

Keep package identity and native runnable declarations in the main package manifest. UI metadata
selects presentation; it does not grant native capabilities by itself.

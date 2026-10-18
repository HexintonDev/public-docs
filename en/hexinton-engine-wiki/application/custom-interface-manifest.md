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

## Entry document and folder

Each surface owns one folder, `frontend/<surface-id>/`, and `entry` names the single HTML file in it,
normally `frontend/<surface-id>/index.html`. Two surfaces never share a folder.

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
- `executionPolicy` on a command binding is `write`, `continuous` or `destructive`. Omit it on
  queries and view models.

Package publication applies the same folder and document rules, so a package that breaks them
cannot be published even if a local build happens to load it.

Keep package identity and native runnable declarations in the main package manifest. UI metadata
selects presentation; it does not grant native capabilities by itself.

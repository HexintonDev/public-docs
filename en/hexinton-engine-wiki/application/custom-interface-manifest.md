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
        "entry": "ui/health.html",
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

Keep package identity and native runnable declarations in the main package manifest. UI metadata
selects presentation; it does not grant native capabilities by itself.

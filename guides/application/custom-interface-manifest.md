# Custom Interface Manifest

Status: current public manifest subset.

Hosted UI declarations live under `frontend.hosted`. Paths are package-relative and use forward
slashes. IDs must be unique within their declaration scope.

```json
{
  "frontend": {
    "hosted": {
      "schemaVersion": 1,
      "entries": [{ "id": "trainer", "path": "ui/trainer.html" }],
      "viewModels": [{ "id": "health", "service": "watch-health", "autoStart": true }],
      "groups": [{ "id": "main", "entry": "trainer" }]
    }
  }
}
```

The host validates schema version, package-relative assets, duplicate IDs, runnable references, and
binding targets before hosting a surface. Unsupported or future fields must not be presented as
current behavior.

Keep package identity and native runnable declarations in the main package manifest. UI metadata
selects presentation; it does not grant native capabilities by itself.

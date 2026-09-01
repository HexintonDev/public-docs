# Custom Interface Quickstart

Status: current authoring workflow.

## 1. Declare a Surface

Add a hosted surface and a binding to a package manifest:

```json
{
  "frontend": {
    "hosted": {
      "schemaVersion": 1,
      "entries": [{ "id": "trainer", "path": "ui/trainer.html" }],
      "groups": [{ "id": "main", "entry": "trainer" }]
    }
  }
}
```

## 2. Expose Engine Work

Declare a Lua `action`, `query`, or `service`. Do not expose raw memory operations to the page.
Bind a control to the declared runnable and bind changing values to a ViewModel/feed.

## 3. Handle States

Render loading, unavailable, stale, error, and ready states. A successful button response means the
command completed; it does not guarantee that a service feed has already published a new value.

## 4. Test

Validate malformed manifests, missing commands, denied capabilities, disconnected sessions, repeated
clicks, stale feed revisions, and cleanup after disable. See [Custom Interface Testing](custom-interface-testing.md).

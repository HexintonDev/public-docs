# Package Format

Status: current public package-authoring reference.

## Directory and Manifest

```text
<package-id>/
  package.json
  lua/
  aa/
  js/
  ui/
```

Only manifest-declared files are runtime inputs. Use package-relative paths and `/` separators.

| Field | Required | Meaning |
| --- | --- | --- |
| `id` | yes | Stable ID and directory name |
| `version` | yes | Version used by dependency resolution |
| `displayName` | no | Human-readable name |
| `dependencies` | no | Explicit package dependencies |
| `runtime` | yes | Files, public entries, and runnables |
| `frontend` | no | Hosted UI declarations |

## Runtime Files and Runnables

```json
{
  "runtime": {
    "files": [
      { "path": "lua/main.lua", "runtime": "lua" },
      { "path": "aa/enable.aa", "runtime": "aa" }
    ],
    "runnables": [
      { "id": "enable", "kind": "enable", "runtime": "lua", "entryFile": "lua/main.lua", "entrySymbol": "enable" },
      { "id": "disable", "kind": "disable", "runtime": "lua", "entryFile": "lua/main.lua", "entrySymbol": "disable" },
      { "id": "hook", "kind": "action", "runtime": "aa", "entryFile": "aa/enable.aa" }
    ]
  }
}
```

Kinds are `enable`, `disable`, `action`, `query`, and `service`. Every package needs one functional
enable and disable lifecycle entry. Auto Assembler is action-oriented. Application JavaScript is
host-side and is not executed by the native engine runtime.

## Dependencies

```json
"dependencies": [{ "id": "trainer.shared", "compatible": "^1.0.0", "tested": "1.0.2" }]
```

Dependencies enable before dependents and disable in reverse order. Consume only declared public
values; private labels, allocations, timers, and hooks are not dependency API.

## Validation

The host checks strict JSON, matching directory and ID, declared files, unique runnable IDs,
entry-file/runtime consistency, lifecycle entries, dependency constraints, and action parameters.
Invalid packages produce structured errors. See [Errors and Results](../engine/errors-and-results.md).

# Package Format

Status: current public package-authoring reference.

## Directory and Manifest

```
<package-id>/
  package.json
  lua/
  ue4ss/
  aa/
  js/
  ui/
```

Only manifest-declared files are runtime inputs. Use package-relative paths and `/` separators.

| Field          | Required | Meaning                               |
| -------------- | -------- | ------------------------------------- |
| `id`           | yes      | Stable ID and directory name          |
| `version`      | yes      | Version used by dependency resolution |
| `displayName`  | no       | Human-readable name                   |
| `dependencies` | no       | Explicit package dependencies         |
| `runtime`      | yes      | Files, public entries, and runnables  |
| `frontend`     | no       | Hosted UI declarations                |

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

Kinds are `enable`, `disable`, `action`, `query`, and `service`. Every package needs one functional enable and disable lifecycle entry. Auto Assembler is action-oriented. Application JavaScript is host-side and is not executed by the native engine runtime.

The native runtime names are `lua`, `aa`, and `ue4ss`. UE4SS uses Lua source and requires `entrySymbol`, just like host Lua; `runtime: "ue4ss"` selects the in-game interpreter. A package can mix runtime declarations, while retaining one shared lifecycle/dependency graph. See the [UE4SS manifest and API](../../hexinton-engine-wiki/engine/ue4ss-runtime.md#manifest-and-entry-functions) and [complete two-package example](../../hexinton-engine-wiki/engine/samples/ue4ss-player-location.md).

## Dependencies

```json
{
  "dependencies": [
    { "id": "trainer.shared", "compatible": "^1.0.0", "tested": "1.0.2" }
  ]
}
```

Dependencies enable before dependents and disable in reverse order. Consume only declared public values; private labels, allocations, timers, and hooks are not dependency API.

Native execution currently resolves `compatible` and ignores `tested`; cloud publication requires
both fields and verifies that the tested version satisfies the range. Retain both fields for an
uploadable package. See [script package dependencies](../../hexinton-engine-wiki/engine/script-packages.md#dependencies).

## Validation

The host checks strict JSON, matching directory and ID, declared files, unique runnable IDs, entry-file/runtime consistency, lifecycle entries, dependency constraints, and action parameters. Invalid packages produce structured errors. See [Errors and Results](../../hexinton-engine-wiki/engine/errors-and-results.md).

# Script Packages and Lifecycle

Status: current public technical guide.

HexMod loads scripts as packages. A package is the unit of validation, dependency resolution,
execution, and cleanup.

Runtime names `lua`, `aa` and `ue4ss` select execution of individual runnables. UE4SS uses the same five kinds and native lifecycle, with separate in-game Lua state. Read [UE4SS authoring](ue4ss-runtime.md) and [cross-runtime dependency calls](dependency-runnable-api.md) when combining Unreal access with existing host Lua.

## Minimal Layout

```text
trainer.health/
  package.json
  lua/main.lua
```

The directory name must equal the manifest `id`. Runtime paths are relative to the package and use
forward slashes.

## Minimal Manifest

```json
{
  "id": "trainer.health",
  "version": "1.0.0",
  "displayName": "Health Trainer",
  "runtime": {
    "files": [
      { "path": "lua/main.lua", "runtime": "lua" }
    ],
    "public": { "lua": "lua/main.lua" },
    "runnables": [
      { "id": "enable", "kind": "enable", "runtime": "lua", "entryFile": "lua/main.lua", "entrySymbol": "enable" },
      { "id": "disable", "kind": "disable", "runtime": "lua", "entryFile": "lua/main.lua", "entrySymbol": "disable" },
      { "id": "set-health", "kind": "action", "runtime": "lua", "entryFile": "lua/main.lua", "entrySymbol": "setHealth" }
    ]
  }
}
```

## Runnable Kinds

- `enable`: allocate or install package-owned resources.
- `disable`: remove resources created by `enable`.
- `action`: perform an explicit operation.
- `query`: return a one-shot read or search result.
- `service`: observe changing state and publish events.

Every package requires one functional `enable` and one functional `disable`. The host enables
dependencies first and disables them in reverse order. If enable fails, newly enabled dependencies
are rolled back.

## Lua Entry

```lua
local healthAddress = "game.exe+0x123456"

function enable()
end

function disable()
end

function setHealth(arguments)
    writeInteger(healthAddress, arguments.value)
    return { ok = true, value = arguments.value }
end
```

The `entrySymbol` must match the Lua function exactly. Lua globals persist for the current script
context; unrelated package globals are not visible. Use explicit dependencies and their published
contract instead of relying on accidental global names.

## Dependencies

```json
{
  "dependencies": [
    { "id": "trainer.shared", "compatible": "^1.0.0", "tested": "1.0.2" }
  ]
}
```

Dependencies are enabled before the dependent package. Do not consume another package's private
labels, allocations, timers, or hooks. Duplicate visible symbols fail as ambiguous.

The current native resolver uses `compatible` (or the legacy alternative `version`) and ignores
`tested`. Cloud package publication currently requires both `compatible` and a valid `tested`
version within that range, so retain the shape above for packages intended for upload. Passing
local dependency validation alone does not establish acceptance by cloud publication.

## Validation Checklist

Before execution, the host validates strict JSON, declared files, unique runnable IDs, runtime and
entry-file consistency, lifecycle entries, dependency constraints, and action arguments.

See [Visibility and Namespaces](visibility-and-namespaces.md) for package-local modules and
dependency-visible values.

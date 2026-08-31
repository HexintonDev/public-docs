# Script Packages and Lifecycle

Status: current public technical guide.

HexMod loads scripts as packages. A package is the unit of validation, dependency resolution,
execution, and cleanup.

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
"dependencies": [
  { "id": "trainer.shared", "compatible": "^1.0.0", "tested": "1.0.2" }
]
```

Dependencies are enabled before the dependent package. Do not consume another package's private
labels, allocations, timers, or hooks. Duplicate visible symbols fail as ambiguous.

## Validation Checklist

Before execution, the host validates strict JSON, declared files, unique runnable IDs, runtime and
entry-file consistency, lifecycle entries, dependency constraints, and action arguments.
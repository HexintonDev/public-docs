# Scripting Getting Started

Status: current public author guide.

A Hexinton Engine script is installed as a package. A package contains `package.json`, declared
runtime files, and optional frontend declarations. The directory name must match the package `id`.

## Minimal Package

```text
example.health/
  package.json
  lua/main.lua
```

```json
{
  "id": "example.health",
  "version": "1.0.0",
  "displayName": "Health Example",
  "runtime": {
    "files": [{ "path": "lua/main.lua", "runtime": "lua" }],
    "runnables": [
      { "id": "enable", "kind": "enable", "runtime": "lua", "entryFile": "lua/main.lua", "entrySymbol": "enable" },
      { "id": "disable", "kind": "disable", "runtime": "lua", "entryFile": "lua/main.lua", "entrySymbol": "disable" },
      { "id": "read-health", "kind": "query", "runtime": "lua", "entryFile": "lua/main.lua", "entrySymbol": "readHealth" }
    ]
  }
}
```

```lua
function enable()
end

function disable()
end

function readHealth()
    return { current = readInteger(getAddressSafe("game.exe+0x1234")) }
end
```

## Authoring Loop

1. Declare every file and runnable in the manifest.
2. Validate process, address, scan, and architecture assumptions.
3. Make `enable` and `disable` idempotent.
4. Return serializable query/action values.
5. Test missing targets, ambiguous scans, repeated enable, action failure, and disable cleanup.

Choose Lua for attached process work, Auto Assembler for assembly actions, and application
JavaScript for capability-gated host tasks. See [runtime selection](runtimes.md) and [package
format](package-format.md).

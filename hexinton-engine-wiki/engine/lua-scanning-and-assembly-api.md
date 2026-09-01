# Lua Scanning and Assembly API

Status: current public API reference.

## Scan Functions

| Prototype | Return | Failure/empty behavior |
| --- | --- | --- |
| `AOBScan(pattern)` | String-list handle of matching addresses, or `nil` | Invalid pattern or scan failure raises; zero matches returns `nil` |
| `AOBScanUnique(pattern)` | One address, or `nil` | Zero or multiple matches return `nil` |
| `AOBScanModule(moduleName, pattern)` | String-list handle, or `nil` | Invalid module/pattern or scan failure raises; zero matches returns `nil` |
| `AOBScanModuleUnique(moduleName, pattern)` | One address, or `nil` | Zero or multiple matches return `nil` |

Use `??` for a wildcard byte. Treat `nil` or ambiguous results as a compatibility failure.

```lua
local hits = AOBScanModule("game.exe", "48 8B ?? ?? 89")
if not hits or #hits ~= 1 then
    error("expected one compatible result")
end
local injection = hits[1]
```

## Assembly and Target Functions

| Prototype | Return | Failure |
| --- | --- | --- |
| `autoAssemble(text, targetself?, disableInfo?)` | Success flag and optional error text | Assembly/parser/process failure returns failure details or raises at the Lua boundary |
| `fullAccess(address, size)` | Operation result | Invalid range/protection change raises a Lua error |
| `targetIs64Bit()` | Boolean | Reflects the attached target architecture |

`autoAssemble` executes Auto Assembler text in the current script/session context. It does not turn
native `EngineSession` patch methods into Lua functions.

```lua
local ok, errorMessage = autoAssemble([[
alloc(exampleCave, 0x1000)
registersymbol(exampleCave)
]])
if not ok then
    error(errorMessage or "assembly failed")
end
```

## Ownership

Pair `alloc` with `dealloc`, and `registersymbol` with `unregistersymbol`. Restore hook-site bytes
and release resources from the package disable path. `createThread` is an Auto Assembler directive
whose thread/resource lifecycle must be handled by the assembly author; this page does not claim a
Lua `createThread` global because none is registered.

A scan result is an address candidate, not proof that bytes are safe to overwrite. Validate expected
bytes, instruction length, architecture, and return path before patching.

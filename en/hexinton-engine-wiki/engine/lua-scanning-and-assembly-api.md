# Lua Scanning and Assembly API

Status: current public API reference.

Scan functions run against the attached default process and execute on the Lua runtime worker. AOB
patterns use space-separated byte text; `??` matches one wildcard byte. A scan result is only an
address candidate. Validate the target module, expected bytes, instruction boundaries, architecture,
and return path before writing a patch.

## `AOBScan`

```lua
AOBScan(pattern) -> stringListHandle | nil
```

Scans the entire attached process for `pattern`. No matches return `nil`. Invalid pattern text,
unavailable process state, or a scan failure raises a Lua error. A successful result is opaque
string-list userdata, not a normal Lua array; each stored address is formatted as an uppercase
hexadecimal string without a `0x` prefix.

The list is 0-based when indexed: `hits[0]` is the first address. `#hits` returns the number of
matches. `hits.Count` is an equivalent count property, and `hits:destroy()` releases the list early.
After destruction, numeric indexing returns `nil` and `#hits` returns `0`; the userdata is also
released automatically by Lua garbage collection.

```lua
local hits = AOBScan("48 8B ?? ?? 89")
if not hits then
    error("pattern was not found")
end
for index = 0, #hits - 1 do
    print(index, hits[index])
end
hits:destroy()
```

## `AOBScanUnique`

```lua
AOBScanUnique(pattern) -> address | nil
```

Scans the entire attached process and returns one numeric address only when exactly one match is
found. Zero matches and multiple matches both return `nil`; neither case distinguishes “not found”
from “ambiguous”. Invalid pattern text, unavailable process state, or a scan failure raises a Lua
error. Treat `nil` as a compatibility failure and do not patch a fallback address.

```lua
local injection = AOBScanUnique("48 8B ?? ?? 89")
if not injection then
    error("pattern is missing or ambiguous")
end
```

## `AOBScanModule`

```lua
AOBScanModule(moduleName, pattern) -> stringListHandle | nil
```

Scans only the loaded module named by `moduleName`. The module name and pattern are required string
arguments. No matches return `nil`; invalid module/pattern text, unavailable process state, or a
scan failure raises a Lua error. Its result uses the same string-list userdata, 0-based indexing,
`#hits`, `Count`, `destroy`, formatting, and garbage-collection behavior as `AOBScan`.

```lua
local hits = AOBScanModule("game.exe", "48 8B ?? ?? 89")
if not hits or #hits ~= 1 then
    error("expected one match in game.exe")
end
local injectionText = hits[0]
hits:destroy()
```

## `AOBScanModuleUnique`

```lua
AOBScanModuleUnique(moduleName, pattern) -> address | nil
```

Scans the named loaded module and returns a numeric address only for exactly one match. Zero or
multiple matches return `nil`. Invalid module/pattern text, unavailable process state, or scan
failure raises a Lua error. This is the module-scoped equivalent of `AOBScanUnique`.

```lua
local injection = AOBScanModuleUnique("game.exe", "48 8B ?? ?? 89")
if not injection then
    error("module pattern is missing or ambiguous")
end
```

## String-List Handle

The handle returned by `AOBScan` and `AOBScanModule` exposes only the following native surface:

| Member | Result |
| --- | --- |
| `hits[index]` | Formatted address string for a 0-based valid index, otherwise `nil` |
| `#hits` | Current number of entries, or `0` after destruction |
| `hits.Count` | Current number of entries, or `0` after destruction |
| `hits:destroy()` | No return value; clears the owned result list |

The address strings can be consumed by address-resolution APIs where a hexadecimal address
expression is accepted, or converted by the caller as appropriate. The handle does not expose a
Lua array iterator, a `pairs` contract, process writes, or an automatic patch/rollback operation.

## `autoAssemble`

```lua
autoAssemble(text) -> true
autoAssemble(text, targetself) -> true
autoAssemble(text, targetself, disableInfo) -> true
autoAssemble(text, ...) -> false, errorMessage
```

Executes Auto Assembler text synchronously in the current script/session context. On success it
returns exactly `true`. On assembly, parser, address, allocation, protection, or process failure it
returns `false` followed by an error string rather than throwing that failure directly. The optional
`targetself=true` mode is explicitly unsupported and returns `false, errorMessage`; a non-`nil`
`disableInfo` argument is also unsupported. Omit both optional arguments and put cleanup in the
package disable path.

```lua
local ok, errorMessage = autoAssemble([[
alloc(exampleCave, 0x1000)
registersymbol(exampleCave)
]])
if not ok then
    error(errorMessage or "assembly failed")
end
```

`autoAssemble` does not expose native `EngineSession` patch methods as Lua functions and does not
create an automatic disable/rollback record.

## `fullAccess`

```lua
fullAccess(address, size) -> true
```

Requests writable/executable access for a positive byte range beginning at `address` in the default
process. `size` must be greater than zero. It returns `true` after the underlying access operation;
an invalid address, non-positive size, unavailable process, or protection failure raises a Lua
error. It does not write bytes, remember the old protection, or restore protection automatically.

```lua
local hook = getAddressSafe("game.exe+0x1234")
if not hook then error("unsupported build") end
fullAccess(hook, 6)
```

## `targetIs64Bit`

```lua
targetIs64Bit() -> boolean
```

Returns whether the attached default process uses 64-bit pointers. Use the result to choose
architecture-specific registers, pointer widths, and instruction forms. It does not inspect the
assembly text and does not change the target architecture.

```lua
local pointerDirective = targetIs64Bit() and "dq" or "dd"
```

## Ownership and Cleanup

Pair `alloc` with `dealloc`, and `registersymbol` with `unregistersymbol`. Restore hook-site bytes
and release allocations from the package disable path. Destroy scan list handles once their address
candidates have been copied or used. `createThread` is an Auto Assembler directive whose thread
and resource lifecycle must be handled by the assembly author; there is no Lua `createThread`
global. A successful scan or assembly action never proves that a patch is compatible or reversible.

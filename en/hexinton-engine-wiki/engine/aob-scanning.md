# AOB Scanning

Status: current public technical guide.

Array-of-bytes (AOB) scanning finds a runtime address from a byte pattern instead of hard-coding an
address that may change between builds.

## Basic Scan

```lua
local hits = AOBScan("48 8B ?? ?? 89")
if not hits or #hits == 0 then
    error("pattern not found")
end

local address = hits[0]
```

`??` is a wildcard byte. Treat an empty result as a compatibility failure and do not write to an
assumed address.

## Unique and Module-Scoped Scans

```lua
local address = AOBScanUnique("48 8B ?? ?? 89")
local moduleAddress = AOBScanModuleUnique("game.exe", "48 8B ?? ?? 89")
local hits = AOBScanModule("game.exe", "48 8B ?? ?? 89")
```

Use a module-scoped scan when the pattern belongs to a known module. Use the unique variant only when
the pattern is expected to have exactly one match.

## Safe Scan Pattern

```lua
local hits = AOBScanModule("game.exe", "F3 0F 11 ?? ?? 48 8B")
if not hits or #hits ~= 1 then
    error("expected exactly one compatible instruction")
end

local hookAddress = hits[0]
```

AOB results are process addresses. Check the attached process, module name, match count, and expected
instruction bytes before installing a patch.
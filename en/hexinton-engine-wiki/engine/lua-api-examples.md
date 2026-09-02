# Lua API Examples

Status: current public example reference.

Each example is deliberately small. Replace target-specific process names, addresses, and patterns
only after verifying them against a controlled target.

## Process Discovery and Handles

```lua
local pid = findProcess("game.exe")
if not pid then
    error("game process not found")
end

local target = openProcess(pid)
local address = target:getAddress("game.exe+0x1234")
local before = target:readInteger(address)
target:writeInteger(address, before + 1)
return { pid = pid, previous = before }
```

Related functions: `findProcess`, `findProcesses`, `getProcessIDFromProcessName`, `findWindow`,
`getWindowProcessID`, `findWindowProcessID`, `getDefaultPid`, and `getOpenedProcessID`.

## Typed Memory and Strings

```lua
local address = getAddress("game.exe+0x1234")
local bytes = readBytes(address, 4, true)
local number = readInteger(address)
local ratio = readFloat(address + 4)
local text = readString(address + 8, 32, false)

writeInteger(address, number + 1)
writeFloat(address + 4, ratio * 2)
return { bytes = bytes, text = text }
```

Use the operation matching the target width, signedness, and representation. Check attachment and
address validity before every write.

## Safe Address Resolution

```lua
local address = getAddressSafe("game.exe+0x1234")
if not address then
    return { ok = false, reason = "address unavailable" }
end

local pointer = readPointer(address)
if not pointer then
    return { ok = false, reason = "pointer unavailable" }
end
```

## AOB Scanning

```lua
local hits = AOBScanModule("game.exe", "48 8B ?? ?? 89")
if not hits or #hits ~= 1 then
    error("expected exactly one compatible match")
end

local address = hits[0]
return { address = address }
```

Use `AOBScan`, `AOBScanUnique`, `AOBScanModule`, and `AOBScanModuleUnique` according to the
required match scope and count.

## Symbols

```lua
local address = getAddress("game.exe+0x1234")
registerSymbol("health_value", address)
local value = readInteger("health_value")
unregisterSymbol("health_value")
return value
```

Symbols created by a package are package-owned and must be removed during cleanup.

## Auto Assembler from Lua

```lua
local ok, errorMessage = autoAssemble([[
alloc(exampleCave, 0x1000)
registersymbol(exampleCave)
]])
if not ok then
    error(errorMessage or "assembly failed")
end
```

Pair every allocation and symbol with deterministic cleanup. See the Auto Assembler examples page
for assembly-only runnable declarations and disable actions.

## Timers and Services

```lua
local timer

function enable()
    timer = createTimer(false)
    timer.Interval = 250
    timer.OnTimer = function()
        publishEvent("health_changed", { current = readInteger(getAddress("game.exe+0x1234")) })
    end
    timer.Enabled = true
end

function disable()
    if timer then
        timer.destroy()
        timer = nil
    end
end
```

Publish an initial value before starting periodic observation and stop the timer in `disable`.

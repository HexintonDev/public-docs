# Lua Scripting

Status: current public technical guide.

Lua is the game-facing scripting runtime. It runs inside Hexinton Engine and provides
process discovery, address resolution, memory access, scanning, symbols, timers, and assembly.

## Process Selection

The package command supplies a default process session. Bare functions use that session:

```lua
local pid = getOpenedProcessID()
local address = getAddress("game.exe+0x1234")
local value = readInteger(address)
```

Use an explicit handle when selecting another process:

```lua
local pid = findProcess("game.exe")
if not pid then
    error("game process not found")
end

local game = openProcess(pid)
local value = game:readInteger("game.exe+0x1234")
game:writeInteger("game.exe+0x1234", value + 1)
```

`findProcess` returns the first matching process ID or `nil`; `findProcesses` returns all matching
IDs. Matching is case-insensitive on Windows and accepts a name such as `game` or `game.exe`.

## Memory Functions

```lua
local address = getAddress("game.exe+0x1234")
local integer = readInteger(address)
local float = readFloat(address + 4)
local pointer = readPointer(address + 8)

writeInteger(address, 999)
writeFloat(address + 4, 1.5)
```

Available typed operations include `readBytes`, `writeBytes`, `readSmallInteger`, `writeSmallInteger`,
`readWord`, `writeWord`, `readInteger`, `writeInteger`, `readQword`, `writeQword`, `readPointer`,
`readFloat`, `writeFloat`, `readDouble`, `writeDouble`, `readString`, and `writeString`.

## Script-Local State

```lua
counter = (counter or 0) + 1
return counter
```

Assigned globals persist for the current script ID. `local` values follow normal Lua scope. A
package-local module can be loaded with `require("helpers.inventory")`; module IDs use dot-separated
segments and cannot contain absolute paths, `..`, or a `.lua` suffix.

See [Visibility and Namespaces](visibility-and-namespaces.md) for complete global, module,
dependency, symbol, and label scope rules.

## Timers and Services

Use a service to publish changing state instead of repeatedly polling from a UI query. A service
should publish an initial value and later values only when the state changes. Timers are package
resources and must be stopped by `disable`.

```lua
local timer

function enable()
    timer = createTimer(false)
end

function disable()
    if timer then
        timer.destroy()
        timer = nil
    end
end
```

## Symbols

```lua
registerSymbol("player_health", getAddress("game.exe+0x1234"))
local value = readInteger("player_health")
unregisterSymbol("player_health")
```

Symbols published by a package are visible to that package and declared dependents. Unrelated
packages cannot resolve them.
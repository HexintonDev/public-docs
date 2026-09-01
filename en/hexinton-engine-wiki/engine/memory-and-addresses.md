# Memory and Address Resolution

Status: current public technical guide.

## Address Expressions

The Lua runtime accepts integer addresses and Cheat Engine-style expressions:

```lua
local absolute = getAddress(0x140012340)
local moduleAddress = getAddress("game.exe+0x1234")
local offsetAddress = getAddress("game.exe+0x1234-0x20")
```

`getAddressSafe(expression)` returns a safe failure value when an expression cannot be resolved;
use it when a missing symbol is expected. Use `getAddress` when failure must stop the action.

## Pointer Chains

Pointer expressions dereference each bracketed level:

```lua
local player = getAddress("[[game.exe+0x1234]+0x18]+0x30")
local health = readInteger(player + 0x20)
```

Pointer resolution fails if an intermediate pointer cannot be read or an expression is invalid.
Validate the target process and module before attempting a write.

## Reads and Writes

```lua
local address = getAddress("player_health")
local oldValue = readInteger(address)
writeInteger(address, 100)
return { previous = oldValue, current = readInteger(address) }
```

Use the narrowest typed operation that matches the target value. Confirm signedness and width;
`readInteger` is not interchangeable with `readQword`, `readFloat`, or `readDouble`.

## Allocations and Labels

Auto Assembler allocations and script labels belong to the current script context. A local name
such as `newmem` is not automatically visible to unrelated packages. Publish a name explicitly with
`registerSymbol` only when it is part of the package's public contract.

When disabling a package, deallocate package-owned memory and unregister package-owned symbols.
# Lua Memory API

Status: current public API reference.

Memory calls run in the Lua runtime worker and operate on the command's default process unless called
as a method on an explicit process handle. Address arguments accept integers or supported address
expressions.

## Typed Operations

| Prototype | Width/representation | Failure |
| --- | --- | --- |
| `readBytes(address, count, returnAsTable?)` | Bytes; table when requested | Invalid address, count, or read raises a Lua error |
| `writeBytes(address, bytes...)` | Raw bytes | Byte values must be `0..255`; invalid write raises a Lua error |
| `readSmallInteger(address, signed?)` | Small integer | Invalid read raises a Lua error |
| `writeSmallInteger(address, value)` | Small integer | Invalid value or write raises a Lua error |
| `readWord(address, signed?)` | Alias of `readSmallInteger` | Same behavior |
| `writeWord(address, value)` | Alias of `writeSmallInteger` | Same behavior |
| `readInteger(address, signed?)` | 32-bit integer | Invalid read raises a Lua error |
| `writeInteger(address, value)` | 32-bit integer | Invalid value or write raises a Lua error |
| `readQword(address)` | 64-bit integer | Invalid read raises a Lua error |
| `writeQword(address, value)` | 64-bit integer | Invalid value or write raises a Lua error |
| `readFloat(address)` | 32-bit floating point | Invalid read raises a Lua error |
| `writeFloat(address, value)` | 32-bit floating point | Invalid value or write raises a Lua error |
| `readDouble(address)` | 64-bit floating point | Invalid read raises a Lua error |
| `writeDouble(address, value)` | 64-bit floating point | Invalid value or write raises a Lua error |
| `readString(address, maxLength, wideChar?)` | Narrow or wide string | Invalid address/length raises a Lua error |
| `writeString(address, text, terminate?, wideChar?)` | Narrow or wide string | Invalid text/write raises a Lua error |

Handle methods expose the same operations, for example `target:readInteger(address)`.

## Byte Arguments

`writeBytes` accepts either a table or positional byte values:

```lua
local address = getAddress("game.exe+0x1234")
local before = readBytes(address, 4, true)
writeBytes(address, { 0x90, 0x90, 0x90, 0x90 })
return before
```

Every byte must be in the inclusive range `0..255`. A single string argument to an explicit process
handle is interpreted as raw byte text by the handle implementation.

## Safety Rules

Validate process attachment and address resolution before writes. Match operation width, signedness,
encoding, and target architecture to the actual value. A read/write failure raises a Lua error and
must not be silently treated as a successful command.

Memory operations are session-scoped. They do not publish state to a UI; use a query result for a
one-shot value or a service/feed for changing state.

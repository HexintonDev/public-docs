# Lua Memory API

Status: current public API reference.

Memory functions execute on the Lua runtime worker. A global function uses the command's default
process session. The global functions follow Cheat Engine's Lua contract, so a converted CE script
runs unchanged; [Cheat Engine Lua compatibility](lua-ce-compatibility.md) lists every CE function
and the few deliberate differences.

## Addresses

Every address argument may be:

- an integer, such as `0x14001234` (a number with a fraction is truncated, as in CE);
- a module expression, such as `game.exe+1234` or `"My Game.exe"+0x10`;
- a registered symbol, such as `pPlayer` or `pPlayer+18`;
- a pointer chain, such as `[[pStaticBase]+F8]+10` or `[[game.exe+2000]+18]+30`.

As in CE, a bare number inside an expression is hexadecimal: `+F8` and `+0xF8` are the same offset.
Expressions resolve in the calling package's script context, so a package sees its own symbols and
those of its declared dependencies.

## Failures

The global memory functions never raise for a memory failure. When the address cannot be resolved
(unknown name, an unreadable or null pointer level in a chain, unmapped or protected memory):

| Function kind | Result |
| --- | --- |
| `read*` | No value (`nil`) |
| `write*` except `writeBytes` | `false` |
| `writeBytes` | `0` (number of bytes written) |

A successful write returns `true` (`writeBytes`: the byte count). Check the result where the
outcome matters:

```lua
local health = readInteger("[[pPlayer]+18]+10")
if health == nil then
	error("Load a save first.", 0)
end
if not writeInteger("[[pPlayer]+18]+10", 999) then
	error("Health couldn't be changed.", 0)
end
```

Programming errors, such as a missing string argument to `writeString`, still raise a Lua error.
Every engine error, including one raised inside native code, can be caught with `pcall`.

The explicit-handle forms (`target:readInteger(address)` and the other
[process handle methods](lua-process-and-address-api.md#process-handle-properties-and-methods))
are engine functions, not CE functions: they raise a Lua error on failure.

## Integer values

Integer writes convert the value like CE's `lua_tointeger`: an integer is used as-is, a number with
a fraction is truncated toward zero (`7.9` writes `7`, `-7.9` writes `-7`) and a numeric string is
converted (`"12"` writes `12`). The value is then reduced to the field width, so `writeByte(a, 300)`
writes `44`. A value that is not a number at all (`nil`, a table, `"abc"`) is not written and the
function returns `false`; CE would write `0` instead.

## `readBytes`

```lua
readBytes(address, count = 1, returnAsTable = false) -> byte1, byte2, ...
readBytes(address, count, true) -> { byte1, byte2, ... }
```

Reads up to `count` raw bytes. With the default `returnAsTable`, the bytes are returned as multiple
values; pass `true` for a 1-based Lua array. When only the start of the range is readable, the
readable bytes are returned. When no byte can be read, or `count` is not positive, it returns no
value.

```lua
local bytes = readBytes("game.exe+1234", 4, true)
if bytes then
	for index, byte in ipairs(bytes) do
		print(index, string.format("%02X", byte))
	end
end
```

## `writeBytes`

```lua
writeBytes(address, { byte, ... }) -> count
writeBytes(address, byte1, byte2, ...) -> count
```

Writes raw bytes and returns the number of bytes written, `0` on failure. Each value keeps its low
byte (`0x1FF` and `-1` both write `0xFF`); table entries that are not numbers are skipped. Note that
`0` is a true value in Lua: test `writeBytes(...) > 0`.

The write uses temporary page-protection handling and restores the previous protection afterwards.
It does not save the previous bytes or provide rollback.

```lua
local address = getAddress("game.exe+1234")
writeBytes(address, { 0x90, 0x90, 0x90, 0x90 })
```

## `readByte` and `readShortInteger`

```lua
readByte(address, signed = false) -> integer | nil
readShortInteger(address, signed = false) -> integer | nil
```

Reads one byte. `readShortInteger` is CE's name for the same function.

## `writeByte` and `writeShortInteger`

```lua
writeByte(address, value) -> boolean
writeShortInteger(address, value) -> boolean
```

Writes one byte.

## `readSmallInteger` and `readWord`

```lua
readSmallInteger(address, signed = false) -> integer | nil
readWord(address, signed = false) -> integer | nil
```

Reads two bytes. `signed = true` returns a value in `-32768..32767`.

```lua
local stamina = readSmallInteger("game.exe+1250", true)
```

## `writeSmallInteger` and `writeWord`

```lua
writeSmallInteger(address, value) -> boolean
writeWord(address, value) -> boolean
```

Writes two bytes.

## `readInteger`

```lua
readInteger(address, signed = false) -> integer | nil
```

Reads four bytes. `signed = true` returns a value in `-2147483648..2147483647`.

```lua
local health = readInteger("game.exe+1234", true)
```

## `writeInteger`

```lua
writeInteger(address, value) -> boolean
```

Writes four bytes.

```lua
writeInteger("game.exe+1234", 999)
```

## `readQword`

```lua
readQword(address) -> integer | nil
```

Reads eight bytes as a 64-bit integer. Use `readPointer` for pointers so the width follows the
target's architecture.

## `writeQword`

```lua
writeQword(address, value) -> boolean
```

Writes eight bytes.

## `readPointer`

```lua
readPointer(address) -> pointer | nil
```

Reads one pointer: four bytes on a 32-bit target, eight bytes on a 64-bit target. A null pointer is
returned as `0`; an unreadable address returns `nil`.

```lua
local object = readPointer("game.exe+3000")
if not object or object == 0 then
	error("Load a save first.", 0)
end
```

## `readFloat` and `readDouble`

```lua
readFloat(address) -> number | nil
readDouble(address) -> number | nil
```

Read a 32-bit or 64-bit IEEE floating-point value.

## `writeFloat` and `writeDouble`

```lua
writeFloat(address, value) -> boolean
writeDouble(address, value) -> boolean
```

Write a 32-bit or 64-bit floating-point value. A numeric string is converted; a value that is not a
number is not written and the function returns `false`.

```lua
writeFloat("game.exe+1400", 2.5)
```

## `readString`

```lua
readString(address, maxLength = 50, wideChar = false) -> string | nil
```

Reads at most `maxLength` characters and stops at the first null terminator. With `wideChar=true`,
it reads UTF-16 code units and returns UTF-8 text. When only part of the range is readable, the
readable prefix is returned; when nothing is readable it returns `nil`.

```lua
local name = readString("game.exe+1800", 32)
local displayName = readString("game.exe+1900", 32, true)
```

## `writeString`

```lua
writeString(address, text, terminate = true, wideChar = false) -> boolean
```

Writes `text`. Narrow mode writes the string bytes; wide mode converts UTF-8 text to UTF-16.
`terminate` appends a null byte (narrow) or a null UTF-16 code unit (wide).

This is the one CE-named memory function whose arguments differ from CE: in CE the third argument
is `wideChar` and no terminator is written. The engine keeps its own order because packages rely on
`writeString(address, text, true)` writing a terminated narrow string. A CE script that writes a
wide string must pass `writeString(address, text, false, true)`.

```lua
writeString("game.exe+1800", "Ready")
writeString("game.exe+1900", "Ready", true, true)
```

## Scope and cleanup

Memory functions do not save original bytes, create a rollback record, or publish UI state.
Calling `fullAccess` first is a separate persistent protection change. A package that writes memory
must match width, signedness, encoding, and architecture, and restore or release its changes from
`disable`. Explicit process handles are opaque session-bound values; after runtime/session teardown
their methods fail rather than silently selecting another process.

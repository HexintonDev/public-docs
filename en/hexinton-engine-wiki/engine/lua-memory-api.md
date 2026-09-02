# Lua Memory API

Status: current public API reference.

Memory functions execute on the Lua runtime worker. A global function uses the command's default
process session. The same operation can be called on an explicit process handle, for example
`target:readInteger(address)`, to make the selected process explicit. Address arguments may be
numeric addresses or supported address expressions.

All failed address resolutions, invalid reads, invalid writes, and unavailable process sessions
raise a Lua error unless an entry below explicitly says that it returns `nil`. A successful write
returns `true`.

## `readBytes`

```lua
readBytes(address, count, returnAsTable?) -> byte1, byte2, ...
readBytes(address, count, true) -> { byte1, byte2, ... }
```

Reads `count` raw bytes from the default process. `count` must be non-negative. With the default
`returnAsTable` value, the function returns one Lua integer per byte as multiple return values, not
one table. Pass `true` to receive a 1-based Lua array instead. A zero-length read returns no values,
or an empty table when `returnAsTable` is true.

An invalid address, negative count, unavailable process, or failed read raises a Lua error. The
explicit-handle form is `target:readBytes(address, count, returnAsTable?)` and uses `target`'s pid.

```lua
local bytes = readBytes("game.exe+0x1234", 4, true)
for index, byte in ipairs(bytes) do
	print(index, string.format("%02X", byte))
end
```

## `writeBytes`

```lua
writeBytes(address, { byte, ... }) -> true
writeBytes(address, byte1, byte2, ...) -> true
```

Writes raw bytes to the default process. The byte source can be a 1-based Lua array or one or more
integer arguments. Every byte must be in the inclusive range `0..255`; at least one byte is
required. The explicit-handle form is `target:writeBytes(address, bytes)` or
`target:writeBytes(address, byte1, byte2, ...)`. For an explicit handle only, one string argument
in the byte position is treated as its raw byte contents.

An invalid address, missing byte, non-integer byte, out-of-range byte, unavailable process, or failed
write raises a Lua error. The function does not save the previous bytes or provide rollback.

```lua
local address = getAddress("game.exe+0x1234")
writeBytes(address, { 0x90, 0x90, 0x90, 0x90 })
```

## `readSmallInteger`

```lua
readSmallInteger(address, signed?) -> integer
```

Reads two bytes from the default process. The optional boolean `signed` selects signed or unsigned
interpretation. The explicit-handle form is `target:readSmallInteger(address)`; its implementation
uses the selected process but does not expose the global signed flag.

An invalid address, unavailable process, or failed read raises a Lua error. Use this operation only
when the target field is two bytes wide.

```lua
local stamina = readSmallInteger("game.exe+0x1250", true)
```

## `readWord`

```lua
readWord(address, signed?) -> integer
```

`readWord` is a registered alias of `readSmallInteger`. It reads two bytes and follows the same
signed flag, default-process scope, explicit-handle limitation, and Lua-error behavior.

```lua
local wordValue = readWord("game.exe+0x1250", false)
```

## `writeSmallInteger`

```lua
writeSmallInteger(address, value) -> true
```

Writes `value` as a two-byte integer to the default process. The explicit-handle form is
`target:writeSmallInteger(address, value)`. The selected process must be attached and writable.
The implementation converts the Lua integer to the native two-byte representation; it does not
expose a separate Lua range check for this operation.

An invalid address, non-integer value, unavailable process, or failed write raises a Lua error. The
operation does not preserve the original value.

```lua
writeSmallInteger("game.exe+0x1250", 250)
```

## `writeWord`

```lua
writeWord(address, value) -> true
```

`writeWord` is a registered alias of `writeSmallInteger`. It writes two bytes and has the same
argument, scope, failure, and cleanup behavior.

```lua
writeWord("game.exe+0x1250", 250)
```

## `readInteger`

```lua
readInteger(address, signed?) -> integer
```

Reads four bytes from the default process. The optional boolean `signed` selects signed or unsigned
interpretation. The explicit-handle form is `target:readInteger(address)`.

An invalid address, unavailable process, or failed read raises a Lua error. The field must be four
bytes wide for the result to represent the target value correctly.

```lua
local health = readInteger("game.exe+0x1234", true)
```

## `writeInteger`

```lua
writeInteger(address, value) -> true
```

Writes `value` as a four-byte integer to the default process. The explicit-handle form is
`target:writeInteger(address, value)`. The implementation converts the Lua integer to the native
four-byte representation; it does not expose a separate Lua range check for this operation.

An invalid address, non-integer value, unavailable process, or failed write raises a Lua error. It
does not calculate a pointer chain, change memory protection, or restore an earlier value.

```lua
writeInteger("game.exe+0x1234", 999)
```

## `readQword`

```lua
readQword(address) -> integer
```

Reads eight bytes as an unsigned 64-bit integer from the default process. The explicit-handle form
is `target:readQword(address)`. It is suitable for 64-bit integer fields, but a pointer should be
read with `readPointer` when target architecture determines the width.

An invalid address, unavailable process, or failed read raises a Lua error.

```lua
local totalExperience = readQword("game.exe+0x2000")
```

## `writeQword`

```lua
writeQword(address, value) -> true
```

Writes `value` as an eight-byte integer to the default process. The explicit-handle form is
`target:writeQword(address, value)`. The implementation converts the Lua integer to the native
eight-byte representation; it does not expose a separate Lua range check for this operation.

An invalid address, non-integer value, unavailable process, or failed write raises a Lua error.
The operation does not infer whether the destination is a pointer or restore previous bytes.

```lua
writeQword("game.exe+0x2000", 1000000)
```

## `readPointer`

```lua
readPointer(address) -> pointer
```

Reads one pointer-sized unsigned value from the default process. The width is selected from the
attached target: four bytes for a 32-bit target or eight bytes for a 64-bit target. The explicit-
handle form is `target:readPointer(address)`.

An invalid address, unavailable process, or failed read raises a Lua error. A valid null pointer is
returned as numeric `0`; it is not converted to Lua `nil`.

```lua
local object = readPointer("game.exe+0x3000")
if object == 0 then
	return { ok = false, reason = "object is null" }
end
```

## `readFloat`

```lua
readFloat(address) -> number
```

Reads four bytes as an IEEE single-precision floating-point value from the default process. The
explicit-handle form is `target:readFloat(address)`.

An invalid address, unavailable process, or failed read raises a Lua error. Use it only for a field
whose representation is a 32-bit float.

```lua
local speed = readFloat("game.exe+0x1400")
```

## `writeFloat`

```lua
writeFloat(address, value) -> true
```

Writes `value` as a 32-bit floating-point value to the default process. The explicit-handle form is
`target:writeFloat(address, value)`.

An invalid address, non-number value, unavailable process, or failed write raises a Lua error.

```lua
writeFloat("game.exe+0x1400", 2.5)
```

## `readDouble`

```lua
readDouble(address) -> number
```

Reads eight bytes as an IEEE double-precision floating-point value from the default process. The
explicit-handle form is `target:readDouble(address)`.

An invalid address, unavailable process, or failed read raises a Lua error. Use it only for a field
whose representation is a 64-bit double.

```lua
local multiplier = readDouble("game.exe+0x1500")
```

## `writeDouble`

```lua
writeDouble(address, value) -> true
```

Writes `value` as a 64-bit floating-point value to the default process. The explicit-handle form is
`target:writeDouble(address, value)`.

An invalid address, non-number value, unavailable process, or failed write raises a Lua error.

```lua
writeDouble("game.exe+0x1500", 1.25)
```

## `readString`

```lua
readString(address, maxLength, wideChar?) -> string
```

Reads at most `maxLength` characters from the default process and stops at the first null
terminator. With `wideChar` omitted or false, it reads one byte per character. With `wideChar=true`,
it reads UTF-16 code units and converts the result to UTF-8 Lua text. The current explicit process
handle does not expose a string read method.

`maxLength` must be non-negative. An invalid address, unavailable process, or failed read raises a
Lua error. The function does not guarantee that the target memory is null-terminated when the limit
is reached.

```lua
local name = readString("game.exe+0x1800", 32, false)
local displayName = readString("game.exe+0x1900", 32, true)
```

## `writeString`

```lua
writeString(address, text, terminate?, wideChar?) -> true
```

Writes Lua `text` to the default process. Narrow mode writes the string bytes; wide mode converts
UTF-8 text to UTF-16 code units. `terminate` defaults to true and appends one null byte in narrow
mode or one null UTF-16 code unit in wide mode. String methods are registered only as globals; the
current explicit process handle does not expose `readString` or `writeString` methods.

An invalid address, non-string text, unavailable process, or failed write raises a Lua error. The
function does not allocate a destination buffer or check that the destination is large enough.

```lua
writeString("game.exe+0x1800", "Ready", true, false)
writeString("game.exe+0x1900", "Ready", true, true)
```

## Byte Arguments and Scope

Global `writeBytes` accepts either a table or positional integer bytes:

```lua
local address = getAddress("game.exe+0x1234")
local before = readBytes(address, 4, true)
writeBytes(address, 0x90, 0x90, 0x90, 0x90)
return before
```

Memory functions do not save original bytes, create a rollback record, change protection
automatically, or publish UI state. A package that writes memory must validate attachment and
address resolution first, match width, signedness, encoding, and architecture, and restore or
release its changes from `disable`. Use a query result for a one-shot value and a service/feed for
changing state. Explicit process handles are opaque session-bound values; after runtime/session
teardown or userdata invalidation, their methods fail rather than silently selecting another process.

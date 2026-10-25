# Cheat Engine Lua Compatibility

Status: current public API reference.

Host Lua provides the Cheat Engine (CE) functions that trainer scripts use, with CE's behaviour, so
a CE `{$lua}` script runs unchanged inside a package. This page lists those functions and the few
places where the engine deliberately differs. Functions CE has that are not listed here (forms,
the address list, the debugger, `injectDLL`, ...) are not available.

## Behaviour that matches CE

| Area | Behaviour |
| --- | --- |
| Addresses | Integers, `module+offset`, symbols, nested pointer chains such as `[[pStaticBase]+F8]+10`. Bare numbers in an expression are hexadecimal. See [addresses](lua-memory-api.md#addresses). |
| Memory failures | `read*` return `nil`, `write*` return `false`, `writeBytes` returns the byte count (`0`). An unset pointer in a chain is a failure, not an error. See [failures](lua-memory-api.md#failures). |
| `getAddress` / `getAddressSafe` | `getAddress` raises a Lua error for an unknown expression; `getAddressSafe` returns `nil`. |
| Errors | Every engine error can be caught with `pcall`, including failures inside native code. |
| Integer arguments | Fractions are truncated toward zero, numeric strings are converted (CE's `lua_tointeger`). |
| Timers | `createTimer(owner, enabled)`, `createTimer(ms, fn, ...)`, `Interval`/`Enabled`/`OnTimer` and their `get`/`set` methods, case-insensitive members, callback errors do not stop the timer. See [`createTimer`](lua-runtime-utilities-api.md#createtimer). |
| `readBytes`, `readString` | `count` defaults to 1 and `maxLength` to 50; a partly readable range returns its readable prefix. |
| `writeBytes` | Each value keeps its low byte; non-number table entries are skipped. |

## CE helper functions

| Function | Result |
| --- | --- |
| `bAnd(a, b)`, `bOr(a, b)`, `bXor(a, b)`, `bNot(a)` | 64-bit bitwise operations |
| `bShl(a, n)`, `bShr(a, n)` | Shifts; `bShr` is a logical shift and `n` is taken modulo 64, as in CE |
| `wordToByteTable(v)`, `dwordToByteTable(v)`, `qwordToByteTable(v)` | Little-endian byte array of an integer |
| `floatToByteTable(v)`, `doubleToByteTable(v)` | Byte array of a 32-bit or 64-bit float |
| `stringToByteTable(s)` | Byte array of a string |
| `byteTableToWord(t, signed?, start?)`, `byteTableToDword(t, signed?, start?)` | Integer from a byte array, optionally signed, starting at index `start` (default 1) |
| `byteTableToQword(t, start?)` | 64-bit integer from a byte array |
| `byteTableToFloat(t, start?)`, `byteTableToDouble(t, start?)` | Float from a byte array |
| `byteTableToString(t, start?)` | String from a byte array |
| `getTickCount()` | Milliseconds since Windows started (`GetTickCount64`) |
| `print(...)` | Writes the arguments, separated by spaces, to the engine log |
| `showMessage(text)` | Writes `text` to the engine log; there is no dialog |
| `enumModules(pid?)` | `{ {Name=, Address=, Is64Bit=, PathToFile=}, ... }` for the attached process |
| `getModuleSize(name)` | Size of a loaded module, or `nil` |
| `getModuleBase(name)` | Base address of a loaded module, or `nil` (not a CE function; equal to `getAddressSafe(name)` for a module) |
| `speedhack_setSpeed(factor)` | Sets the game speed through the engine game speed (`speed.set`); the package's teardown restores normal speed |
| `speedhack_getSpeed()` | Current game speed factor |
| `readShortInteger`, `writeShortInteger` | CE names of `readByte` and `writeByte` |
| `syntaxcheck` | Global `false`; CE sets it while checking a script |

A byte array is an ordinary Lua table with values `0..255` at indexes 1, 2, .... A missing entry
reads as the end of the array, and the remaining bytes are zero.

## Deliberate differences

| Function | CE | Engine | Why |
| --- | --- | --- | --- |
| `writeString(address, text, x, y)` | `x` = wide string; no terminator is written | `x` = write a terminator (default `true`), `y` = wide string | Existing packages call `writeString(address, text, true)` for a terminated narrow string. Pass `false, true` for a CE-style wide write. |
| `createTimer(false)` / `createTimer(true)` | The boolean is the owner; the timer starts enabled | The boolean is `Enabled` | Existing packages create a stopped timer with `createTimer(false)`. Use `createTimer(nil, enabled)` in new code; both runtimes agree on it. |
| Integer write of a non-number | Writes `0` | Writes nothing and returns `false` | Writing `0` into game memory by mistake is never the intent. |
| `print`, `showMessage` | Lua console, dialog | Engine log | Packages have no CE window. |
| `enumModules` `Is64Bit` | Read from each module's file | The process architecture | Modules of one process share its architecture in practice. |
| Process handle methods (`target:readInteger`, ...) | Not in CE | Raise a Lua error on failure | Engine API, kept unchanged. |

## Running a CE script in a package

A CE `{$lua}` section can be loaded into the package's own environment, where it shares globals
with the package as CE scripts share globals between table entries:

```lua
local chunk = assert(load(ceSource, "=Infinite Health", "t", _ENV))
chunk()
```

Pointer chains, failed reads, timers and helper calls then behave as they do in CE. Prefer native
[hotkeys](native-hotkeys.md) and package settings over CE forms and hotkey objects, which the
engine does not provide.

# Lua API Reference

Status: current public API reference for Hexinton Engine runnables.

Function names are case-sensitive. Bare functions use the command's default process session.

## Process Discovery

| Function | Result |
| --- | --- |
| `findProcess(name)` | First matching process ID, or `nil` |
| `findProcesses(name)` | Lua array of matching process IDs |
| `getProcessIDFromProcessName(name)` | Matching process ID |
| `findWindow(className, title)` | Top-level window handle, or `nil` |
| `getWindowProcessID(handle)` | Window owner process ID |
| `findWindowProcessID(className, title)` | Matching window owner process ID |
| `getDefaultPid()` | Default command process ID |
| `getOpenedProcessID()` | Opened/default process ID |

Process-name matching is case-insensitive on Windows. A name such as `game` may match `game.exe`.

## Process Handles

```lua
local pid = findProcess("game.exe")
local game = openProcess(pid)
local value = game:readInteger("game.exe+0x1234")
game:writeInteger("game.exe+0x1234", value + 1)
```

`openProcess()` uses the default process; `openProcess(pid)` selects the specified process. Handle
methods include `getAddress`, `getAddressSafe`, `readBytes`, `writeBytes`, `readSmallInteger`,
`writeSmallInteger`, `readWord`, `writeWord`, `readInteger`, `writeInteger`, `readQword`,
`writeQword`, `readPointer`, `readFloat`, `writeFloat`, `readDouble`, `writeDouble`,
`readMem`, `writeMem`, `registerSymbol`, and `unregisterSymbol`.

## Address and Symbol Functions

| Function | Purpose |
| --- | --- |
| `getAddress(expression)` | Resolve an address or fail |
| `getAddressSafe(expression)` | Resolve an address with safe failure |
| `registerSymbol(name, address, donotsave?)` | Publish a package-owned symbol |
| `unregisterSymbol(name)` | Remove a package-owned symbol |
| `readPointer(address)` | Read a pointer value |

Expressions may be integer addresses, module expressions such as `game.exe+0x1234`, or pointer
chains such as `[[game.exe+0x1234]+0x18]+0x30`.

## Typed Memory I/O

| Read | Write |
| --- | --- |
| `readBytes(address, count, returnAsTable?)` | `writeBytes(address, ...)` |
| `readSmallInteger(address, signed?)` | `writeSmallInteger(address, value)` |
| `readWord(address, signed?)` | `writeWord(address, value)` |
| `readInteger(address, signed?)` | `writeInteger(address, value)` |
| `readQword(address)` | `writeQword(address, value)` |
| `readFloat(address)` | `writeFloat(address, value)` |
| `readDouble(address)` | `writeDouble(address, value)` |
| `readString(address, maxLength, wideChar?)` | `writeString(address, text, terminate?, wideChar?)` |

Use the operation whose width and representation match the target value. Memory writes require an
attached, valid process session.

## Scanning and Assembly

| Function | Purpose |
| --- | --- |
| `AOBScan(pattern)` | Scan the process |
| `AOBScanUnique(pattern)` | Require one matching address |
| `AOBScanModule(module, pattern)` | Scan one module |
| `AOBScanModuleUnique(module, pattern)` | Require one match in one module |
| `autoAssemble(text, targetself?, disableInfo?)` | Execute Auto Assembler text |
| `fullAccess(address, size)` | Change access for a range |
| `targetIs64Bit()` | Check target architecture |

Use `??` for a wildcard byte in an AOB pattern. Check scan results before reading or writing them.

## Runtime Utilities

| Function | Purpose |
| --- | --- |
| `createTimer(...)` | Create a runtime timer |
| `sleep(milliseconds)` | Pause the current script |
| `process` | Attached process main module name |
| `publishEvent(event, value)` | Publish service state to a declared feed |

Timer callbacks are serialized with script execution. Timers and published service state belong to
the package and must stop or become inactive during `disable`.

## Return Values and Errors

Runnable functions may return JSON-compatible values: `nil`, booleans, numbers, strings, and tables
containing compatible values. Raise an error when compatibility checks fail. Do not continue with a
write after an unresolved address, missing scan result, stale attachment, or invalid argument.
# Lua API Reference

Status: current public API reference for Hexinton Engine runnables.

Function names are case-sensitive. Bare functions use the command's default process session.

The index is organized by function so a reader or assistant can open the exact contract instead of
scanning a capability summary. Each linked entry provides prototype, parameters, return value,
failure behavior, scope, and a focused example.

## Process Discovery

| Function | Detailed contract |
| --- | --- |
| [`findProcess`](lua-process-and-address-api.md#findprocess) | First matching process ID, or `nil` |
| [`findProcesses`](lua-process-and-address-api.md#findprocesses) | Lua array of matching process IDs |
| [`getProcessIDFromProcessName`](lua-process-and-address-api.md#getprocessidfromprocessname) | Matching process ID |
| [`findWindow`](lua-process-and-address-api.md#findwindow) | Top-level window handle, or `nil` |
| [`getWindowProcessID`](lua-process-and-address-api.md#getwindowprocessid) | Window owner process ID |
| [`findWindowProcessID`](lua-process-and-address-api.md#findwindowprocessid) | Matching window owner process ID |
| [`findWindowProcessId`](lua-process-and-address-api.md#findwindowprocessid-and-findwindowpid) | Alias of `findWindowProcessID` |
| [`findWindowPid`](lua-process-and-address-api.md#findwindowprocessid-and-findwindowpid) | Alias of `findWindowProcessID` |
| [`getDefaultPid`](lua-process-and-address-api.md#getdefaultpid) | Default command process ID |
| [`getOpenedProcessID`](lua-process-and-address-api.md#getopenedprocessid) | Opened/default process ID |

Process-name matching is case-insensitive on Windows. A name such as `game` may match `game.exe`.

## Process Handles

```lua
local pid = findProcess("game.exe")
local game = openProcess(pid)
local value = game:readInteger("game.exe+0x1234")
game:writeInteger("game.exe+0x1234", value + 1)
```

| Function/member | Detailed contract |
| --- | --- |
| [`openProcess`](lua-process-and-address-api.md#openprocess) | Create a PID-bound process handle |
| [`openprocess`](lua-process-and-address-api.md#openprocess) | Alias of `openProcess` |
| [`target.pid`](lua-process-and-address-api.md#process-handle-properties-and-methods) | Read the bound PID |
| [`target:getPid()`](lua-process-and-address-api.md#process-handle-properties-and-methods) | Read the bound PID |
| [`target:getPID()`](lua-process-and-address-api.md#process-handle-properties-and-methods) | Alias of `target:getPid()` |
| [`target:getAddress(expression)`](lua-process-and-address-api.md#process-handle-properties-and-methods) | Resolve in the selected process |
| [`target:getAddressSafe(expression)`](lua-process-and-address-api.md#process-handle-properties-and-methods) | Resolve with `nil` failure |
| [`target:registerSymbol(...)`](lua-process-and-address-api.md#process-handle-properties-and-methods) | Register a selected-process symbol |
| [`target:unregisterSymbol(...)`](lua-process-and-address-api.md#process-handle-properties-and-methods) | Remove a selected-process symbol |
| [`target:readBytes(...)`](lua-process-and-address-api.md#process-handle-properties-and-methods) | Read raw bytes |
| [`target:writeBytes(...)`](lua-process-and-address-api.md#process-handle-properties-and-methods) | Write raw bytes |
| [`target:readSmallInteger(...)`](lua-process-and-address-api.md#process-handle-properties-and-methods) | Read two bytes |
| [`target:readWord(...)`](lua-process-and-address-api.md#process-handle-properties-and-methods) | Alias of `target:readSmallInteger(...)` |
| [`target:writeSmallInteger(...)`](lua-process-and-address-api.md#process-handle-properties-and-methods) | Write two bytes |
| [`target:writeWord(...)`](lua-process-and-address-api.md#process-handle-properties-and-methods) | Alias of `target:writeSmallInteger(...)` |
| [`target:readInteger(...)`](lua-process-and-address-api.md#process-handle-properties-and-methods) | Read four bytes |
| [`target:writeInteger(...)`](lua-process-and-address-api.md#process-handle-properties-and-methods) | Write four bytes |
| [`target:readQword(...)`](lua-process-and-address-api.md#process-handle-properties-and-methods) | Read eight bytes |
| [`target:writeQword(...)`](lua-process-and-address-api.md#process-handle-properties-and-methods) | Write eight bytes |
| [`target:readPointer(...)`](lua-process-and-address-api.md#process-handle-properties-and-methods) | Read a target-sized pointer |
| [`target:readFloat(...)`](lua-process-and-address-api.md#process-handle-properties-and-methods) | Read a 32-bit float |
| [`target:writeFloat(...)`](lua-process-and-address-api.md#process-handle-properties-and-methods) | Write a 32-bit float |
| [`target:readDouble(...)`](lua-process-and-address-api.md#process-handle-properties-and-methods) | Read a 64-bit double |
| [`target:writeDouble(...)`](lua-process-and-address-api.md#process-handle-properties-and-methods) | Write a 64-bit double |
| [`target:readMem(...)`](lua-process-and-address-api.md#process-handle-properties-and-methods) | Alias of `target:readBytes(...)` |
| [`target:readmem(...)`](lua-process-and-address-api.md#process-handle-properties-and-methods) | Alias of `target:readBytes(...)` |
| [`target:writeMem(...)`](lua-process-and-address-api.md#process-handle-properties-and-methods) | Alias of `target:writeBytes(...)` |
| [`target:writemem(...)`](lua-process-and-address-api.md#process-handle-properties-and-methods) | Alias of `target:writeBytes(...)` |

`openProcess()` uses the default process; `openProcess(pid)` selects the specified process. The
handle is opaque userdata and is invalidated only with its runtime/session lifetime; it has no
explicit `destroy()` method. Handle integer reads do not accept the global `signed?` argument, and
handle string reads/writes and AOB scans are not registered.

## Address and Symbol Functions

| Function | Purpose |
| --- | --- |
| [`getAddress(expression)`](lua-process-and-address-api.md#getaddress) | Resolve an address or fail |
| [`getAddressSafe(expression)`](lua-process-and-address-api.md#getaddresssafe) | Resolve an address with safe failure |
| [`registerSymbol(name, address, donotsave?)`](lua-process-and-address-api.md#registersymbol) | Publish a package-owned symbol |
| [`unregisterSymbol(name)`](lua-process-and-address-api.md#unregistersymbol) | Remove a package-owned symbol |
| [`readPointer(address)`](lua-memory-api.md#readpointer) | Read a pointer value |

Expressions may be integer addresses, module expressions such as `game.exe+0x1234`, or pointer
chains such as `[[game.exe+0x1234]+0x18]+0x30`.

## Typed Memory I/O

| Read | Write |
| --- | --- |
| [`readBytes(address, count, returnAsTable?)`](lua-memory-api.md#readbytes) | [`writeBytes(address, ...)`](lua-memory-api.md#writebytes) |
| [`readSmallInteger(address, signed?)`](lua-memory-api.md#readsmallinteger) | [`writeSmallInteger(address, value)`](lua-memory-api.md#writesmallinteger) |
| [`readWord(address, signed?)`](lua-memory-api.md#readword) | [`writeWord(address, value)`](lua-memory-api.md#writeword) |
| [`readInteger(address, signed?)`](lua-memory-api.md#readinteger) | [`writeInteger(address, value)`](lua-memory-api.md#writeinteger) |
| [`readQword(address)`](lua-memory-api.md#readqword) | [`writeQword(address, value)`](lua-memory-api.md#writeqword) |
| [`readFloat(address)`](lua-memory-api.md#readfloat) | [`writeFloat(address, value)`](lua-memory-api.md#writefloat) |
| [`readDouble(address)`](lua-memory-api.md#readdouble) | [`writeDouble(address, value)`](lua-memory-api.md#writedouble) |
| [`readString(address, maxLength, wideChar?)`](lua-memory-api.md#readstring) | [`writeString(address, text, terminate?, wideChar?)`](lua-memory-api.md#writestring) |

Use the operation whose width and representation match the target value. Memory writes require an
attached, valid process session.

## Scanning and Assembly

| Function | Purpose |
| --- | --- |
| [`AOBScan(pattern)`](lua-scanning-and-assembly-api.md#aobscan) | Scan the process |
| [`AOBScanUnique(pattern)`](lua-scanning-and-assembly-api.md#aobscanunique) | Require one matching address |
| [`AOBScanModule(module, pattern)`](lua-scanning-and-assembly-api.md#aobscanmodule) | Scan one module |
| [`AOBScanModuleUnique(module, pattern)`](lua-scanning-and-assembly-api.md#aobscanmoduleunique) | Require one match in one module |
| [`autoAssemble(text, targetself?, disableInfo?)`](lua-scanning-and-assembly-api.md#autoassemble) | Execute Auto Assembler text |
| [`fullAccess(address, size)`](lua-scanning-and-assembly-api.md#fullaccess) | Change access for a range |
| [`targetIs64Bit()`](lua-scanning-and-assembly-api.md#targetis64bit) | Check target architecture |

Use `??` for a wildcard byte in an AOB pattern. Check scan results before reading or writing them.

## Runtime Utilities

| Function | Purpose |
| --- | --- |
| [`createTimer(...)`](lua-runtime-utilities-api.md#createtimer) | Create a runtime timer |
| [`sleep(milliseconds)`](lua-runtime-utilities-api.md#sleep) | Pause the current script |
| [`process`](lua-runtime-utilities-api.md#process) | Attached process main module name |
| [`publishEvent(event, value)`](lua-runtime-utilities-api.md#publishevent) | Publish service state to a declared feed |

Timer callbacks are serialized with script execution. Timers and published service state belong to
the package and must stop or become inactive during `disable`.

## Return Values and Errors

Runnable functions may return JSON-compatible values: `nil`, booleans, numbers, strings, and tables
containing compatible values. Raise an error when compatibility checks fail. Do not continue with a
write after an unresolved address, missing scan result, stale attachment, or invalid argument.
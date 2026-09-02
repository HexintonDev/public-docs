# Lua Process and Address API

Status: current public API reference.

The functions on this page are registered native Lua globals. Names are case-sensitive except for
the aliases listed below. Discovery functions inspect Windows processes and windows. Address,
symbol, and memory functions use the command's default process unless called through an explicit
process handle.

## `findProcess`

```lua
findProcess(processName) -> pid | nil
```

Returns the first process ID whose executable name matches `processName`. Windows matching is
case-insensitive and accepts both `game` and `game.exe`. An empty name raises a Lua argument error;
no matching process returns `nil`. The result is only a PID; it does not attach or open a process
handle.

```lua
local pid = findProcess("game.exe")
if not pid then
    return { ok = false, reason = "game is not running" }
end
return { ok = true, pid = pid }
```

## `findProcesses`

```lua
findProcesses(processName) -> { pid1, pid2, ... }
```

Returns a 1-based Lua array containing every matching process ID. An empty name raises a Lua
argument error; no matching process returns an empty array. The array does not attach to any result
process.

```lua
local pids = findProcesses("game")
for index, pid in ipairs(pids) do
    print(index, pid)
end
```

## `getProcessIDFromProcessName`

```lua
getProcessIDFromProcessName(processName) -> pid | nil
```

Registered alias of `findProcess`. It returns the first case-insensitive process-name match, raises
for an empty name or discovery exception, and returns `nil` when no process matches.

## `findWindow`

```lua
findWindow(className?, title?) -> windowHandle | nil
```

Finds a top-level window using an optional class-name filter and optional title filter. At least one
filter must be non-empty; both filters omitted, `nil`, or empty raises a Lua error. A missing window
returns `nil`, while a discovery exception raises a Lua error. The returned integer is a Windows
window handle, not a process ID.

```lua
local window = findWindow(nil, "Example Game")
if not window then
    return { ok = false, reason = "window not found" }
end
return { window = window }
```

## `getWindowProcessID`

```lua
getWindowProcessID(windowHandle) -> pid | nil
```

Returns the process ID that owns `windowHandle`. A zero or unknown handle returns `nil`; an
available window returns its owning PID. The argument must be an integer, and a discovery failure
raises a Lua error.

```lua
local window = findWindow(nil, "Example Game")
local pid = window and getWindowProcessID(window)
if not pid then error("window has no accessible owner") end
local target = openProcess(pid)
```

## `findWindowProcessID`

```lua
findWindowProcessID(className?, title?) -> pid | nil
```

Finds a window using the supplied optional filters and returns its owning process ID. Both filters
cannot be empty. A missing window or a window without an owner returns `nil`; invalid discovery
input or a discovery exception raises a Lua error.

## `findWindowProcessId` and `findWindowPid`

```lua
findWindowProcessId(className?, title?) -> pid | nil
findWindowPid(className?, title?) -> pid | nil
```

Both names are registered aliases of `findWindowProcessID` and have identical filters, return
values, errors, and default-process independence.

## `getDefaultPid`

```lua
getDefaultPid() -> pid
```

Returns the PID of the command's current default process session. It does not discover a new
process, open a different PID, or validate a user-supplied process name. The current runtime must
have an attached process session for the returned PID to be useful.

## `getOpenedProcessID`

```lua
getOpenedProcessID() -> pid
```

Registered alias of `getDefaultPid`. It returns the current command/default session PID.

## `openProcess`

```lua
openProcess() -> processHandle
openProcess(pid) -> processHandle
```

Creates an opaque process handle bound to the command's default PID or to the supplied PID. A PID
of `0`, an invalid PID, an unavailable session, or a process-access failure raises a Lua error. The
function does not change the command's default process; the handle selects the process for methods
called through it.

```lua
local pid = findProcess("game.exe")
if not pid then error("game is not running") end
local target = openProcess(pid)
return { pid = target.pid, samePid = target:getPid() == pid }
```

## Process Handle Properties and Methods

`openProcess` returns a userdata value with the following process-specific surface. All methods use
the handle's PID and fail if the process session is unavailable or the handle has been destroyed.
String reads/writes and AOB scan methods are not exposed on this handle. The handle has no explicit
`destroy()` method; it is runtime userdata and becomes unusable only when its runtime/session is
gone or the userdata has been invalidated.

| Handle member | Prototype | Return | Failure or alias rule |
| --- | --- | --- | --- |
| `pid`, `Pid`, `PID`, `ProcessID` | `target.pid` | Bound PID | Destroyed handle raises a Lua error |
| `getPid`, `getPID` | `target:getPid()` | Bound PID | Aliases; destroyed/unavailable handle raises |
| `getAddress` | `target:getAddress(expression)` | Numeric address | Unresolved expression or session failure raises |
| `getAddressSafe` | `target:getAddressSafe(expression)` | Address or `nil` | Converts resolution failure to `nil` |
| `registerSymbol` | `target:registerSymbol(name, address, donotsave?)` | `true` | Invalid name/address or symbol conflict raises |
| `unregisterSymbol` | `target:unregisterSymbol(name)` | Boolean removed flag | Invalid name raises; missing symbol returns `false` |
| `readBytes`, `readMem`, `readmem` | `target:readBytes(address, count, returnAsTable?)` | Multiple bytes or table | Aliases; invalid count/address/read raises |
| `writeBytes`, `writeMem`, `writemem` | `target:writeBytes(address, bytes...)` | `true` | Aliases; bytes must be valid `0..255` values |
| `readSmallInteger`, `readWord` | `target:readSmallInteger(address)` | Unsigned 16-bit integer | `readWord` alias; invalid read raises |
| `writeSmallInteger`, `writeWord` | `target:writeSmallInteger(address, value)` | `true` | `writeWord` alias; non-integer or failed write raises; no separate Lua range check |
| `readInteger` | `target:readInteger(address)` | Unsigned 32-bit integer | Invalid read raises |
| `writeInteger` | `target:writeInteger(address, value)` | `true` | Non-integer or failed write raises; no separate Lua range check |
| `readQword` | `target:readQword(address)` | Unsigned 64-bit integer | Invalid read raises |
| `writeQword` | `target:writeQword(address, value)` | `true` | Non-integer or failed write raises; no separate Lua range check |
| `readPointer` | `target:readPointer(address)` | Target-sized pointer | Invalid read raises |
| `readFloat` | `target:readFloat(address)` | 32-bit float | Invalid read raises |
| `writeFloat` | `target:writeFloat(address, value)` | `true` | Invalid value/write raises |
| `readDouble` | `target:readDouble(address)` | 64-bit double | Invalid read raises |
| `writeDouble` | `target:writeDouble(address, value)` | `true` | Invalid value/write raises |

Handle integer reads do not expose the global `signed?` argument. The process-handle byte reader
follows the global byte reader's return shape: without `true` it returns one integer per byte, and
with `true` it returns a 1-based table. A handle is session-bound and does not provide rollback,
automatic protection changes, or automatic cleanup of memory writes.

```lua
local target = openProcess(findProcess("game.exe"))
local address = target:getAddressSafe("game.exe+0x1234")
if not address then error("address unavailable") end
local before = target:readBytes(address, 4, true)
target:writeBytes(address, { 0x90, 0x90, 0x90, 0x90 })
return { pid = target.pid, original = before }
```

## `getAddress`

```lua
getAddress(expression) -> address
```

Resolves a numeric address, module expression, registered symbol, offset, or supported nested
pointer chain in the default process/script context. An unresolved expression, unavailable session,
or invalid expression raises a Lua error. This function does not read memory; it only resolves an
address.

```lua
local address = getAddress("game.exe+0x1234")
local nested = getAddress("[[game.exe+0x2000]+0x18]+0x30")
return { direct = address, nested = nested }
```

## `getAddressSafe`

```lua
getAddressSafe(expression) -> address | nil
```

Resolves an address in the default process/script context and converts resolution failures into
`nil`. Invalid Lua argument types can still raise an argument error. Check the result before any
read, write, scan-dependent patch, or symbol registration.

```lua
local address = getAddressSafe("game.exe+0x1234")
if not address then
    return { ok = false, reason = "unsupported game build" }
end
return { ok = true, address = address }
```

## `registerSymbol`

```lua
registerSymbol(name, address, donotsave?) -> true
```

Publishes a user-defined symbol in the current script/process context. `address` may be numeric or
an address expression. The optional `donotsave` boolean controls whether the symbol is persisted by
the underlying symbol registration record. The symbol is package-owned and should be unregistered
before its backing allocation or patch is released.

An invalid name, unresolved address, symbol conflict, unavailable session, or registration failure
raises a Lua error. Registration does not allocate memory or validate that the address contains safe
machine code.

```lua
local address = getAddress("game.exe+0x1234")
registerSymbol("healthAddress", address, true)
```

## `unregisterSymbol`

```lua
unregisterSymbol(name) -> boolean
```

Removes a symbol owned by the current script context and returns `true` when a symbol was removed.
It returns `false` when the name is not currently registered. An invalid name or repository failure
raises a Lua error. Call it before deallocating the memory or removing the address that the symbol
describes.

```lua
if unregisterSymbol("healthAddress") then
    print("symbol removed")
end
```

## Scope and Threading

These calls execute on the Lua runtime worker. The current script context controls package-local
labels, symbols, globals, and registered timers. A process handle selects a process session but does
not change package visibility or the command's default process. Reads and writes are not reversible
automatically; package `disable` logic must unregister symbols, restore patches, release handle
references, destroy timers as applicable, and release package-owned resources.

See [Pointers and Address Chains](pointers.md) for expression syntax and [Lua Memory API](lua-memory-api.md)
for the complete global and handle memory operation contracts.

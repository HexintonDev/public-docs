# Lua Process and Address API

Status: current public API reference.

All functions in this page are registered native Lua globals. Names are case-sensitive except for
explicit aliases. Bare address functions use the command's default process session.

## Process Discovery

| Prototype | Return | Failure |
| --- | --- | --- |
| `findProcess(processName)` | First matching pid, or `nil` | Empty name or discovery exception raises a Lua error |
| `findProcesses(processName)` | 1-based Lua array of pids | Empty name or discovery exception raises a Lua error |
| `getProcessIDFromProcessName(processName)` | Same result as `findProcess` | Same as `findProcess` |
| `findWindow(className?, title?)` | Window handle, or `nil` | Both filters empty or discovery exception raises a Lua error |
| `getWindowProcessID(windowHandle)` | Owning pid, or `nil` | Zero/unknown handle returns `nil` |
| `findWindowProcessID(className?, title?)` | Owning pid, or `nil` | Both filters empty or discovery exception raises a Lua error |
| `findWindowProcessId(className?, title?)` | Alias of `findWindowProcessID` | Same behavior |
| `findWindowPid(className?, title?)` | Alias of `findWindowProcessID` | Same behavior |
| `getDefaultPid()` | Command pid | Returns the current runtime default |
| `getOpenedProcessID()` | Command pid | Alias of `getDefaultPid` |

Windows process-name matching is case-insensitive and accepts `game` or `game.exe`.

```lua
local pid = findProcess("game.exe")
if not pid then
    return { ok = false, reason = "not found" }
end
return { pid = pid }
```

## Explicit Process Handles

| Prototype | Return | Failure |
| --- | --- | --- |
| `openProcess(pid?)` | Opaque process handle | pid `0`, invalid pid, or unavailable session raises a Lua error |
| `handle.pid` | Handle pid | Handle destruction raises a Lua error |
| `handle:getAddress(expression)` | Address | Unresolved expression raises a Lua error |
| `handle:getAddressSafe(expression)` | Address or `nil` | Resolution failure returns `nil` |

A missing pid argument selects the command's default process. A supplied pid selects that process;
the pid must not be `0`.

```lua
local target = openProcess(findProcess("game.exe"))
local address = target:getAddressSafe("game.exe+0x1234")
if not address then error("address unavailable") end
return { pid = target.pid, address = address }
```

## Address Resolution

| Prototype | Return | Failure |
| --- | --- | --- |
| `getAddress(expression)` | Numeric address | Unresolved module, symbol, or pointer chain raises a Lua error |
| `getAddressSafe(expression)` | Numeric address or `nil` | Converts resolution failure to `nil` |
| `readPointer(address)` | Target-sized pointer | Invalid read raises a Lua error |

Expressions may be numeric addresses, module expressions, registered symbols, offsets, or the
supported nested pointer-chain syntax. See [Pointers and Address Chains](pointers.md).

## Scope and Thread

These calls execute inside the Lua runtime worker. The current script context controls package-local
labels, symbols, and globals. Process handles select a session; they do not change package visibility.

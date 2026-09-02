# Lua Runtime Utilities API

Status: current public API reference.

These globals execute on the Lua runtime worker. Timer callbacks are serialized with submitted Lua
calls, and event publication detaches the payload before it is passed to the host publisher. Values
returned from a runnable and values sent through `publishEvent` must be JSON-compatible Lua values.

## `createTimer`

```lua
createTimer() -> timerHandle
createTimer(enabled) -> timerHandle
createTimer(interval, callback) -> timerHandle
```

Creates a timer userdata owned by the current runtime. With no arguments it creates an enabled
repeating timer with the default interval of 1000 milliseconds. With one boolean argument, the
boolean sets the initial `Enabled` state and the timer keeps the default interval. With an integer
interval and a function callback, it creates an enabled one-shot timer. The interval must be
non-negative. The two-argument overload requires a callback function in the second position.

The handle exposes `Interval`, `Enabled`, and `OnTimer` properties, plus `destroy()`. Assigning a
new interval while enabled reschedules the next firing. Enabling a disabled timer schedules it from
the current time. Assigning `OnTimer=nil` removes the callback; assigning a non-function raises a
Lua error. Destroying an already destroyed or missing timer is harmless. A callback exception is
logged and disables that timer; it is not propagated into the caller that happened to be waiting.

```lua
local timer = createTimer(false)
timer.Interval = 250
timer.OnTimer = function()
    publishEvent("health_changed", {
        current = readInteger(getAddress("game.exe+0x1234"))
    })
end
timer.Enabled = true
```

For a package service, retain the handle and destroy it from `disable` before releasing script
state:

```lua
local timer

function enable()
    timer = createTimer(false)
    timer.Interval = 250
    timer.OnTimer = function()
        publishEvent("health_changed", readInteger("game.exe+0x1234"))
    end
    timer.Enabled = true
end

function disable()
    if timer then
        timer.destroy()
        timer = nil
    end
end
```

## Timer Handle Properties

| Member | Read | Write | Behavior |
| --- | --- | --- | --- |
| `Interval` | Integer milliseconds | Non-negative integer | Reschedules an enabled timer from now |
| `Enabled` | Boolean | Boolean | Starts or stops future callbacks |
| `OnTimer` | Function or `nil` | Function or `nil` | Replaces or removes the callback |
| `destroy` | Function | Call as `timer:destroy()` | Removes the timer and releases its callback reference |

The handle is a runtime userdata, not a serializable result value. Do not return it from a runnable
or publish it in an event payload. The one-shot overload destroys itself after its callback returns
successfully. A repeating timer remains scheduled until disabled, destroyed, or disabled because
its callback failed.

## `sleep`

```lua
sleep(milliseconds) -> no values
```

Blocks the current Lua operation for a non-negative integer number of milliseconds and returns no
Lua values. A negative duration raises a Lua error. Because the operation runs on the Lua worker,
`sleep` also delays other submitted script calls and timer callbacks in that runtime. Use short waits
only; timers are preferred for repeatable service work.

```lua
sleep(10)
return { ready = true }
```

## `publishEvent`

```lua
publishEvent(eventName) -> no values
publishEvent(eventName, value) -> no values
```

Publishes an event for the current process, script context, and service runnable. `eventName` must
be a non-empty string. The optional value defaults to `nil`; when supplied, it must be a detached,
JSON-compatible Lua value: `nil`, boolean, number, string, an array with positive consecutive
integer keys, or an object with string keys. Cyclic tables, tables that mix array and object keys,
holes, unsupported userdata, functions, and threads raise a Lua error.

Publication requires an event publisher to be configured by the host. Without one, the function
raises a Lua error. A publisher exception is reported as a Lua error. The function is not a command
return value and does not wait for a UI subscriber to consume the event.

```lua
publishEvent("health_changed", {
    current = readInteger("game.exe+0x1234"),
    source = "trainer"
})
```

## `process`

```lua
process -> string
```

`process` is a predefined global string containing the attached process's main module name. It is
not a process handle, PID, table, or callable function. Use `getDefaultPid()` for the current PID,
`openProcess(pid)` for an explicit handle, and the process/address APIs for discovery and address
resolution.

```lua
return { module = process, pid = getDefaultPid() }
```

## `targetIs64Bit`

```lua
targetIs64Bit() -> boolean
```

Returns whether the attached default process has eight-byte pointers. It is useful for choosing
pointer widths and architecture-specific assembly, but does not alter the target or validate a
script's assembly text. See [Lua Scanning and Assembly API](lua-scanning-and-assembly-api.md).

## Runtime Rules and Result Values

Timer callbacks execute on the Lua runtime worker, not the application UI thread. Keep callbacks
short, avoid `sleep` in callbacks, and do not call UI APIs directly from them. Script destruction
automatically removes timers owned by that script; explicit `destroy` in `disable` still makes
ownership clear and releases callback references promptly.

Runnable return values must be JSON-compatible: `nil`, booleans, numbers, strings, and tables made
from compatible values. Cyclic or unsupported values fail during result conversion. A timer handle,
process handle, scan list, function, or thread is not a valid runnable result or event payload.

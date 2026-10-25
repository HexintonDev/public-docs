# Lua Runtime Utilities API

Status: current public API reference.

These globals execute on the Lua runtime worker. Timer callbacks are serialized with submitted Lua
calls, and event publication detaches the payload before it is passed to the host publisher. Values
returned from a runnable and values sent through `publishEvent` must be JSON-compatible Lua values.

These are host-Lua APIs. For `hex.query`, `hex.action`, `hex.startService`, `hex.null` and `hex.array`, see the [dependency runnable reference](dependency-runnable-api.md). UE4SS packages instead use [managed game-thread helpers](ue4ss-runtime.md#managed-package-api), including `hex.every` and `hex.onCleanup`; ordinary host `createTimer` is not installed in the UE interpreter.

## `loadfile` and `dofile`

```lua
loadfile(filename?, mode?, environment?) -> chunk | nil, error
dofile(filename?) -> all chunk return values
```

Filenames are UTF-8, including Unicode Windows directories and filenames. Relative filenames
resolve against the process working directory. A missing filename reads stdin using ordinary Lua
behavior. `loadfile` compiles without invoking the chunk; its optional `mode` uses Lua's `t`, `b`
or `bt` rules. When supplied, the third argument becomes the chunk's environment. Omitting it
uses the standard global environment; it does not inherit the calling package's local environment.

`loadfile` returns `nil` plus the actual load error; `dofile` raises load/execution errors and
forwards all results, including nil slots. UTF-8 BOM and a leading shebang line are supported;
text source line numbers remain accurate. `dofile` supports yielding when called in a coroutine.

Package-local `require` retains its existing module-ID, cache and scope rules and supports a
Unicode package root. This does not change filename handling for other stock Lua libraries such
as `io.open`, `os.rename` or native-library loaders. The host's JSON protocol accepts ordinary
UTF-8 text and `\uXXXX` escapes, including valid surrogate pairs; malformed pairs are parse errors.

Game calls, target allocation and the CE worker helpers `synchronize` and `queue` are in
[Lua game calls and worker helpers](lua-game-calls-and-worker-api.md).

## `createTimer`

```lua
createTimer(interval, callback, ...) -> timerHandle   -- one-shot
createTimer(enabled) -> timerHandle                   -- engine form, one boolean
createTimer(owner?, enabled?) -> timerHandle          -- CE form
```

Timers follow Cheat Engine. The arguments are matched in this order:

1. An integer `interval` followed by a function: a one-shot timer. After `interval` milliseconds it
   calls `callback(...)` once with the extra arguments (not with the timer), then destroys itself,
   whether or not the callback succeeded.
2. Exactly one boolean: a repeating timer whose `Enabled` state is that boolean. This is the
   engine's original form and keeps working; CE itself would read the boolean as an owner and start
   the timer enabled.
3. Anything else is CE's `createTimer(owner, enabled)`: `owner` is ignored (the engine has no forms
   or components), `enabled` defaults to `true`, and `nil` also means `true`. `createTimer()`,
   `createTimer(nil)` and `createTimer(MainForm)` create an enabled timer; `createTimer(nil, false)`
   creates a disabled one.

A repeating timer starts with an `Interval` of 1000 milliseconds and does nothing until it has an
`OnTimer` callback. Each call receives the timer as its argument (`sender`).

```lua
local timer = createTimer(nil, false)
timer.Interval = 250
timer.OnTimer = function(sender)
    publishEvent("health_changed", { current = readInteger("[[pPlayer]+18]+10") })
end
timer.Enabled = true

createTimer(2000, function(message) print(message) end, "two seconds later")
```

For a package service, retain the handle and destroy it from `disable`:

```lua
local timer

function enable()
    timer = createTimer(nil, false)
    timer:setInterval(250)
    timer.OnTimer = function()
        publishEvent("health_changed", readInteger("game.exe+1234"))
    end
    timer:setEnabled(true)
end

function disable()
    if timer then
        timer:destroy()
        timer = nil
    end
end
```

## Timer Handle Properties and Methods

| Member | Behavior |
| --- | --- |
| `Interval` | Milliseconds between calls. Setting it on an enabled timer restarts the countdown. A repeating timer with `Interval` 0 never fires (as in CE). |
| `Enabled` | `true` while the timer is scheduled. Enabling a disabled timer starts the countdown from now. |
| `OnTimer` | Function called as `OnTimer(sender)`, or `nil` to remove it. Assigning another type raises a Lua error. |
| `getInterval()`, `setInterval(ms)` | Method forms of `Interval` |
| `getEnabled()`, `setEnabled(enabled)` | Method forms of `Enabled` |
| `getOnTimer()`, `setOnTimer(fn)` | Method forms of `OnTimer` |
| `destroy()` | Removes the timer and releases its callback. Calling it again is harmless. |

Member names are case-insensitive, as in CE: `timer.enabled` and `timer.Enabled` are the same.
After `destroy()`, the properties read `nil` and the other methods raise an error.

**Errors in a callback.** As in CE, an error raised by `OnTimer` does not stop the timer: it is
written to the engine log and the timer runs again at its next interval. Repeated errors from the
same timer are logged at most once every five seconds. A callback that should stop on failure must
do so itself, for example `sender.Enabled = false` before raising.

The handle is a runtime userdata, not a serializable result value. Do not return it from a runnable
or publish it in an event payload. Script teardown destroys every timer the script created.

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

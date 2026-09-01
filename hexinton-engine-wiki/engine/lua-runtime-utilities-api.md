# Lua Runtime Utilities API

Status: current public API reference.

## Timer and Scheduling Functions

### `createTimer`

```lua
createTimer() -> timer
createTimer(enabled) -> timer
createTimer(interval, callback) -> timer
```

Creates a timer. The two-argument overload is a one-shot timer; service code commonly creates a
disabled timer, assigns `Interval` and `OnTimer`, then enables it. Destroy the handle during disable.

### `sleep`

```lua
sleep(milliseconds) -> nil
```

Blocks the current Lua operation. Use short waits only; a service should generally use a timer so
its lifecycle remains controllable.

| Prototype | Return | Ownership/failure |
| --- | --- | --- |
| `createTimer()` | Enabled repeating timer handle | Creates an enabled timer with the default interval |
| `createTimer(enabled)` | Timer handle | The boolean controls initial enabled state |
| `createTimer(interval, callback)` | Enabled one-shot timer handle | Interval must be non-negative and callback must be a function |
| `sleep(milliseconds)` | No value | Blocks the current Lua operation; use short waits only |
| `publishEvent(event, value)` | No value | Invalid event/value or publication failure raises a Lua error |
| `process` | Main module name string | Available for the attached default process |
| `targetIs64Bit()` | Boolean | See [scanning and assembly](lua-scanning-and-assembly-api.md) |

### `publishEvent`

```lua
publishEvent(event, value) -> nil
```

Publishes a JSON-compatible value to a declared service feed. Invalid event data or publication
failure raises a Lua error.

The timer object exposes the registered handle properties. The `(interval, callback)` overload is
one-shot; common service usage calls `createTimer(false)`, assigns `Interval` and `OnTimer`, enables
the handle, and destroys it during disable.

```lua
local timer

function enable()
    timer = createTimer(false)
    timer.Interval = 250
    timer.OnTimer = function()
        publishEvent("health_changed", {
            current = readInteger(getAddress("game.exe+0x1234"))
        })
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

## Runtime Rules

Timer callbacks execute on the Lua runtime worker and are serialized with script calls. A callback
must not assume it runs on the application UI thread or call UI APIs directly. Keep callbacks short,
avoid waiting for application work, and stop them before releasing script state.

`publishEvent` is intended for a declared service/feed. Publish an initial state before periodic
updates and publish only meaningful changes. The event is not a command result and must be scoped to
the current package/session attachment.

## Return Values

Runnable return values must be JSON-compatible: `nil`, booleans, numbers, strings, and tables made
from compatible values. Cyclic or unsupported values are failures at result conversion. Raise or
return an explicit failure before continuing after an invalid address, missing scan, stale session,
or invalid argument.

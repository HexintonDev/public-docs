# Game Speed API

Status: implemented host-Lua API, reviewed against the engine on 2026-11-02.

```lua
speed.set(2.0)
local factor = speed.get()
speed.set(1.0)
```

`speed.set(factor)` accepts a finite number from **0.1 through 10**, including both endpoints.
`1.0` means normal speed. It returns no value and raises a catchable Lua error on failure.
`speed.get()` returns the session's last acknowledged factor, initially `1.0`; it is not a
measurement of the game's actual update rate. Frame caps, game logic and engine settings can limit
the visible effect.

The engine uses a configured native speed provider when available, otherwise supported Unity
time-scale access on the game thread, or clock hooks. A failure after a Unity operation might have
run is reported rather than falling back and potentially scaling the game twice. The API does not
provide a package option to force a backend or bypass a failed operation.

## Package ownership and cleanup

Speed is a session-wide setting. The most recent setter owns restoration; packages should not
compete to control it. Restore normal speed in the feature's off action and in `disable`:

```lua
function enable() end

function setSpeed(params)
    speed.set(params.factor)
end

function normalSpeed()
    speed.set(1.0)
end

function disable()
    speed.set(1.0)
end
```

Script destruction also releases the owning script's speed setting, and runtime/session teardown
attempts restoration. Keep explicit cleanup so feature behavior is clear; an unsuccessful teardown
is not proof that the game has returned to normal speed.

`speedhack_setSpeed(factor)` and `speedhack_getSpeed()` are CE compatibility aliases for the same
engine setting and ownership. The aliases do not load a trainer's speed DLL. Prefer `speed.set/get`
in new packages. A game-specific field controlling day length or simulation speed remains a
separate operation requiring its own validation and restoration.

Errors include `speed_factor_invalid`, `speed_unity_failed` and `speed_hooks_failed`, as well as
helper or provider failures. Check real game time in the supported game/build rather than treating
a successful setter or getter as a measured speed guarantee.

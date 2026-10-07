# Unreal Player Location and Host-Lua Consumer

Status: copyable sample for the implemented UE4SS and dependency-call contracts, reviewed on 2026-10-07. Requires a prepared UE4SS runtime and shared `UEHelpers`. The transport is fixture-tested; the pawn lookup must be verified in your game's active world. This is not evidence that Palworld coordinates or other game-specific features have been qualified.

This sample exposes a query, a finite capture action and a repeating service. A second ordinary Lua package consumes the query/action. It has no memory offsets, game-build hashes or special file format.

## Provider layout and manifest

```text
ue4ss.experiment/
  package.json
  ue4ss/main.lua
ue4ss.consumer/
  package.json
  lua/main.lua
```

`ue4ss.experiment/package.json`:

```json
{
  "id": "ue4ss.experiment",
  "version": "1.0.0",
  "dependencies": [],
  "runtime": {
    "runnables": [
      {"id":"enable","kind":"enable","runtime":"ue4ss","entryFile":"ue4ss/main.lua","entrySymbol":"enable"},
      {"id":"disable","kind":"disable","runtime":"ue4ss","entryFile":"ue4ss/main.lua","entrySymbol":"disable"},
      {"id":"capture","kind":"action","runtime":"ue4ss","entryFile":"ue4ss/main.lua","entrySymbol":"capture"},
      {"id":"status","kind":"query","runtime":"ue4ss","entryFile":"ue4ss/main.lua","entrySymbol":"status"},
      {"id":"watch","kind":"service","runtime":"ue4ss","entryFile":"ue4ss/main.lua","entrySymbol":"watch"}
    ]
  }
}
```

`ue4ss.experiment/ue4ss/main.lua`:

```lua
local UEHelpers = require("UEHelpers")
local captured

local function position()
    local controller = UEHelpers.GetPlayerController()
    if not controller or not controller:IsValid() then return nil end
    local pawn = controller.Pawn
    if not pawn or not pawn:IsValid() then return nil end
    local value = pawn:K2_GetActorLocation()
    return {x=value.X, y=value.Y, z=value.Z}
end

function enable()
    local engine = FindFirstOf("Engine")
    assert(engine and engine:IsValid(), "Unreal engine object is unavailable")
    hex.onCleanup(function() captured = nil end)
end

function disable() captured = nil end
function capture() captured = position() end

function status()
    local engine = FindFirstOf("Engine")
    return {
        engineValid=engine and engine:IsValid() or false,
        location=position() or hex.null,
        captured=captured or hex.null
    }
end

function watch(params)
    local interval = params.intervalMs or 250
    assert(type(interval) == "number" and interval >= 25, "Invalid interval")
    publishEvent("ue4ss_status", status())
    return hex.every(interval, function()
        publishEvent("ue4ss_status", status())
    end)
end
```

The controller/pawn is resolved for each observation, and the vector is copied into a plain table. No pawn means `location: null`; it is a successful state such as a menu/loading observation. `capture` retains a detached observation, which cleanup clears. It does not teleport or modify inventory.

The service publishes an initial value and then periodic values on the game thread. Returning the cancellation function registers it for package cleanup; the timer is also managed. The interval assertion checks a rule not established by a simple integer parameter schema.

## Consumer manifest and code

`ue4ss.consumer/package.json`:

```json
{
  "id": "ue4ss.consumer",
  "version": "1.0.0",
  "dependencies": [{"id":"ue4ss.experiment","version":"^1.0.0"}],
  "runtime": {
    "runnables": [
      {"id":"enable","kind":"enable","runtime":"lua","entryFile":"lua/main.lua","entrySymbol":"enable"},
      {"id":"disable","kind":"disable","runtime":"lua","entryFile":"lua/main.lua","entrySymbol":"disable"},
      {"id":"location","kind":"query","runtime":"lua","entryFile":"lua/main.lua","entrySymbol":"location"},
      {"id":"capture","kind":"action","runtime":"lua","entryFile":"lua/main.lua","entrySymbol":"capture"},
      {"id":"watch","kind":"service","runtime":"lua","entryFile":"lua/main.lua","entrySymbol":"watch"}
    ]
  }
}
```

`ue4ss.consumer/lua/main.lua`:

```lua
function enable() end
function disable() end

function location()
    return hex.query("ue4ss.experiment", "status", {}).location
end

function capture()
    hex.action("ue4ss.experiment", "capture", {})
end

function watch(params)
    hex.startService("ue4ss.experiment", "watch", {
        intervalMs=params.intervalMs or 250
    })
end
```

The planner enables the provider before the consumer. `location` receives a detached host-Lua value; no UE API is installed in this interpreter. The wrapper service starts the provider's service, whose observations still identify `ue4ss.experiment` / `watch` / `ue4ss_status`. It does not publish a consumer event or implicitly create a consumer feed. For a real UI, declare provider service/feed bindings using [ViewModels and feeds](../view-models-and-feeds.md).

## Exercise and verify

1. Supply the prepared framework in the actual applied scripts root; see [loading prerequisites](../ue4ss-runtime.md#loading-and-prerequisites).
2. Enable the consumer, then invoke its `location` query. In a menu expect null; in a verified active world expect `{x,y,z}`.
3. Invoke consumer `capture`, then query provider `status` to inspect the stored observation. Action success itself contains no captured coordinates.
4. Start `watch` once and inspect provider events through the host feed path. A second start must fail with `service_already_running`.
5. Respawn/travel and verify fresh coordinates or null. Verify the helper selects the intended local player in your game.
6. Disable the consumer. If no other owner retains the provider, cleanup stops its timer. If another owner retains it, it continues. Re-enable and confirm there is no duplicate timer.
7. Test failed source/encoding, explicit disable and detach. Inspect structured errors plus the framework log. Never repeat a mutation automatically after `ue4ss_completion_unconfirmed`.

Pure host-Lua transformations can use ordinary Lua unit cases. Studio has no separate `ue4ss` case runtime in its test-declaration format; fixture tests and live-game checks serve different purposes. See [UE4SS testing](../testing.md#ue4ss-packages) and [dependency error handling](../dependency-runnable-api.md#values-and-errors).

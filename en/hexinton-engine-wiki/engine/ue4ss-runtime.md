# UE4SS Runtime

Status: implemented runtime authoring contract, reviewed on 2026-10-07. Availability depends on the client build and a prepared UE4SS framework bundle. Automatic delivery of that bundle into the application's applied scripts root is not yet implemented. The runtime is intended for production use; complete client-to-game qualification is still required.

UE4SS runs Lua inside an attached Unreal game process. HexMod uses it to resolve Unreal objects, read reflected properties, call reflected functions and register managed callbacks. Ordinary engine Lua continues to run in a separate host-side interpreter with memory, address, symbol and assembly APIs. You can combine the two through [dependency runnable calls](dependency-runnable-api.md).

## Choosing this runtime

Use `runtime: "ue4ss"` for operations expressed through Unreal objects and functions. Use ordinary `runtime: "lua"` for memory scans, addresses, symbols and existing host-side logic. Their APIs and global variables are separate. An Unreal object is not a host-Lua process handle or symbol.

No special script file extension is required. UE4SS entries are `.lua` files, conventionally beneath `ue4ss/`, selected by their manifest runtime. The same package lifecycle and five runnable kinds apply:

| Kind | Entry behavior | Completion |
| --- | --- | --- |
| `enable` | Initialize package state/resources | Lifecycle initialization completed |
| `disable` | Explicitly undo reversible work | Disable attempted, followed by resource cleanup |
| `action` | Perform a finite requested operation | Function returned; its return value is discarded |
| `query` | Return a detached observation | Result decoded into the existing command `result` |
| `service` | Register callbacks/timers and return promptly | Startup completed; observations/failures arrive later |

There is no new UI command format. Trainer controls and custom interfaces bind declared actions/queries/feeds through the existing host contracts. See [trainer controls](trainer-controls.md) and [ViewModels and feeds](view-models-and-feeds.md).

## Loading and prerequisites

The target must be a running native Windows x64 game, and the client must have sufficient process access. The selected UE4SS distribution must work with the game. On first UE4SS execution, the engine verifies/stages its prepared framework assets, loads the DLL into the game and waits for a matching bridge session. A separate user-run injector or pre-launch installation is not required by this loading path.

The framework profile must already be at `.frameworks/ue4ss/profile.json` beneath the native context's actual applied scripts root. Bridge assets must accompany the native host module. Saving a package or copying its `.lua` file alone does not supply these prerequisites. Until automatic application delivery exists, a build/developer-prepared installation is needed; do not assume ordinary Apply provisions it.

One connection is shared by UE packages in that process. Another already-loaded UE4SS installation is rejected unless it is the matching staged HexMod installation. Detach releases package work and the connection; it does not unload the DLL. An uncertain loader attempt requires game restart. An established matching DLL can be reused with a fresh attachment session.

Framework SHA-256 checks verify the supplied assets. There is no game executable hash allowlist: game updates are evaluated by loading and executing the runtime. Passing integrity checks is not evidence that every reflected game API works.

## Manifest and entry functions

```json
{
  "id": "unreal.player",
  "version": "1.0.0",
  "runtime": {
    "runnables": [
      {"id":"enable","kind":"enable","runtime":"ue4ss","entryFile":"ue4ss/main.lua","entrySymbol":"enable"},
      {"id":"disable","kind":"disable","runtime":"ue4ss","entryFile":"ue4ss/main.lua","entrySymbol":"disable"},
      {"id":"location","kind":"query","runtime":"ue4ss","entryFile":"ue4ss/main.lua","entrySymbol":"location"},
      {"id":"watch","kind":"service","runtime":"ue4ss","entryFile":"ue4ss/main.lua","entrySymbol":"watch"}
    ]
  }
}
```

The directory must be `unreal.player`. `entrySymbol` names the Lua function, including supported dotted table lookup. Entry source is loaded once per package lifetime. Query/action/service entries receive the parameter object; `disable` is invoked without parameters. Shared native parameter-schema checks apply before dispatch; schemas do not replace validation of game state inside the function.

Package-local `require("util.position")` resolves `ue4ss/util/position.lua` and caches it in that package. `require("UEHelpers")` uses the distribution's shared helper, which must be supplied. The UE VM follows package visibility rules for its own dependencies; host-Lua globals are not imported across the process boundary. Export declared runnables for cross-runtime access.

## Unreal operations and object lifetime

`FindFirstOf` selects a non-default instance; validate a returned object before using it. Reflected members can expose properties or functions, and reflected function calls use the object's methods. See the upstream [FindFirstOf contract](https://docs.ue4ss.com/lua-api/global-functions/findfirstof.html) and [UObject API](https://docs.ue4ss.com/lua-api/classes/uobject.html).

Resolve the intended controller/pawn again after respawn, save loading or world travel. A retained object wrapper does not establish that the underlying object is still valid. Return plain coordinates, IDs or other detached data rather than UObject/FVector wrappers. The [location sample](samples/ue4ss-player-location.md) resolves objects on every observation.

Entrypoints and `hex.every` callbacks are dispatched on the game thread. Keep them short. A query must read/compute its result before returning; scheduling a deferred callback inside it does not make that callback's result its return value. Actions that schedule later work acknowledge the entrypoint return, not completion of the deferred effect.

Hooks execute in their upstream callback context. The managed `ExecuteInGameThread(callback)` wrapper can marshal thread-sensitive work. It accepts a callback, associates it with package lifetime and suppresses it after release. Do not assume every hook already runs on the game thread. See upstream [game-thread scheduling](https://docs.ue4ss.com/lua-api/global-functions/executeingamethread.html).

## Managed package API

These helpers are available in UE4SS package environments:

| API | Contract |
| --- | --- |
| `hex.onCleanup(callback)` | Register a function for reverse-order cleanup; returns that function |
| `hex.every(intervalMs, callback)` | Schedule repeating game-thread work; interval at least 25 ms; at most one queued callback; returns a cancellation function |
| `hex.null` | Explicit JSON null, preserving object fields and array positions |
| `hex.array(table)` | Mark an array, including an empty array |
| `publishEvent(name, payload)` | Publish a detached service observation with provider package/service identity |
| `RegisterHook(path, preCallback, postCallback?)` | Register wrapped callbacks and retain both returned IDs for package cleanup |
| `UnregisterHook(path, preId, postId)` | Remove a hook registered by this package; an unowned tuple fails |
| `ExecuteInGameThread(callback)` | Schedule a package-owned wrapped callback |

The `RegisterHook` target must exist when registered, and upstream callback parameter/return rules still apply. The wrapper adds ownership and fault handling; it does not redefine the underlying [UE4SS hook API](https://docs.ue4ss.com/lua-api/global-functions/registerhook.html).

Raw `LoopAsync`, `ExecuteAsync`, `ExecuteWithDelay` and `NotifyOnNewObject` are blocked in managed package code. Use `hex.every` and explicit cleanup. Other upstream globals are not a complete managed resource set. This environment is not a sandbox for untrusted scripts, and ordinary host-Lua memory functions or `hex.query/action/startService` are not installed here.

## Services and cleanup

```lua
function watch(params)
    local interval = params.intervalMs or 250
    assert(type(interval) == "number" and interval >= 25, "Invalid interval")
    publishEvent("player_location", location())
    return hex.every(interval, function()
        publishEvent("player_location", location())
    end)
end
```

A service registers its work and returns. A returned function is recorded as cleanup. Startup success does not promise future observations. Do not write an infinite service loop or block while waiting for a host response.

Managed hooks, timers and cleanup callbacks belong to the provider package. Cleanup runs in reverse registration order and attempts remaining callbacks even if one fails. Explicit final-owner disable calls the disable entrypoint and then cleanup. Detach invokes resource cleanup without calling the disable entrypoint; put restorers needed for both cases in `hex.onCleanup`.

Arbitrary effects are not automatically reversible. An item grant, spawn or game save change may remain after disable. Property restorers must revalidate the relevant object before restoring an explicitly reversible change.

A service startup or wrapped callback error faults and cleans up the affected package. Native service state and hosted feed bookkeeping are reconciled; no automatic restart occurs. Release all retaining consumers/explicit owners and enable again. Connection loss requires reattachment. Starting an active service again fails with `service_already_running`; there is no independent public stop-service API.

## Results, transport and troubleshooting

Only finite numbers, strings, booleans, objects, arrays and null cross the bridge. Copy Unreal values to plain tables. Functions, UObject handles, cycles and unsupported userdata cannot cross. Ordinary `{}` is an empty object; use `hex.array({})` for an empty array. Nesting is bounded to 32 tables and messages to 1 MiB. See [dependency values and errors](dependency-runnable-api.md#values-and-errors).

The current local cross-process transport uses complete JSON request/reply/event files. Package authors do not read or write these files. Native then places events in an in-memory queue for the application. Observations are asynchronous and use the existing feed/ViewModel path.

| Symptom/code | Meaning / response |
| --- | --- |
| `ue4ss_unavailable` | Preparation, loading, readiness or connection failed; check actual framework root, assets, process access and framework log |
| `ue4ss_execution_failed` | Entry/callback failure or unsupported query value; inspect framework log, target IDs and result encoding |
| `ue4ss_completion_unconfirmed` | A request has no confirmed reply; it may have executed. No replay occurs. Reattach and inspect state before another explicit mutation |
| Hosted `ue4ss_disconnected` | Asynchronous connection-loss projection; reattach |
| Successful null location | No usable pawn/controller at that observation; menus/loading can legitimately have no player |
| Provider events absent on consumer feed | Events retain provider IDs; a wrapper does not remap them automatically |

The readiness deadline is 30 seconds; invocation acknowledgment is bounded to 10 seconds. These bounds do not guarantee game responsiveness. Original game-side exception text stays in the framework log; public errors expose a safe reason and target identity.

The checked-in integration fixtures validate loading, IPC, lifecycle, ownership and failure handling with mocked reflection. They do not prove live Palworld inventory, spawning, teleportation or active-world coordinates. Verify reflected targets, local-player selection, world transitions and reversible cleanup in the actual game. See [testing](testing.md#ue4ss-packages).

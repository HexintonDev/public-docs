# Lua Dependency Runnable API

Status: implemented host-Lua API, reviewed on 2026-10-07. UE4SS targets require a client build containing that runtime and the UE4SS framework installed with it; see [UE4SS availability](ue4ss-runtime.md#loading-and-prerequisites).

Ordinary engine Lua can call declared dependency runnables through the same native validation and dispatch used by UI commands. This permits host memory logic to use an Unreal observation without sharing interpreters or inventing a second lifecycle.

## Methods

```text
hex.query(packageId, runnableId, parameters?)       -- detached result
hex.action(packageId, runnableId, parameters?)      -- true after entry returns
hex.startService(packageId, runnableId, parameters?) -- true after startup returns
```

IDs must be non-empty strings naming the provider package and its exact manifest runnable. Each method requires the corresponding `query`, `action` or `service` kind. Parameters must be a JSON-compatible object and pass the target's existing `parameterSchema`; omission or `nil` means `{}`. A query returns its detached value. Action/service return `true`, not the target function's return value.

The target runtime must support the operation. Host Lua and UE4SS provide queries/services; an AA action can be invoked through `hex.action`, while AA queries/services are unsupported. Calls use the attached PID/session of the caller.

The shared schema validator supports object `required`/`properties`, array `items` and primitive types, including integer. It is not a complete JSON Schema implementation. Validate range/game-specific requirements in the entrypoint too. For example, declaring an integer `intervalMs` does not enforce the managed timer's minimum interval by itself.

## Declaration and authorization

```json
{
  "dependencies": [{"id":"unreal.player","compatible":"^1.0.0","tested":"1.0.0"}]
}
```

The provider must be directly declared by the caller and already enabled. A transitive-only dependency does not grant this API. The normal planner enables dependencies before entering consumer `enable`, so that function can call an admitted provider. Calls do not enable arbitrary packages, acquire another command owner or change reference counts. Global Lua chunks without package identity cannot use this capability.

The API resolves the actually executing package, including nested calls and coroutines. Copying another package's API table does not grant its authority. It is available in the host VM, not the game-side UE4SS environment or application JavaScript.

## Query and action

```lua
function readLocation()
    local value = hex.query("unreal.player", "location", {})
    if value == hex.null then return hex.null end
    return {x=value.x, y=value.y, z=value.z}
end

function capture()
    hex.action("unreal.player", "capture", {})
end
```

The provider must declare both named runnables; the action name above is illustrative. A successful query can return `false` or null. Test for the value's actual contract rather than interpreting any false-like value as a failure.

## Services and package ownership

```lua
function watch(params)
    hex.startService("unreal.player", "watch", {intervalMs=params.intervalMs or 250})
end
```

The result confirms startup only. Subsequent events/faults travel through native events and host feeds. The service belongs to `unreal.player`, and its events retain that package and runnable identity. Declaring a consumer wrapper service does not remap provider events to the consumer. Bind feeds to the provider's declaration, or explicitly publish consumer observations from consumer-owned work. There is no synchronous event-subscription API here.

An already-running provider service fails with `service_already_running`. There is no separate stop-service method. Disabling one consumer preserves a provider held by another consumer or explicit command owner. Final owner release disables/cleans the provider. Enable/disable remain native lifecycle commands; do not emulate them as dependency calls.

After a UE package callback fault, release all retaining owners and enable again. After connection loss/unconfirmed completion, reattach. The engine does not silently restart services or replay actions.

## Values and errors

Supported values are finite numbers, strings, booleans, objects, arrays and `hex.null`. Values are detached copies. Unreal userdata, Lua functions, process handles, cycles and mixed incompatible tables cannot cross. Strings preserve Unicode and embedded NUL bytes. JSON numbers do not provide arbitrary-precision integers.

`hex.null` preserves explicit null fields and array slots that assigning Lua `nil` would remove. `hex.array({})` is an empty array; ordinary `{}` is an empty object. Returned arrays keep their array identity. Nesting is bounded to 32 tables, parameter/message sizes to 1 MiB, and nested runnable calls to 32. These are bounds, not recommended payload sizes.

```lua
local ok, result = pcall(function()
    return hex.query("unreal.player", "location", {})
end)
if not ok then
    -- result.code, result.message, result.scriptId, result.runnableId
    -- result.domain == "process_engine"; result.severity == "error"
    error(result) -- keep structured target identity when forwarding the failure
end
```

Caught failures are tagged tables. Uncaught/rethrown tables retain the target package, runnable and native code rather than becoming an unrelated caller failure. Do not replace a structured error with a success-shaped empty value. Human-readable messages can change; branch on `code`.

| Code/family | Meaning |
| --- | --- |
| `dependency_access_denied` | Target is not a direct declared dependency |
| `missing_package` / `missing_engine_context` | Required active package/session is absent |
| `missing_runnable` / `invalid_runnable_type` | Missing ID or wrong kind |
| `invalid_parameters` | Invalid object/schema or size limit |
| `unsupported_runtime` | Runtime lacks the operation |
| `service_already_running` | Existing provider service key |
| `runtime_busy` | Background call could not acquire the native transaction immediately |
| `ue4ss_unavailable` | UE preparation/connection unavailable |
| `ue4ss_execution_failed` | UE entry/value/package fault |
| `ue4ss_completion_unconfirmed` | Execution may have happened; no confirmed completion, no replay |

These codes are not exhaustive. Preserve unfamiliar failures. Game-side source exceptions remain in the framework log; the public result retains the phase-safe reason and target identity.

## Foreground and timer execution

Calls inside a normal runnable borrow the existing native transaction and wait synchronously for the target. Same host-VM nested calls run inline on that VM's worker. Other work on the caller's worker waits until the call finishes.

Host-Lua timer calls use nonblocking admission to avoid deadlocking a command waiting for the Lua worker. Contention raises catchable `runtime_busy`; it does not queue the operation. A timer can skip a read and observe again on a later tick. Do not automatically repeat an action after uncertain completion.

```lua
local ok, value = pcall(function()
    return hex.query("unreal.player", "location", {})
end)
if not ok then
    if value.code == "runtime_busy" then return end
    error(value)
end
publishEvent("consumer_location", value)
```

This callback body assumes a declared provider and a consumer-owned timer/service. Destroy that host timer during consumer disable. Unhandled ordinary host-timer errors are logged and disable the timer; they are not returned as a foreground command result.

See the [complete location sample](samples/ue4ss-player-location.md), [thread model](thread-model.md), [package lifecycle](script-packages.md) and [errors/results](errors-and-results.md).

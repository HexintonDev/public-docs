# Runtime Model

Status: current public architecture reference. UE4SS is an implemented integration whose framework-delivery and qualification limits are documented separately.

## Runtime Families

The client has four distinct execution environments:

| Runtime | Executes | Typical responsibility |
| --- | --- | --- |
| Lua | Native engine worker | Process discovery, address resolution, memory I/O, scans, symbols, timers, services |
| UE4SS Lua | Attached Unreal game process | Reflected object/function access, managed hooks and game-thread observation |
| Auto Assembler | Native engine assembly runtime | Allocations, labels, scans, jumps, and assembly actions |
| Application JavaScript | Desktop host | Capability-gated host scripts, launch and file/window/process discovery |

Application JavaScript is not a substitute for Lua. It cannot directly call native memory APIs. A UI
must call a declared command, query, view model, or state feed exposed by the host.

## Choosing a Runtime

- Use Lua for game-facing logic and lifecycle-owned resources.
- Use UE4SS for Unreal object/function operations, with a prepared framework. Ordinary Lua can consume its declared runnables through the [dependency API](dependency-runnable-api.md).
- Use Auto Assembler for assembly operations declared in a package manifest; it supports enable/disable/actions, but not queries/services.
- Use application JavaScript for approved host-side operations. WebView surface JavaScript is a separate presentation environment using declared bindings.

A package may expose multiple runnable types, but each runnable declares its runtime explicitly.
Choose the smallest fitting combination from the source behavior; there is no requirement to
translate an assembly patch into Lua or express pointer/value logic in assembly. Read
[runtime selection](../../start-here/getting-started/runtimes.md) for concrete choices and contracts.

Native Lua and UE4SS use separate VMs; neither implicitly receives the other's globals or object handles. They share the native planner, runnable kinds and ownership rules. Cross-runtime calls exchange detached JSON-compatible values. See [UE4SS runtime](ue4ss-runtime.md) for entrypoints, services, cleanup and loading limits.

## Boundaries

The desktop application owns package discovery, session orchestration, and UI hosting. Hexinton
Engine owns process sessions, script contexts, runtime execution, symbols, timers, and cleanup. The
host passes structured command arguments into the engine and receives structured results, warnings,
and errors back. UE4SS execution crosses a local game-process bridge internally; that transport does not change the UI command contract. Queries confirm a returned observation, actions confirm entrypoint completion, and service startup is distinct from later feed delivery.

## Safe Rule

Never put credentials, unrestricted filesystem access, or raw WebView bridge calls into a trainer
surface. Keep native capabilities behind declared package bindings and validate all inputs at the
engine boundary.

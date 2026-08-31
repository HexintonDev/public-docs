# Runtime Model

Status: current public architecture reference for Hexinton Engine scripting.

## Runtime Families

Hexinton Engine has three distinct execution environments:

| Runtime | Executes | Typical responsibility |
| --- | --- | --- |
| Lua | Native engine worker | Process discovery, address resolution, memory I/O, scans, symbols, timers, services |
| Auto Assembler | Native engine assembly runtime | Allocations, labels, scans, jumps, and assembly actions |
| Application JavaScript | Desktop host | Trainer UI logic, hosted surfaces, and capability-gated application behavior |

Application JavaScript is not a substitute for Lua. It cannot directly call native memory APIs. A UI
must call a declared command, query, view model, or state feed exposed by the host.

## Choosing a Runtime

- Use Lua for game-facing logic and lifecycle-owned resources.
- Use Auto Assembler for assembly actions declared in a package manifest.
- Use application JavaScript only for presentation and host-authorized UI behavior.

A package may expose multiple runnable types, but each runnable declares its runtime explicitly.

## Boundaries

The desktop application owns package discovery, session orchestration, and UI hosting. Hexinton
Engine owns process sessions, script contexts, runtime execution, symbols, timers, and cleanup. The
host passes structured command arguments into the engine and receives structured results, warnings,
and errors back.

## Safe Rule

Never put credentials, unrestricted filesystem access, or raw WebView bridge calls into a trainer
surface. Keep native capabilities behind declared package bindings and validate all inputs at the
engine boundary.

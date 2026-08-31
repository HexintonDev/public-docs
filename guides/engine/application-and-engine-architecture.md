# Application and Engine Architecture

Status: current public architecture reference.

## Two-Layer System

The installed client has an application layer and an engine layer. They cooperate through a
structured host boundary rather than sharing unrestricted implementation details.

| Layer | Owns | Does not own |
| --- | --- | --- |
| Application | Game catalog, installation profiles, launch flow, session UI, package editing, hosted trainer surfaces | Direct native memory operations |
| Hexinton Engine | Process attachment, Lua and Auto Assembler execution, address resolution, memory operations, symbols, timers, cleanup | Product navigation and visual layout |

## Session Flow

```text
User selects installation
  -> Application launches or attaches to target
  -> Engine opens a process session
  -> Application loads package metadata
  -> Engine validates and runs a package command
  -> Engine returns a result or error
  -> Application updates controls and feeds
```

A command result confirms completion of that command. It does not replace the authoritative state
published by a service-backed view model.

## Package Boundary

The package manifest is the contract between application and engine. It declares runtime files,
runnables, dependencies, parameters, and frontend bindings. The application uses declarations to
render controls; the engine uses them to validate and execute work.

## Failure Boundary

The engine must reject unresolved addresses, missing scan results, invalid arguments, stale process
sessions, unsupported runtimes, and failed assembly. The application should display the structured
failure and keep the control state consistent with the last authoritative feed.

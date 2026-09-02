# Application and Engine Architecture

Status: current public architecture reference.

## Three Layers

The installed client has three cooperating layers. Each layer has a different authority and
communication boundary.

| Layer | Owns | Does not own |
| --- | --- | --- |
| UI / WebView / React | Input drafts, rendering, focus, pending command presentation, surface interaction | Native memory, authoritative trainer values, session transitions |
| Desktop application / C# | Game/session state, package sync, command routing, ViewModels, projections, feeds, WebView transport | Direct native memory operations and Lua state |
| Hexinton Engine | Process sessions, Lua and Auto Assembler execution, addresses, memory, symbols, timers, cleanup | Product navigation, React state, visual layout |

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

A command result confirms completion of one requested operation. It is not the Service, Feed,
ViewModel, or current target value. A Service observes changing values; the host Feed routes those
observations; the host ViewModel stores the current projected value; and the UI renders that
ViewModel. A state delta is only the transport message describing a ViewModel mutation.

## Complete Command Flow

For a hosted trainer action, the authoritative order is:

```text
UI event
  -> commandId created by application command transport
  -> command router validates the message and publishes running
  -> hosted command service validates widget input and binding
  -> GameSession enqueues the invocation on its serialized worker
  -> transaction executor prepares parameters and value/delta resolution
  -> session gateway validates Attached state, pid, session token, package, and runnable
  -> runtime worker selects Lua or Auto Assembler
  -> native host executes the runnable under the pid context lock
  -> engine returns success JSON or structured error
  -> transaction executor returns typed outcome
  -> GameSession commits the trainer projection when policy allows
  -> session projector updates the application ViewModel
  -> application state channel assigns the next stateVersion
  -> WebView writer sends the contiguous delta
  -> WebView writer sends the matching checkpoint
  -> UI applies the delta and renders the committed value
  -> command router publishes succeeded or failed for the commandId
```

The command-operation projection and session trainer projection are separate:

| Projection | Meaning |
| --- | --- |
| `/commandOperations/{commandId}` | One command's running, succeeded, or failed lifecycle |
| `/sessions/{gameId}/trainerView` | The authoritative committed trainer value |

A successful native result does not automatically mean that a changing game value has been
observed. Service-backed values use the Service -> Feed -> ViewModel path.

## Package Boundary

The package manifest is the contract between application and engine. It declares runtime files,
runnables, dependencies, parameters, and frontend bindings. The application uses declarations to
render controls; the engine uses them to validate and execute work.

## Failure Boundary

The engine must reject unresolved addresses, missing scan results, invalid arguments, stale process
sessions, unsupported runtimes, and failed assembly. The application should display the structured
failure and keep the control state consistent with the last Feed value accepted by the host.

## Service -> Feed -> ViewModel Flow

```text
Lua Service publishes an initial observation and starts its timer
  -> native event queue stores package-scoped event and sequence
  -> host event pump drains events after the process wait handle signals
  -> event is checked against current pid and session token
  -> event is enqueued on the owning GameSession worker
  -> stale/out-of-scope event is rejected
  -> matching Feed stores the accepted observation
  -> immutable TrainerView/ViewModel is updated
  -> projector emits one application-state delta
  -> WebView receives delta, then matching checkpoint
```

## Boundary Rules

- UI code uses typed application commands and projected ViewModels.
- The desktop application validates package declarations and session admission before invoking the
  engine.
- The engine validates runtime, process, address, memory, assembly, and cleanup behavior.
- A command result is correlated by `commandId`; a state update is ordered by `stateVersion`.
- A WebView checkpoint acknowledges delivery order; it is not a second state authority.
- A service event must match the current attachment generation before it can update the UI.

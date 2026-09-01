# Thread Model

Status: current public runtime behavior reference.

## Thread, Lane, and Lock

A thread is an actual execution thread. A lane is a serialized execution path and is not
necessarily tied to one OS thread. A lock serializes shared state but does not choose where work
runs.

The application and engine use all three concepts:

```text
WPF/WebView UI thread
	-> GameSession async lane per gameId
	-> ScriptRuntimeWorker for the selected runtime
	-> native command bridge
	-> EngineContextState lock per pid
	-> Lua worker per pid, or synchronous Auto Assembler path
```

`gameId` identifies an application session. `pid` identifies the currently attached process. A
JavaScript finder may run before a pid exists; Lua and Auto Assembler attached work require one.

## Application Session Lane

Each `GameSession` owns a single-reader work queue. It processes state transitions, launch results,
detection observations, package reloads, trainer commands, and shutdown one item at a time. The lane
is async/await based and is not guaranteed to remain on one OS thread.

Keep polling, CPU-heavy work, blocking filesystem access, and infinite loops out of this lane. A
long awaited operation delays later commands and state transitions for that game.

## Engine Worker

Lua execution and Lua C API calls are confined to the engine's Lua worker thread. Native host code
must not call Lua directly from an arbitrary application or timer thread. Requests are serialized
through the runtime command path.

At the native boundary, commands for one pid are also serialized by that process context's
`executionMutex`. Commands for different pids may proceed after runtime admission, subject to their
runtime worker queues.

## Managed Runtime Workers

The desktop host schedules script execution through a process-lifetime `ScriptExecutor`. The current
model has one dedicated managed worker per runtime type (`js`, `lua`, and `aa`). This keeps script
execution off the WPF dispatcher, but it does not make every command independently concurrent.

The managed Lua and AA adapters are not the same as the native Lua worker. They admit work to the
native boundary; native Lua then queues the operation onto its per-pid Lua thread.

## Lua Runtime

The host creates a native `LuaRuntime` lazily for each pid that uses Lua. Its persistent Lua state,
script environments, module caches, timers, and callbacks are owned by that runtime and accessed on
its dedicated worker. A caller submits work and waits for its result; arbitrary callers must not touch
the `lua_State` directly.

Lua therefore has two serialization points: the native per-pid context lock and the Lua runtime
worker queue. Timer callbacks run on the Lua worker, not on the UI thread.

## Auto Assembler Runtime

Auto Assembler currently has no dedicated native runtime worker. The managed AA worker invokes the
assembly file synchronously while the native pid context lock is held:

```text
AA managed worker
	-> native command
	-> lock pid executionMutex
	-> read and execute AssemblyScript synchronously
	-> unlock pid executionMutex
```

An AA action therefore occupies its managed AA worker and its target pid lock until it returns. It
must not contain an unbounded wait or an infinite loop. `createThread(address)` starts target code
from the assembly directive, but the package still owns the code and its cleanup responsibility.

## Application JavaScript and Jint

Application JavaScript runs through the managed script executor and a Jint runtime. A normal JS
execution creates a fresh isolated Jint Engine, host API instance, and module execution scope. It
does not share JS globals with another execution and does not expose CLR access.

Long-running JS tasks use one persistent Jint Engine for that task and must not be accessed
concurrently. Jint execution is bounded by timeout, cancellation, memory, and statement limits.
Finder scripts are one-shot application-JavaScript calls made on each detection heartbeat; they do
not own timers or session lifetime.

## Timers

A timer callback is scheduled by the runtime but its Lua work is serialized with script execution.
Treat timer callbacks as engine work: keep them short, avoid blocking waits, and stop them during
package cleanup.

## Application UI

The desktop application and WebView UI have their own UI/event threads. UI code should issue a
host command or consume a view model/feed; it should not assume that a command result and the next
feed update occur in the same turn.

## Runtime Blocking Scope

An ordinary synchronous action should finish promptly. If it blocks, it can hold the originating
GameSession lane, the selected managed runtime worker, the native pid lock, and for Lua the native
Lua execution path. Model long-running observation as a service that starts quickly and publishes
later events. Do not implement a service as an infinite action loop.

## Ordering Rules

1. Validate a command before starting native work.
2. Complete one script operation before another operation mutates the same script context.
3. Publish an initial service value before enabling periodic observation.
4. Stop timers before releasing their script state.
5. Disable package-owned hooks, symbols, and allocations before destroying the session.
6. Re-enter the owning GameSession lane before applying detection or service results to projected
	state.
7. Stop event pumps and timers before releasing the Lua state or native process context.

## Practical Consequence

Do not use shared mutable state as an implicit cross-thread protocol. Use command results for one
operation and revisioned state feeds for ongoing state. This prevents stale UI values from being
mistaken for completed engine work.

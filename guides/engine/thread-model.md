# Thread Model

Status: current public runtime behavior reference.

## Engine Worker

Lua execution and Lua C API calls are confined to the engine's Lua worker thread. Native host code
must not call Lua directly from an arbitrary application or timer thread. Requests are serialized
through the runtime command path.

## Timers

A timer callback is scheduled by the runtime but its Lua work is serialized with script execution.
Treat timer callbacks as engine work: keep them short, avoid blocking waits, and stop them during
package cleanup.

## Application UI

The desktop application and WebView UI have their own UI/event threads. UI code should issue a
host command or consume a view model/feed; it should not assume that a command result and the next
feed update occur in the same turn.

## Ordering Rules

1. Validate a command before starting native work.
2. Complete one script operation before another operation mutates the same script context.
3. Publish an initial service value before enabling periodic observation.
4. Stop timers before releasing their script state.
5. Disable package-owned hooks, symbols, and allocations before destroying the session.

## Practical Consequence

Do not use shared mutable state as an implicit cross-thread protocol. Use command results for one
operation and revisioned state feeds for ongoing state. This prevents stale UI values from being
mistaken for completed engine work.

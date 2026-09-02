# Hexinton Engine Sample Library

Status: current public sample index.

This library contains small, copyable examples for one Hexinton Engine behavior at a time. Each
sample states its prerequisites, source layout, expected result, and cleanup behavior. Game names,
addresses, and AOB patterns are illustrative unless a sample explicitly says otherwise.

## Samples

| Sample ID | Topic | What it demonstrates |
| --- | --- | --- |
| [`package/enable-action-disable`](package-enable-action-disable.md) | Package lifecycle | Dependency-first enable, Lua action parameters, cleanup, symbols, and an allocation |
| [`lua/multifile-require`](lua-multifile-require.md) | Lua modules | Package-local `require`, module caching, and dependency-visible globals |
| [`process/explicit-handles`](process-explicit-handles.md) | Process handles | Default versus explicit processes, typed I/O, process discovery, and windows |
| [`service/timer-feed`](service-timer-feed.md) | Services | Initial event, recurring timer, parameter schema, and cleanup |
| [`aa/allocation-action-pair`](aa-allocation-action-pair.md) | Auto Assembler | Separate allocation/release AA actions for a named cave |

## Using a Sample

1. Read the sample page and its assumptions.
2. Copy the structure, not the game-specific values.
3. Replace addresses, process names, and symbols with values verified for your target.
4. Add a cleanup path before enabling a write, patch, hook, service, or timer.
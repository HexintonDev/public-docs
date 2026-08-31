# ProcessEngine Sample Library

Status: current public sample index.

This library contains small, test-backed examples for one ProcessEngine behavior at a time. It is
not a collection of unverified trainer snippets. Each sample links to the exact fixture and test
used as evidence.

## Pinned Source Version

All links in this first edition are pinned to HexModClient commit
[`3307591`](https://github.com/HexintonDev/HexModClient/tree/3307591b19f2f73e543636e77de0d77b959a3fd2).

## Samples

| Sample ID | Topic | What it demonstrates |
| --- | --- | --- |
| [`package/enable-action-disable`](package-enable-action-disable.md) | Package lifecycle | Dependency-first enable, Lua action parameters, cleanup, symbols, and an allocation |
| [`lua/multifile-require`](lua-multifile-require.md) | Lua modules | Package-local `require`, module caching, and dependency-visible globals |
| [`process/explicit-handles`](process-explicit-handles.md) | Process handles | Default versus explicit processes, typed I/O, process discovery, and windows |
| [`service/timer-feed`](service-timer-feed.md) | Services | Initial event, recurring timer, parameter schema, and cleanup |
| [`aa/allocation-action-pair`](aa-allocation-action-pair.md) | Auto Assembler | Separate enable/disable AA actions that allocate and release a named cave |

## Using a Sample

1. Read the sample page and its assumptions.
2. Open the pinned fixture link.
3. Open the pinned test link and inspect the asserted behavior.
4. Copy the structure, not the game-specific values.
5. Replace fixture addresses, process names, and symbols with values verified for your target.
6. Add a cleanup path before enabling a write, patch, hook, service, or timer.

## Link Policy

These are commit-pinned source links. New wiki releases should create a new sample-index revision
with links pinned to the release commit. Do not silently change an existing sample's evidence.
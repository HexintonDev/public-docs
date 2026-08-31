# Hexinton Engine Sample Library

Status: current public sample index.

This library contains small, test-backed examples for one Hexinton Engine behavior at a time. It is
not a collection of unverified trainer snippets. Each sample links to the exact fixture and test
used as evidence.

## Pinned Source Version

All samples in this first edition were checked against HexModClient commit `3307591`.
Repository-relative evidence paths are recorded on each sample page without external links.

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
2. Review the evidence paths and asserted behavior described on the page.
3. Copy the structure, not the game-specific values.
4. Replace fixture addresses, process names, and symbols with values verified for your target.
5. Add a cleanup path before enabling a write, patch, hook, service, or timer.

## Link Policy

These are commit-pinned evidence records. New wiki releases should create a new sample-index
revision with the release commit and repository-relative evidence paths. Do not silently change an
existing sample's evidence.
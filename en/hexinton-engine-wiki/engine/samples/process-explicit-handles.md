# Sample: Explicit Process Handles

Sample ID: `process/explicit-handles`

This sample shows how a command's default process differs from processes opened explicitly by ID.

## Behavior

Bare `writeInteger` uses the command/default process. `openProcess(pid)` returns a handle whose
methods read and write that explicit process. The sample also covers `findProcess`, `findProcesses`,
`findWindow`, `getWindowProcessID`, and `findWindowProcessID`.

## Reuse Rules

- Use bare functions only when the command's attached target is intentional.
- Use `local target = openProcess(pid)` when operating on another process.
- Prefer `target:method(...)`; the tested dot-call form is also supported.
- Validate `pid` before calling `openProcess`; `0` is an invalid process ID.

## Expected Result

Writes to the default, secondary, tertiary, and window-owned processes remain isolated. Symbols
published through a handle belong to that handle's script context, not to unrelated scripts.
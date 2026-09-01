# Troubleshooting

Status: current author and user troubleshooting guide.

## Package Does Not Validate

Check that the directory name equals manifest `id`, JSON is strict, every referenced file exists,
runnable IDs are unique, and `enable`/`disable` are present. Read the structured error `code`; do not
branch on message wording.

## Process or Address Is Missing

Confirm the target is running and use process discovery before attachment. Use `getAddressSafe` for
optional expressions. For AOB scans, require the expected match count and reject zero or ambiguous
results. Confirm module name, architecture, and build before writing.

## Enable Succeeds but Feature Does Not Work

Check the command result, then check the authoritative service/feed state. A successful command does
not prove that a service published its first value. Inspect warnings, target assumptions, and whether
the action was run against the intended pid.

## Disable or Detach Hangs

Stop package timers and hosted services first. Drain or cancel event-pump work, then destroy the
native process context. A Lua callback must not wait on work that requires the same Lua worker. Add
paired start/completion diagnostics around detection, session teardown, event-pump reset, context
destruction, and runtime destruction.

## Patch or Hook Failure

Use an expected-byte check, verify AOB uniqueness, preserve displaced instructions, and keep the
original bytes available for restoration. Allocation alone is not a hook. After a partial enable,
run cleanup and confirm symbols, allocations, patches, and timers are gone.

## What to Collect

Record package ID/version, runnable ID, target process/build, structured error code, warning code,
last accepted feed revision, and the operation that preceded the failure. Remove credentials and
sensitive memory values before sharing diagnostics.

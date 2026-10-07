# Errors and Results

Status: current public engine host contract.

This page describes the structured boundary returned by Hexinton Engine commands. Package authors
should branch on machine-readable codes, not human-readable message text.

## Success Result

Every successful command has `ok: true`, a stable `status`, the command `pid`, package `scriptId`,
`runnableType`, and `runnableId`.

```json
{
  "ok": true,
  "status": "action_executed",
  "pid": 1234,
  "scriptId": "trainer.player",
  "runnableType": "action",
  "runnableId": "give-runes"
}
```

Current statuses are `enabled`, `disabled`, `action_executed`, `query_executed`, and `service_started`. Query results
are returned in `result`; enable/action commands may return `executedScripts`; disable commands may
return `disabledScripts` and `wasEnabled`.

## Query Result

```json
{
  "ok": true,
  "status": "query_executed",
  "pid": 1234,
  "scriptId": "trainer.player",
  "runnableType": "query",
  "runnableId": "read-hp",
  "result": { "hp": 120, "maxHp": 180 }
}
```

A command result confirms one operation. It is not a replacement for a service-backed state feed.

## Warnings

Warnings are non-fatal. A command can succeed while returning a `warnings` array. Warning objects
have `severity: "warning"`, a stable `code`, a human-readable `message`, and optional package and
source-location fields. The current dependency warning is `dependency_untested_version`.

Applications should display or log warnings without treating them as command failure.

## Failure Result

A failed native host call returns no success JSON. The host exposes a structured last-error value:

```json
{
  "ok": false,
  "error": {
    "severity": "error",
    "code": "invalid_parameters",
    "message": "Parameters are invalid",
    "formatted": "[invalid_parameters] Parameters are invalid",
    "scriptId": "trainer.player",
    "runnableId": "give-runes"
  }
}
```

`code` is stable; `message` and `formatted` are for people and logs. Branch on `error.code`.
Optional fields include `scriptId`, `runnableId`, `file`, `line`, and `column`. Lua tracebacks may
currently be included in `message`.

Current error families include invalid commands and parameters, missing engine context, JSON parse
failures, invalid or missing manifests, missing lifecycle/runnables, invalid runnable types,
missing files, dependency failures, runtime failures, and assembly failures. New codes may be added;
clients must preserve unknown errors instead of assuming the list is exhaustive.

Host-Lua dependency calls raise a [structured Lua error table](dependency-runnable-api.md#values-and-errors), retaining target identity through uncaught/rethrown errors. UE4SS codes include `ue4ss_unavailable`, `ue4ss_execution_failed` and `ue4ss_completion_unconfirmed`. Unconfirmed completion may already have changed the game; do not replay the operation automatically. Game-side exception text remains in the framework log, while the public error gives a safe reason and target IDs. Asynchronous connection-loss delivery uses hosted `ue4ss_disconnected`.

## Event Polling

Engine events are ordered and drained separately from command results. A successful poll has
`ok: true`, `count`, `remaining`, and `events`. Each event contains `eventType`, `pid`, and a
monotonic `sequence`. A pid of `0` selects all process contexts; `max_events` of `0` drains all
matching events. Queued events survive context destruction until drained.

Current event types include `context_created`, `context_destroyed`, `diagnostic`, `service_event`, and UE `runtime_failure`. Service startup success is not a later event or a health guarantee. See [UE4SS service failure and recovery](ue4ss-runtime.md#services-and-cleanup).

## Cleanup After Failure

An enable operation can fail after enabling a dependency. The engine rolls back newly enabled
scripts. Package authors must still make disable actions idempotent and release package-owned hooks,
allocations, symbols, timers, and service state.

## Application Handling

The application should preserve correlation information, show a useful user-facing message, and
keep authoritative UI state sourced from the latest state projection. Never parse `message` to infer
an error category, and never treat a successful command as proof that a long-running service has
published its first state.

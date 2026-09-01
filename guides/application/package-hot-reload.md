# Package Hot Reload

Status: current desktop package-apply behavior.

Saving a local file does not activate it. `sessions.reloadPackages` prepares and applies a complete
resolved package graph as one activation generation.

## Transaction

```text
prepare -> compile preview -> plan -> disable affected packages
  -> promote staged packages -> assign graph and trainer view
  -> restore compatible prior enable intent -> publish one state revision
```

Preparation validates downloaded or local packages in staging and compiles the preview view before
native resources change. The planner detects added, removed, changed, and transitively affected
packages, including same-version content changes.

Only affected enabled packages are disabled. A failed disable aborts the reload and compensation is
attempted. After successful promotion, only packages that existed before and remain compatible are
re-enabled. New or renamed packages are never enabled automatically.

## Command

```json
{
  "command": "sessions.reloadPackages",
  "arguments": { "gameId": "demo.game.alpha" }
}
```

A successful receipt reports `committed`, changed/added/removed package IDs, and any
`reenableFailures`. A re-enable failure means the new graph is committed but that package remains
disabled; the result must not claim otherwise.

## Invariants

- A session never exposes a mixture of the old package graph and new trainer view.
- File watching may request apply but must not mutate native state directly.
- Native callbacks re-enter the session worker before changing projected state.
- Staged promotion rolls back already-promoted directories when a later move fails.
- Cancellation after a native lifecycle call is a partial-operation case; it cannot prove that the
  native side effect did not happen.

Changed hosted surfaces, hotkeys, and service feeds must reconcile against the committed generation
and must not register a second copy while the old generation is active.

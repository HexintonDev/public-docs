# Package Hot Reload

Status: desktop Studio's manual-Apply implementation, updated 2026-10-02. Older client builds may
have different save/reload behavior.

Saving a local file does not activate it. `sessions.reloadPackages` prepares and applies a complete
resolved package graph as one activation generation.

## Save, review, Apply, and execution

| Operation | Effect |
| --- | --- |
| Save a package file | Updates editable files and marks the game as pending Apply. Running packages keep using their applied files. |
| Keep an AI edit | Accepts the editor review decision. It does not Apply or enable the package. |
| Apply package changes | Captures and applies the complete game's resolved package graph, including saved local changes. |
| Enable or run an action/query | Executes against the applied generation, even if newer saved edits are pending. |

Saving does not disable or re-enable anything. Applied runtime files are separate from editable
files, including files read lazily on the first invocation. Editing `package.json`, Lua, Auto
Assembler, JavaScript, or custom surface files therefore does not activate those edits by saving.
Unsaved editor drafts must be saved before Apply can include them.

Apply covers the whole game graph, not just the selected package or conversation. New packages can
appear in Studio's Files tree before they have been applied. They have no applied runnable catalog
until Apply, and Apply does not automatically enable new or renamed packages. The trainer uses the
committed view; Studio's authoring preview may reflect the editable package before Apply.

Applied copies belong to the current session. A new session initially captures the current workspace;
pending drafts are not preserved as a separate inactive generation across session restart.

## Transaction

```text
prepare -> capture files -> static validation -> compile preview -> plan -> disable affected packages
  -> promote package/applied files -> assign graph and trainer view
  -> restore compatible prior enable intent -> publish one state revision
```

Preparation validates downloaded or local packages in staging and compiles the preview view before
native resources change. The planner detects added, removed, changed, and transitively affected
packages, including same-version content changes.

Only affected enabled packages are disabled. A failed disable aborts the reload and compensation is
attempted. After successful promotion, only packages that existed before and remain compatible are
re-enabled. New or renamed packages are never enabled automatically.

For an enabled package, the previous generation's disable function runs before files are promoted;
the new generation's enable function runs after commit. Dependency changes can restart affected
enabled dependents. Unaffected packages are not deliberately disabled/re-enabled.

Apply checks the exact captured candidate with the shared manifest, entry, hosted binding, native
dependency, Lua and JavaScript parse checks before disabling anything. A static failure rejects the
candidate and keeps the previous applied generation. Checks that require execution, including AA
assembly against a target, remain unrun. Action-only packages can be applied without a hosted widget.

This is not a behavior test. An enable function can parse successfully and fail during execution
after commit, leaving the package disabled with a detailed re-enable error.

## Command

```json
{
  "command": "sessions.reloadPackages",
  "arguments": { "gameId": "demo.game.alpha" }
}
```

A receipt reports `committed`, changed/added/removed package IDs, `reenableFailures`, and
`reenableErrors` keyed by package ID. Each detailed error retains its original code, message, and
diagnostic details. A re-enable failure means the new graph is committed but that package remains
disabled. Fix the reported cause, save and Apply the correction, then enable explicitly if needed.

Preparation or preview-validation failures keep the previous applied generation. Applied-file
promotion/compilation failure restores previous applied files before lifecycle compensation is
attempted. Compensation itself can fail; do not infer successful restoration from an error alone.

The Studio assistant uses the same session operations through `get_package_commands` and
`execute_package_command`. Its Apply result may have `ok: false` and `value.committed: true` when
re-enable fails. Inspect the committed flag and detailed errors before retrying. Cancellation after
native execution starts cannot establish that all side effects were undone.

See [Studio AI Assistant](../../start-here/getting-started/studio-ai-assistant.md) for tool parameters,
explicit execution requests, shared Problems reports, and the distinction between checks and live execution.

## Invariants

- A session never exposes a mixture of the old package graph and new trainer view.
- File watching marks pending Apply. It never applies packages or mutates native state.
- Native callbacks re-enter the session worker before changing projected state.
- Staged promotion rolls back already-promoted directories when a later move fails.
- Cancellation after a native lifecycle call is a partial-operation case; it cannot prove that the
  native side effect did not happen.

Changed hosted surfaces, hotkeys, and service feeds must reconcile against the committed generation
and must not register a second copy while the old generation is active.

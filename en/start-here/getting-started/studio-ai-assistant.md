# Studio AI Assistant

Status: current manual-Apply Studio assistant workflow, updated 2026-10-02. Availability depends on
the installed client build. Dedicated package validation and unit/integration runners are still being
expanded; the tools below describe what exists today.

## Game and package scope

Open Studio for a game to work with that game's packages. Its conversation list and new conversations
already belong to that game. Several conversations can run concurrently on independent packages.
If another conversation owns the same editing target, the assistant can wait or create a separate copy.

Synced packages are readable references. Editing one creates an independent local copy. A copy does
not replace or disable the source package, and does not rewrite other packages' dependencies.
Local packages and local copies are writable after the assistant prepares its editing target.

## Editing workflow

1. Describe the change and relevant package. The assistant discovers the actual installed packages
   and reads manifests and references before choosing APIs.
2. It prepares a writable target or creates a local package, then edits files with its coding tools.
3. Saved edits refresh the Files tree and mark the game pending Apply. Open a changed file from the
   conversation to review it in the code editor. Keep/Undo are review actions, not runtime activation.
4. The assistant runs available checks and reports their actual scope and results. Ordinary editing
   requests leave the changes pending Apply.
5. Select **Apply package changes**, or explicitly ask the assistant to Apply. Apply updates the
   complete game's graph, including other saved package changes.
6. Enable the feature or run its action/query using the trainer, a binding, or an explicit assistant
   execution request. Execution uses the applied generation.

Saving never restarts packages. Apply may disable affected enabled packages using their old code and
restore them using the new code. New packages remain disabled until an explicit enable or invocation.
Actions and queries can enable their owning package before running, so a query is live execution too.

For example:

- **Edit only:** "Add a health query to my local package and run available checks. Leave it pending Apply."
- **Apply:** "Apply this game's saved package changes."
- **Run:** "Inspect example.health's applied commands, then run read-health on the attached target."

If you ask to execute newly saved code, that code must be applied first. If the requested version or
whole-game Apply scope is unclear, the assistant should clarify it rather than silently choosing.
An execution request does not authorize uploading or publishing the package.

## Package tools

These are assistant tool names, not PowerShell commands or Lua host APIs.

| Tool | Behavior |
| --- | --- |
| `list_game_packages` | Lists this game's packages, files, ownership, and writability. |
| `prepare_package_edit(packageId, createSeparateCopy=false, waitForEditor=false)` | Returns a writable ID/directory; copies a synced source and coordinates agent editors. |
| `create_local_package(packageId, displayName)` | Creates and prepares a local package with an initial manifest. |
| `validate_package(packageId)` | Checks manifest identity/version presence and referenced runtime entry files. It does not run the package. |
| `get_package_commands(packageId)` | Returns applied runnable IDs, kinds, parameter schemas, applied/enabled state, attachment, pending Apply, and the last runtime error. |
| `execute_package_command(operation, packageId?, runnableId?, argumentsJson?)` | Executes `apply`, `enable`, `disable`, `action`, or `query` through the current game's session. |

`get_package_commands` reads applied manifests, so it can show the old runnable set while edits are
pending. A newly created package has `applied: false` and no applied runnables until Apply.
`attachedPid` and game status describe the current attachment; target-dependent execution needs a
valid attachment. Runnable IDs and parameter names come from the actual applied manifest.

For `apply`, the package and runnable IDs are omitted: this is a whole-game operation. Enable/disable
require a package ID. Action/query also require the manifest's runnable ID. `argumentsJson`, when
supplied, is a string containing a JSON object, not an array. For example, a parameterless query call is:

```json
{
  "operation": "query",
  "packageId": "example.health",
  "runnableId": "read-health",
  "argumentsJson": "{}"
}
```

This example requires an installed, applied package with that declared query. It is not a built-in
health API. The conversation supplies the game ID; these tools cannot choose a different game.

## Checks and error feedback

The assistant should distinguish these results:

- identity/entry-file checks from `validate_package`;
- syntax/schema checks from an actual compiler or validator;
- unit tests of isolated logic;
- integration tests against a controlled target;
- execution against the actual game.

Only report a check as passed when that corresponding check ran. Apply and a successful preview do
not prove game behavior. The dedicated package-facing syntax and unit/integration runner is not yet
exposed. Existing package test scripts can run in a terminal when available; unrun tests must be
reported as unrun. Loading a runtime script is execution, not a side-effect-free syntax check.

Execution returns `ok`, the result value, diagnostics, and an error when present. Users and the
assistant receive the original runtime cause, including a location when supplied by the runtime.
Failures can occur after some work has completed. In particular, this receipt fragment means Apply
committed the new generation but a package failed to re-enable:

```json
{
  "ok": false,
  "value": {
    "committed": true,
    "reenableFailures": ["example.health"]
  }
}
```

Read the accompanying `reenableErrors`, fix the cause, Apply the correction, and explicitly enable
if needed. Repeating the same command is not a substitute for reading its failure. The previous
failed attempt remains useful evidence; Stop does not prove native side effects were undone.

## References and terminals

The assistant can search the public documentation through GitBook MCP and read full API pages.
Bundled package/runtime/interface/debugging skills provide relevant local references when remote
documentation is unavailable. Local references are a revisioned client snapshot and can be older
than the published site. Documentation does not establish a game's addresses or verify a mod.

The Terminal panel supports multiple persistent shells. Agent terminal tools use the same service;
each conversation has its own default shell, and an explicit terminal ID can select an existing one.
If a command is still running, read its existing execution rather than start it again. Terminal
instances last for the application lifetime. Applied package copies last for the game session; a
new session initially captures the then-current workspace, including saved edits.

See [Package Hot Reload](../../hexinton-engine-wiki/application/package-hot-reload.md),
[Package Format](package-format.md), and [Testing](../../hexinton-engine-wiki/engine/testing.md).

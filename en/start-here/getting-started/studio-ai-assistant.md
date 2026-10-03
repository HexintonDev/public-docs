# Studio AI Assistant

For optional live-memory and static-analysis setup, see [Studio Reverse-engineering Tools](studio-reverse-engineering-tools.md).

Status: current manual-Apply Studio assistant workflow, updated 2026-10-02. Availability depends on
the installed client build. Shared static validation and diagnostic navigation are available in this
build; package unit/integration runs are available through the Tests tab and assistant tools.

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
4. The assistant is instructed to always write/update meaningful unit **and** integration tests for
   every package creation or change, including UI and metadata changes, then run static validation
   and both suites. It reports actual results and blocked coverage. Ordinary editing leaves pending Apply.
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
| `validate_package(packageId)` | Captures saved files and checks manifest identity/version presence, descriptors/entries, hosted bindings, the native dependency graph, and Lua/JavaScript syntax. It does not Apply or run code. |
| `get_package_diagnostics()` | Reads this game's current validation reports and recorded execution failures, including source/revision and original causes. |
| `discover_package_tests(packageId)` | Reads the package's `tests/studio.tests.json` declaration without execution. |
| `run_package_tests(packageId, kind="all")` | Runs captured saved files through fresh Lua/JavaScript workers or declared framework commands. Integration cases receive an owned memory target. No Apply or live-game execution. |
| `get_package_test_results()` | Reads this game's bounded app-lifetime test history, including detailed errors, locations, output and captured revision. |
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

Open **Problems** beside Widget Preview and Terminal, then select **Validate game** to check saved
packages without activating them. Successful user saves, including Studio's autosave and explicit
Overwrite, also validate that saved package after a short debounce. Editor markers and the Problems
count update without switching the selected output tab. This never applies or executes a package.
Failed saves retain the buffer and show their save error; validation does not run for unsaved content.
Independent failures are reported together: a broken manifest does not suppress Lua or JavaScript
syntax checks.

AI file writes and external filesystem changes do not trigger this editor-save check. The assistant
continues to call `validate_package` explicitly and retrieve diagnostics through its normal tools.
Repeated user saves are coalesced, and a save during an active check schedules a check of the newer
revision. If a validation command cannot finish, Studio distinguishes that failure from saving the file.

Each report identifies the package, saved/applied source, content revision, operation and check list.
Checks are `passed`, `failed`, or `not_run`. Lua uses the native Lua parser without calling the
chunk; `.js`/`.mjs` files use the same parser library as the application JavaScript runtime, without
importing or executing modules. Dependency checks use the native manifest planner. Hosted binding
errors preserve the compiler's actual cause.

AA assembly, address/symbol resolution against a process, and runtime behavior are explicitly
`not_run`. An application-only package has no native runtime graph to check. Static validation
does not type-check TypeScript, execute imports, check every runtime export, or prove addresses and
behavior. A report with `valid: true` means no checked error occurred; inspect its unrun checks.
Run behavioral checks separately in **Tests**; a static report does not include test-suite results.

Failed **user Apply** opens Problems. Apply checks the exact captured candidate before disabling
affected packages, so a static failure leaves the previous applied generation running. Agent checks
and command failures appear in the conversation without selecting Problems. Both surfaces use the
same reports and preserve original error codes, messages and details. Select a file diagnostic to
open its normal editor tab. Available line/column locations and markers apply only when the saved
file revision matches; dirty or changed files retain their contents and are identified as stale.

The latest saved report replaces earlier saved results for that package. Execution failures retain
their applied revision. Report history is bounded to 100 reports per game and lasts for the app
lifetime; tool outputs also persist in conversation history. Later runtime failures are readable
through `get_package_diagnostics`; they do not automatically start another assistant turn.

Execution returns `ok`, the result value, diagnostics, reports, and an error when present. Users and the
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

Assistant turns have no Studio-imposed duration limit. A turn continues until the
agent finishes, you select **Stop**, application shutdown cancels it, or an actual
runtime/provider failure occurs. Streaming and tool activity continue throughout
long runs. Stop still aborts the agent and waits for owned package tests to clean up.

## Package tests and current package

Open **Tests** beside Problems and select **Run tests**. The package ID at the far right of this tab
bar follows the active code-editor tab, including AI review tabs. Explorer package rows do not carry
a separate selected-package highlight; opening a package header opens its manifest. With no active
file there is no current package and Run tests is disabled.

Tests use saved files, so save your buffer first. Each run captures the package and installed
dependencies; each case gets a fresh worker copy. Lua and JavaScript use the real installed runtimes.
Integration cases also start a fresh owned process with known memory and pointer-chain addresses.
External test frameworks can use declared command cases. See [Testing](../../hexinton-engine-wiki/engine/testing.md)
for the declaration and test context. This is a controllable fixture, not a replacement for final
testing against the actual game.

The panel shows passed, failed, not-run and cancelled cases, durations, detailed failure causes and
expandable output. File failures open ordinary editor tabs with revision-aware locations. **Stop**
terminates the owned worker/target tree and skips remaining cases. Different packages can run
concurrently; a second run for the same game/package is rejected while the first is active.
History keeps the latest 20 runs per game for the app lifetime. Results obtained by the assistant
appear inline in its tool activity without switching the output tab; they also remain in chat history.

Both suites are mandatory in assistant instructions. Missing unit or integration cases produce
`not_run` and an **Incomplete** run, never a green empty suite. This policy guides the model; it
does not prove the model wrote adequate tests. Review assertions and coverage. Unavailable tooling
or a game-specific prerequisite must be reported with its actual reason, rather than faked coverage.
Save does not automatically run behavioral tests; on-save static diagnostics remain separate.

The assistant can use [Package Test Helpers v1](../../hexinton-engine-wiki/engine/package-test-helpers.md)
for assertions, temporary mocks, cleanup and Lua memory restoration. Studio supplies these inside
each test copy through `context.helpers`; users do not need to copy a helper folder into every
package. The assistant still needs to import actual package code, write meaningful checks and
await JavaScript cleanup scopes. This adds no test-generation button or automatic Apply behavior.

## References and terminals

Developer experiments have verified CE and Ghidra MCP through the actual assistant. These are
not automatic installation/attachment features yet; see the
[reverse-engineering MCP experiment](../../hexinton-engine-wiki/application/reverse-engineering-mcp-experiment.md)
for tested behavior and limits.

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

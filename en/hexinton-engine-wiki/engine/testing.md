# Testing and Failure Handling

Status: current public technical guide.

Scripts that read or modify another process should be tested against a controlled target before use
with a real game.

## Static checks and test runs

Studio's **Problems → Validate game** and the assistant's `validate_package` use shared static
checks without Apply or code execution. Reports include manifest, entries, hosted binding, native
dependency and Lua/JavaScript parse results. AA and runtime behavior are explicitly unrun. Failed
user Apply opens Problems and rejects static failures before lifecycle changes. File diagnostics
open the normal editor with revision-aware locations. See [Studio AI Assistant](../../start-here/getting-started/studio-ai-assistant.md#checks-and-error-feedback)
for the check limits and saved/applied error feedback.

Use Studio's **Tests** tab or the assistant's `run_package_tests` for separate behavioral results.
A syntax pass cannot establish that host functions exist, imports resolve at runtime, addresses are
correct, or cleanup restores memory.
Test declaration discovery is also performed by the test runner. A package can pass static
validation while `tests/studio.tests.json` references a missing test file; run the tests and inspect
their discovery result before treating the suite as verified.

## What to test

Test the operations your package actually performs, in proportion to its size. For most trainer
features one integration case is enough: run the production action against the controlled target,
check the bytes or value it wrote, then run the off/disable path and check the original state is
back. Add a unit case when the package has pure logic worth checking on its own (conversions,
clamping, parsing). A metadata- or UI-only change needs `validate_package`, not invented tests.

Add the checks below only when the package has the matching behavior:

| If the package... | Also check |
| --- | --- |
| takes action arguments | a valid value and one malformed value |
| scans for an AOB | the scan finds the seeded pattern; a missing pattern fails the action |
| follows a pointer chain | the resolved field; an unreadable pointer fails the action |
| installs a hook | the jump/cave bytes, and that off restores the original instruction |
| runs a timer or service | it stops and is destroyed on off/disable |
| depends on another package | the dependency's declared API, not its private names |
| has cleanup that can run before "on" | disable succeeds when the feature was never turned on |

Do not build test suites for situations the package does not handle (partial writes, foreign
patches, detach races) or add recovery code just so there is something to test.

The repository includes fake process, memory integration, Lua runtime, address resolver,
`AssemblyScript`, package host, dependency, and lifecycle tests. Use those tests as behavioral
examples when adding a new runtime feature.

## UE4SS Packages

Verify framework provisioning, on-demand load, finite entrypoints, serializable results, managed cleanup, service events and structured failures. Then test in an active game world: local-player selection, missing pawn, respawn, travel/save loading, repeated enable/disable and detach. The [location sample](samples/ue4ss-player-location.md) provides a bounded query/action/service exercise.

Native bridge fixtures inject a test DLL and run actual transport/lifecycle code with mocked reflection. They do not prove a game's APIs or complete client provisioning. The ordinary application does not yet automatically deliver the framework bundle. Studio test declarations accept `lua`, `js` and `command`; no `ue4ss` case runtime was added. Test pure host transformations with Lua cases and qualify Unreal operations through the prepared runtime/live attachment. Do not assume copied Studio test snapshots contain the framework.

## Package test declaration

Create `tests/studio.tests.json` inside the package. This is a separate test declaration; do not add
an unsupported `tests` field to `package.json`. Example:

```json
{
  "schemaVersion": 1,
  "cases": [
    { "id": "clamp-boundaries", "kind": "unit", "runtime": "lua", "entryFile": "tests/unit.lua" },
    { "id": "memory-restoration", "kind": "integration", "runtime": "lua", "entryFile": "tests/memory.lua", "timeoutMs": 10000 }
  ]
}
```

Each case has a unique `id`, `kind` (`unit` or `integration`), `runtime` (`lua`, `js`, or `command`)
and optional `timeoutMs` (default 10000; allowed 100–120000). Ordinary Lua/JS cases require an existing
package-relative `entryFile`; `entrySymbol` defaults to `runTest`. An integration Lua case with
ordered `steps` does not need a case-level `entryFile`. JS `capabilities` optionally
declare the existing application runtime capabilities the test uses. Entries may not escape the
package. Declarations are limited to 100 cases and 128 KiB.

`run_package_tests(packageId, kind="all")` captures the selected package's installed dependency
closure. Changing files during capture rejects the run with `test_snapshot_changed`; save and run
again. Unrelated packages are not copied or initialized. Missing dependencies are explicit failures.
Each case uses its own copied files and fresh process, including between two cases of one suite.
The runner never Applies, enables, or changes the user's live game session. Test code and custom
commands retain their normal tool access; this mechanism is not an operating-system sandbox.

The context passed to tests contains:

| Field | Meaning |
| --- | --- |
| `packageRoot` | Absolute path to this case's disposable package copy. |
| `scriptsRoot` | Captured packages directory; dependencies live under their package IDs. |
| `fixture` | `null` for unit cases; owned controlled target metadata for integration cases. |
| `helpers` | Versioned test-only modules: `version`, `luaFile`, `jsModule`. See [Package Test Helpers v1](package-test-helpers.md). |

Lua tests may return a function or a table with the entry symbol, or define that global symbol.
JavaScript tests export the entry symbol from their module. A thrown error/assertion or a returned
`false` fails the case; other returned values are diagnostic results, not assertion counts enforced
by Studio. Tests must exercise production code rather than repeating its implementation.

For a package with `lua/health.lua` returning a module with `clamp(value, maximum)`, a unit test is:

```lua
return function(ctx)
  local health = dofile(ctx.packageRoot .. '/lua/health.lua')
  assert(health.clamp(150, 100) == 100, 'clamp above max')
  assert(health.clamp(-10, 100) == 0, 'clamp below zero')
  assert(health.clamp(50, 100) == 50, 'valid value')
end
```

For JS, import production code normally, for example `import { clamp } from '../math.mjs'` in
`tests/unit.mjs`, and export `runTest(ctx)` with assertions that throw on a mismatch. Existing JS
module resolution and capability rules apply. TypeScript compilation is not built into this runner.

Unicode Windows working directories, Lua test entry filenames and entry symbols are supported.
The driver quotes Lua strings as UTF-8 byte escapes; context JSON can contain Unicode escapes.
`dofile`/`loadfile` imports and diagnostic filenames preserve UTF-8. Other stock Lua filesystem APIs
retain their own filename behavior; see [file loading](lua-runtime-utilities-api.md#loadfile-and-dofile).

[Package Test Helpers v1](package-test-helpers.md) supplies readable assertions, scoped temporary
mocks and cleanup, plus native Lua memory capture/restoration. Tests load these explicitly from
their context. Use the lifecycle examples there to check repeated enable/disable and failed enable
cleanup against actual production code. They are test utilities, not a scaffold generator or an
automatic lifecycle runner.

## Controlled integration target

Every integration case receives a fresh owned target process, never the actual game. Its 256-byte
allocation contains the 16-byte ASCII signature `HEXMODTESTPLAYER`, a pointer at
`signatureAddress + 16` to the player, and a pointer at `playerAddress + 32` to stats.
`fixture` provides `pid`, `baseAddress`, `size`, `signatureAddress`, `playerAddress`, and `statsAddress`.
At stats, 32-bit health is at `+0`, maximum health at `+4`, and gold at `+16`, initially 100 each.
These addresses are fixture contracts, not game offsets.
Use the supplied addresses instead of assuming where the fixture placed its player or stats.
If a test builds a different pointer graph inside the allocation, check every cell and record fits
within `baseAddress .. baseAddress + size`, preserve the fixture's signature/pointers/stats, and
restore all modified bytes. Overlapping a synthetic vitals record with fixture gold can make a
resolver test appear correct while corrupting unrelated state.

For a production module exposing `setHealth(statsAddress, value)` that clamps to max and returns
the resulting health:

```lua
return function(ctx)
  local health = dofile(ctx.packageRoot .. '/lua/health.lua')
  local stats = ctx.fixture.statsAddress
  local original = readInteger(stats)
  local ok, failure = pcall(function()
    assert(readPointer(ctx.fixture.signatureAddress + 16) == ctx.fixture.playerAddress)
    assert(readPointer(ctx.fixture.playerAddress + 32) == stats)
    assert(health.setHealth(stats, 150) == 100, 'must clamp to maximum')
    assert(health.setHealth(stats, 50) == 50, 'must write requested health')
  end)
  writeInteger(stats, original)
  assert(readInteger(stats) == original, 'must restore memory')
  assert(ok, failure)
end
```

Import dependencies explicitly from `ctx.scriptsRoot`. Lua uses the native runtime attached to the
owned target (unit Lua attaches only to its own worker). Ordinary entry-file cases do not automatically
call the production package's enable/disable hooks. Use the host-driven control steps below to test
the player's real toggle, or explicitly invoke hooks when testing a module's own exports.
Assert the effects and cleanup. Test pure address-independent logic separately, replacing
host/dependency functions where needed. Fixture tests cannot establish that real-game AOBs or
offsets are correct. Named test modules support the bounded layouts below; game behavior and
automatic AA/.NET framework discovery are not emulated.

## Named game memory and real toggle steps

An integration `lua` or `command` case may declare `fixture.modules`. The owned target reserves
real Win32 memory under these module names. The engine resolves `game.exe+offset` and
`aobScanModule(..., game.exe, ...)` against that memory, so production addresses and scans can stay
unchanged. This does not load the game's executable or emulate its gameplay.

```json
{
  "fixture": {
    "modules": [
      { "name": "game.exe", "size": 65536 },
      { "name": "DarkSoulsIII.exe", "size": 83886080,
        "committedRanges": [{ "offset": 74874880, "size": 8192 }] }
    ]
  }
}
```

`size` reserves the module's address span. Without `committedRanges`, the whole module is allocated;
modules larger than 16 MiB require explicit ranges, so a high game offset need not allocate the
whole span. Each `{offset, size}` range allocates the covering pages, must fit inside the module,
and has positive size. Use page-aligned ranges for a clear layout. At most 32 modules and 1024
ranges per module are supported; names are unique file names, case-insensitively, and sizes are
positive integers up to 1 TiB. Named memory is not currently supported by JavaScript test cases.

The returned context contains an array `ctx.fixture.modules` with `{name, base, size}` for each
module, in declaration order. For example, seed an AOB with
`writeBytes(ctx.fixture.modules[1].base + 0x80, 0x8B, 0x00, 0x41, 0x54, 0x90, 0x90)`.
Place every pointer cell and destination in allocated pages; pointer values must describe the
source's exact chain. Keep MaxHP and HP separate when the source uses separate fields.

For the player's actual toggle, declare an integration Lua case with ordered steps:

```json
{
  "id": "player-toggle", "kind": "integration", "runtime": "lua",
  "fixture": { "modules": [{ "name": "game.exe", "size": 65536 }] },
  "steps": [
    { "kind": "lua", "entryFile": "tests/setup.lua" },
    { "kind": "toggle", "widgetId": "example.package/feature", "enabled": true },
    { "kind": "lua", "entryFile": "tests/check-on.lua" },
    { "kind": "toggle", "widgetId": "example.package/feature", "enabled": false },
    { "kind": "lua", "entryFile": "tests/check-off.lua" },
    { "kind": "cleanup" },
    { "kind": "lua", "entryFile": "tests/check-cleanup.lua" }
  ]
}
```

The case allows 1–100 steps. A `lua` step uses a package-relative `entryFile` and optional
`entrySymbol` (default `runTest`), with the same context and owned target throughout the case.
Lua seeds or asserts memory, then returns to the host. A `toggle` step uses the compiled
`frontend.hosted` widget identified by `package-id/widget-id`, selects its declared on/off action,
and executes the actual manifest runnable through the same command builder as the app. It supplies
the app's empty parameter object `{}`; fixture addresses are not action arguments. A Lua action
therefore receives its normal single parameter table, not a positional test address. A `cleanup`
step runs the package's declared lifecycle disable through the host with `{}`. Check the resulting
memory afterward; calling cleanup alone is not an assertion.

Use named memory and these steps to keep the production resolver and invocation unchanged. Do not
add a scan-scope parameter, replace production addresses, or call a different function signature
just to make a fixture test pass. Direct `dofile` calls or `autoAssemble` of a production file can
check its implementation, but do not by themselves verify the player's command path. Step evidence
records the selected widget, runnable, parameters and cleanup; assertions still decide whether the
feature's effects match its source.

## External framework commands

A command case can run an existing unit or integration framework:

```json
{
  "id": "frontend-contract",
  "kind": "unit",
  "runtime": "command",
  "command": "node",
  "arguments": ["--test", "tests/frontend.test.mjs"],
  "timeoutMs": 30000
}
```

The command runs in the disposable package directory with an argument vector; Studio does not
implicitly invoke a shell. Specify `powershell.exe`, `dotnet`, or another executable when needed.
Arguments and executable paths support `{packageRoot}`, `{scriptsRoot}`, and `{fixtureManifest}`.
The last placeholder is a JSON **context file**, also available in `HEXMOD_TEST_CONTEXT`; its
`fixture` field contains integration metadata. Required tools/dependencies must already exist or
be prepared explicitly. Exit zero passes; nonzero exit preserves the exit code and stdout/stderr.
Output is bounded to the last 16000 characters with an explicit truncation notice.

## Results and limits

Studio and the assistant share the same run/results. Failures preserve code, message, file/line when
available, captured revision, duration and output. Imported-code failures navigate to that captured
production file. Dirty or revised editor buffers are preserved; stale locations are not installed.
A failed case does not suppress later cases. Timeout or Stop terminates its owned process tree,
including the fixture/command children. Worker startup errors are text failures, not runtime-install
dialogs. Shutdown cancels and waits for active tests before disposing workspace services.

The default `all` run requires both unit and integration cases. Missing coverage is `not_run`, with
overall `incomplete`; blocked/empty suites are never passing. A package with no pure logic declares
only integration cases and runs `run_package_tests(packageId, kind="integration")`. Hex Assistant
runs static validation and the suites that apply to the change, fixes failures and reports what was
not covered. Adequate assertions remain a review responsibility. Tests execute saved copies; Save's automatic static checks do not run these suites.

## Failure Rules

Treat these as failures, not empty successful results: no AOB match when a match is required, more
than one match for a unique pattern, unresolved address or pointer chain, missing dependency or
ambiguous symbol, unsupported runtime, invalid manifest, stale attachment, and assembly or memory
failure.

Never continue a write after a check the package performs (scan, pointer, expected bytes) has failed.

## Assertions that establish the feature contract

Import and exercise the same production operation that the runtime calls, not a copy of its logic.
Seed the fixture from the source's values (the original instruction bytes, the source's offsets),
not from constants copied out of the package under test. For an encoded branch or call, also check
the decoded destination. Inspect memory after a failure test: an exception alone does not show
whether anything was written.

Testing an address-accepting helper against fixture memory does not also test its production
resolver, package lifecycle or UI bindings. The engine can prepare a package before its first
query/action, so initial control reads must not apply a gameplay toggle, and both action IDs must
route to the intended operations.

Fixture tests cannot identify the running game build, and packages are not expected to. Record the
build a package was written or tested against in its `docs/` notes; do not gate operations on a
version check. See [game versions](engine-essentials.md#game-versions).

## Safe Development Loop

```text
write or update package
  -> validate manifest
  -> attach to a controlled target
  -> enable
  -> test one action
  -> verify the result
  -> disable
  -> verify restoration
```

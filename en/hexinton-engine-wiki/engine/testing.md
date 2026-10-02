# Testing and Failure Handling

Status: current public technical guide.

Scripts that read or modify another process should be tested against a controlled target before use
with a real game.

## Minimum Test Cases

Studio's **Problems → Validate game** and the assistant's `validate_package` use shared static
checks without Apply or code execution. Reports include manifest, entries, hosted binding, native
dependency and Lua/JavaScript parse results. AA and runtime behavior are explicitly unrun. Failed
user Apply opens Problems and rejects static failures before lifecycle changes. File diagnostics
open the normal editor with revision-aware locations. See [Studio AI Assistant](../../start-here/getting-started/studio-ai-assistant.md#checks-and-error-feedback)
for the check limits and saved/applied error feedback.

Package-facing unit/integration test runners are a later step. A syntax pass cannot establish that
host functions exist, imports resolve at runtime, addresses are correct, or cleanup restores memory.

Verify each package with:

1. valid `enable` and `disable` execution;
2. action arguments, including missing and malformed values;
3. missing AOB patterns and ambiguous matches;
4. invalid address and pointer expressions;
5. read/write width and signedness;
6. dependency-first enable and reverse cleanup;
7. timer and service shutdown;
8. stale or detached process sessions;
9. failed enable rollback;
10. failed disable reporting.

The repository includes fake process, memory integration, Lua runtime, address resolver,
`AssemblyScript`, package host, dependency, and lifecycle tests. Use those tests as behavioral
examples when adding a new runtime feature.

## Failure Rules

Treat these as failures, not empty successful results: no AOB match when a match is required, more
than one match for a unique pattern, unresolved address or pointer chain, missing dependency or
ambiguous symbol, unsupported runtime, invalid manifest, stale attachment, and assembly or memory
failure.

Never continue a write after compatibility validation has failed.

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

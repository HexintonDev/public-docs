# From live values to static analysis

**Developer experiment, exercised against a controlled target on 2026-10-03.**
This workflow uses the existing CE and Ghidra MCP extension connections. It does
not add automatic installation, attachment or tool launch controls to Studio.
See [connection prerequisites](reverse-engineering-mcp-experiment.md).

## Establish the authoritative field

1. Discover CE instances and select the exact CE host. Attach to the intended game
   PID; the host PID, game PID and MCP instance ID are different identities.
2. Scan for a visible value, perform a known gameplay change, refine the scan and
   read the remaining candidates. NPCs and current/previous HUD snapshots may all
   contain the same number. A candidate tracking gameplay is not sufficient proof.
3. In an owned test process, make a reversible bounded write to a candidate, check
   actual gameplay state, then restore it. Writing a HUD copy may be overwritten
   on the next frame without changing the underlying actor. Avoid write experiments
   in a user's real game without their requested execution scope.
4. Verify field types individually. Adjacent floats can share an initial value:
   maximum shield and stamina were both 125 in the controlled target. Change stamina
   through a dash and reread exact addresses before assigning offsets.

## Find the writer and move to Ghidra

Attach the CE debugger before starting a bounded write capture. Trigger a small
gameplay event and poll the capture for instruction addresses and registers. Keep
the actual error when capture setup fails; correct the cause before retrying.
The captured register may point to a stats block, while its caller receives an
actor containing a pointer to that block.

CE can reject excessive concurrent dispatches with `busy`. Inspect `hostEffect`:
`not_started` allows a retry after outstanding calls finish. Serialize dependent
scans, pointer walks, captures and writes; preserve state before deciding whether
an operation reported as started is safe to repeat.

Use the exact binary name returned by Ghidra's `list_project_binaries`, and wait
for analysis/indexing to finish. Translate a live module instruction using:

```text
RVA = live instruction address - live module base
Ghidra address = Ghidra image base + RVA
```

Confirm instruction bytes at both addresses, decompile the containing function,
inspect its callers/xrefs and compare the logic with controlled gameplay changes.
These formulas apply to module addresses; arbitrary heap pointers do not have a
meaningful module RVA. Do not assume Ghidra and the live process use the same base.

An optimized executable without symbols will often have `FUN_...` names. Searching
for a source function name or relying on semantic search can miss the relevant
routine. Captured writers, instruction addresses, string references and caller
xrefs provide concrete ways to navigate. A successful MCP call can still contain
a per-item analysis error: inspect results rather than only transport status.

## Prove the resolver survives replacement

Use exact pointer references and the decompiled ownership/control flow to find a
module-relative root and the current-player path. Broad offset searches can produce
accidental chains through unrelated allocations. In the controlled evaluation, a
C++ locale facet initially looked like a root because the health block happened to
be nearby. Static inspection rejected that hypothesis.

Keep the **address of the root slot** separate from the **pointer stored in it**.
For example, `getAddressSafe('game.exe+0x1234')` resolves that module address;
`readPointer(slotAddress)` reads its contents. Passing the slot address directly
to a function expecting the world/object pointer skips a dereference. Tests should
exercise the complete public resolver, including module lookup and that first read,
as well as any lower-level traversal helper.

Resolve each link again after player replacement or scene changes. A retired actor
can remain readable, so a successful memory read does not prove it is current.
Change the new player and check that commands affect it while old/NPC data remains
unchanged. Also test a fresh process with ASLR; survival across replacement alone
does not prove survival across restart or a different game build.

Some visible values are derived or encoded. The controlled target's plain credit
scan returned HUD copies. Purchases and the transaction routine must be investigated
before proposing wallet writes. Read-only observations should remain read-only
until the underlying representation and update behavior are established.

## Turn discoveries into a package

Use Studio's writable-package preparation, real API contracts and a resolver that
checks pointers and refreshes them per command. Record the tested executable/build
and any signature/RVA assumptions. Write both unit and integration suites importing
production code, then run `validate_package` and `run_package_tests`.

Controlled test fixtures and mocks verify specific behavior; they do not establish
compatibility with the analyzed game. Apply and run the package only within the
user's requested scope, then independently check real target state. Preserve failed
test/runtime errors and distinguish a committed Apply from successful activation.
See [package testing](../engine/testing.md) and
[test helpers](../engine/package-test-helpers.md).

## Keep a reusable investigation checkpoint

Save the tested binary identity, module RVAs, field types, pointer traversal,
rejected candidates, reversible experiments and remaining uncertainties before
switching from analysis to package coding. Reuse this evidence rather than asking
a coding conversation to rediscover the target. Clearly label findings supplied
by a person or evaluator as assisted analysis.

Large tool catalogs and long analysis transcripts can cause repeated context
compaction. A focused conversation with the required tools and a concise evidence
checkpoint can help, but it does not prove that a generated package is correct.
Require actual validation, test results and live execution evidence. A plan to run
tests, or a successful Apply, is not a passed test suite.

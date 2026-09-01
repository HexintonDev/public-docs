# Hexinton Engine Wiki Framework

Status: current public documentation framework.

This page defines how the public Hexinton Engine pages, implementation references, and test
evidence are organized into a searchable English wiki.
It is not a basic tutorial and it is not a promise that every planned feature already exists.

## Goals

The finished wiki must allow a script author or AI assistant to:

- identify the correct runtime for a task;
- create a valid package and runnable;
- find an address with a module expression, pointer chain, or AOB pattern;
- read and write typed values safely;
- create and remove allocations, symbols, patches, and hooks;
- configure trainer controls, services, view models, and custom interfaces;
- test a script against a controlled target;
- distinguish current behavior from planned or unsupported behavior.

The wiki should be reference-first. Each page covers one stable topic, gives complete API behavior,
and links to focused examples instead of placing every feature in one long tutorial.

## Proposed Information Architecture

### 00. Orientation

| Page | Responsibility |
| --- | --- |
| `README.md` | Wiki entry point, capability map, terminology, status policy |
| `runtime-model.md` | Lua, Auto Assembler, application JavaScript, and host boundaries |
| `concepts.md` | Process session, script context, package, runnable, symbol, allocation, revision |
| `compatibility.md` | Target architecture, game versions, failure policy, and unsupported cases |

### 01. Package and Lifecycle

| Page | Responsibility |
| --- | --- |
| `script-packages.md` | Package layout, manifest schema, runnable declarations |
| `lifecycle-and-dependencies.md` | Enable/disable order, dependency resolution, rollback, cleanup |
| `lua-modules.md` | Package-local `require`, module IDs, cache, and failure modes |
| `errors-and-results.md` | Structured results, warnings, errors, and terminal command outcomes |

### 02. Lua Runtime API

| Page | Responsibility |
| --- | --- |
| `lua-api-reference.md` | Complete function index grouped by capability |
| `process-discovery.md` | Process and window lookup, process IDs, explicit handles |
| `memory-and-addresses.md` | Typed I/O, address expressions, pointer chains, safe resolution |
| `symbols-and-allocations.md` | Script-local names, published symbols, allocation ownership |
| `timers-and-services.md` | Timer lifecycle, service execution, event publication, live state |

### 03. Finding Addresses

| Page | Responsibility |
| --- | --- |
| `aob-scanning.md` | AOB syntax, wildcards, module and unique scans, match validation |
| `pointer-chains.md` | Pointer traversal, nesting, failure handling, examples |
| `address-validation.md` | Module checks, architecture checks, range checks, compatibility guards |

### 04. Patching and Hooks

| Page | Responsibility |
| --- | --- |
| `auto-assembler.md` | Directives, allocation, labels, chunks, assembly result handling |
| `hooks.md` | Scan-driven hooks, overwrite length, return paths, restoration |
| `patching.md` | Reversible byte patches and enable/disable design |
| `assembly-and-labels.md` | Persistent script names versus pass-local assembler labels |

### 05. Trainer UI

| Page | Responsibility |
| --- | --- |
| `trainer-controls.md` | Actions, toggles, numbers, readouts, queries, and control mapping |
| `view-models-and-feeds.md` | Services, authoritative state, revisions, and UI synchronization |
| `custom-surfaces.md` | Surface declarations, bindings, sandbox, protocol, and limits |
| `ui-examples.md` | Focused control examples: toggle, numeric action, search, inventory, teleport |

### 06. Testing and Samples

| Page | Responsibility |
| --- | --- |
| `testing.md` | Test strategy, fake targets, integration checks, and regression rules |
| `sample-library.md` | Small indexed examples with permanent URLs and expected behavior |
| `troubleshooting.md` | Manifest, runtime, attachment, scan, assembly, and UI failures |
| `security-and-safety.md` | Trusted packages, process writes, cleanup, privacy, and online-game warnings |

## Public Evidence Sources

The public pages in this repository are the reader-facing source of truth for the first release.
Executable behavior is cross-checked against the Hexinton Engine implementation and test fixtures in
the source repository. Internal architecture, product design, requirements, and implementation
plans are intentionally excluded from this repository.

The implementation and test evidence comes from the Hexinton Engine source repository. Source
repository URLs are intentionally omitted from this public site.

Before publishing a final page, reconcile conflicting statements against the implementation and
tests. Mark design-only behavior as planned instead of presenting it as current.

## Fixed URL Sample Library

Examples must not be embedded as an ever-growing block of code in every page. Store each focused
sample once and link to it from the relevant reference page.

Each sample entry should have:

- a stable sample ID, such as `lua/read-integer` or `hook/aob-returnhere`;
- one short purpose statement;
- a complete package or file path;
- expected inputs and outputs;
- required target assumptions;
- cleanup behavior;
- links to the source fixture and the test that validates it.

Keep a permanent evidence record for every published sample. When behavior must be historically
exact, record the source commit and repository-relative evidence paths in the sample page:

```text
ProcessEngine/tests/script_fixtures/<sample>/...
ProcessEngine/tests/<test-file>
```

Use `main` URLs only for living examples that are intentionally kept in sync with the current
implementation. The sample index should record the commit used for each release of the wiki.

## Evidence Rules for Every Page

Every technical claim should be classified as one of:

| Evidence | Meaning |
| --- | --- |
| Implementation | Behavior present in the current Hexinton Engine or host code |
| Test | Behavior covered by a repository test or fixture |
| Existing docs | Already documented but requiring implementation reconciliation |
| Planned | Design or future behavior that is not safe to use as current API |

An API entry is not complete until it documents its signature, input constraints, return value,
failure behavior, process/session scope, lifecycle ownership, and at least one focused sample.

## Build Order

1. Approve this information architecture and naming. Completed for the current public Wiki.
2. Build the sample index and associate each example with local implementation/test evidence. The
  initial library is completed at [samples/README.md](samples/README.md).
3. Reconcile package and Lua runtime contracts.
4. Reconcile memory, pointer, and AOB behavior.
5. Reconcile Auto Assembler, patch, and Hook behavior.
6. Document trainer controls, view models, and custom surfaces.
7. Add cross-links, AI retrieval metadata, troubleshooting, and security guidance.
8. Run link checks and validate every example against the relevant fixture or test.

## Definition of Done

The wiki is ready for public use when:

- every current engine capability has one owning page;
- no page claims a feature that is only a mock, design, or placeholder;
- every API family has signatures, constraints, failures, scope, and examples;
- examples link to fixed source and test URLs;
- package, Lua, UI, and cleanup behavior are documented together without duplication;
- all Markdown links resolve;
- an AI can answer “how do I implement X?” by following the capability map to a focused page and
  then to a validated sample.
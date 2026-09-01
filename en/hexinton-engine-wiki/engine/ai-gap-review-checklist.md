# AI Gap Review Checklist

Status: public documentation maintenance procedure.

Use this checklist whenever the Hexinton Engine implementation, package schema, host protocol, or
test fixtures change. The goal is to find missing documentation, not to generate confident-looking
pages from assumptions.

## Evidence Order

For every claim, inspect evidence in this order:

1. Current implementation and public registrations.
2. Focused automated tests and fixtures.
3. Existing public documentation.
4. Design or planned material, clearly marked as planned.

Never promote a design statement to current API behavior without implementation or test evidence.

## API Matrix

For each public Lua or Auto Assembler API, record:

| Field | Required question |
| --- | --- |
| Name | Is spelling and case sensitivity documented? |
| Runtime | Is it Lua, Auto Assembler, or application JavaScript? |
| Signature | Are argument types, optional arguments, and overloads shown? |
| Inputs | What values are valid, invalid, empty, or ambiguous? |
| Return | What is returned on success and failure? |
| Scope | Which process, script, package, or session owns the result? |
| Lifecycle | What must `disable` or teardown release? |
| Threading | Is execution serialized, callback-driven, or UI-owned? |
| Example | Is there a minimal copyable example? |
| Evidence | Is the implementation/test path and revision recorded? |

An API entry is incomplete when any required field is unknown. Mark the unknown as a gap instead of
inventing behavior.

## Lua Coverage

Audit these groups independently:

- process and window discovery;
- default and explicit process handles;
- address expressions and safe resolution;
- every typed memory read and write;
- strings, byte arrays, widths, signedness, and encoding;
- AOB scans, module scans, unique scans, and empty results;
- symbols and ownership;
- Auto Assembler invocation and result handling;
- timers, sleep, services, and event publication;
- architecture and target checks;
- JSON-compatible return values and errors.

Each group must link to at least one focused page and one runnable example.

## Auto Assembler Coverage

Audit directives and behaviors separately:

- allocation and deallocation;
- labels and symbol registration;
- module, region, and ordinary AOB scans;
- assertions and failure behavior;
- jumps, overwrite length, and return paths;
- thread creation and its cleanup requirements;
- enable/disable action pairing;
- partial failure rollback;
- x86/x64 and target-module assumptions.

Assembly examples must state what the engine does automatically and what the author must implement.
Do not describe allocation as a complete hook.

## Architecture Coverage

Confirm the Wiki explains:

- application layer versus Hexinton Engine layer;
- package discovery, validation, execution, and cleanup;
- Lua worker-thread confinement;
- timer callback ordering;
- command result versus authoritative state feed;
- service-backed ViewModel lifecycle;
- query, command, feed, and surface boundaries;
- loading, stale, unavailable, and error UI states.

## Gap Report Format

For each missing item, write one row:

| Gap | Evidence checked | Risk | Required page/example | Status |
| --- | --- | --- | --- | --- |
| Missing signature, failure rule, or example | Implementation/test paths | Low, medium, or high | Owning page and sample | Open or resolved |

Prioritize gaps that could cause an unsafe memory write, leaked hook/allocation, wrong process
selection, stale UI state, or an AI to recommend an unsupported feature.

## Completion Rules

An audit is complete only when:

- every public registration is present in the API matrix;
- every API family has a runnable example or an explicit reason it cannot have one;
- failures and cleanup are documented;
- architecture and threading boundaries are linked from the Wiki entry point;
- local links pass validation;
- changes are committed to the public documentation repository and synchronized to GitBook.
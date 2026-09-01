# Sample: Auto Assembler Allocation Action Pair

Sample ID: `aa/allocation-action-pair`

This sample demonstrates an Auto Assembler action that allocates a named code cave and a second action
that releases it.

## Behavior

The `enable-godmode` action runs `alloc(godmodeCave, 32)`. The matching `disable-godmode` action
runs `dealloc(godmodeCave)`. The Lua package lifecycle remains Lua because Auto Assembler is
action-only in the native host.

## Files

```text
package.json
aa/enable-godmode.aa
aa/disable-godmode.aa
```

The enable file contains `alloc(godmodeCave, 32)` followed by `registerSymbol(godmodeCave)`.
The disable file contains `unregisterSymbol(godmodeCave)` followed by `dealloc(godmodeCave)`.

## Reuse Rules

- Declare Auto Assembler runnables with `runtime: "aa"` and an `entryFile`.
- Do not use Auto Assembler for `enable`, `disable`, or a continuously running service.
- Pair every `alloc` with a deterministic `dealloc` action or a Lua `disable` cleanup path.
- Build real hooks on top of validated scans and restoration logic; allocation alone is not a hook.

## Expected Result

The first action creates `godmodeCave`; the second action removes it. The test verifies the
allocation/action lifecycle through the package command host.
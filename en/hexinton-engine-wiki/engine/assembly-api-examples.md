# Auto Assembler API Examples

Status: current public assembly example reference.

## Assembly Action Manifest

An Auto Assembler runnable points to an assembly file:

```json
{
  "id": "enable-cave",
  "kind": "action",
  "runtime": "aa",
  "entryFile": "aa/enable-cave.aa"
}
```

## Allocate and Release

Enable action:

```asm
alloc(exampleCave, 0x1000)
registersymbol(exampleCave)
```

Matching disable action:

```asm
unregistersymbol(exampleCave)
dealloc(exampleCave)
```

Every allocation and symbol must have an owner and a cleanup path.

## Module-Scoped Scan

```asm
aobScanModule(injection, game.exe, 48 8B ?? ?? 89)
```

A scan must be compatible with the target build. The current directive binds the first match and
fails when there is no match; choose a scope and pattern known to produce one safe result. Do not
rely on `assert(...)`: it is not currently a parsed engine directive.

## Manual Hook Shape

```asm
alloc(newmem, 0x1000, injection)
label(returnhere)

newmem:
  ; replacement instructions
  jmp returnhere

injection:
  jmp newmem
  nop
returnhere:
```

The author must choose a safe overwrite length, preserve displaced instructions where required,
and restore original bytes during disable. The current engine does not synthesize a complete
trampoline or calculate overwrite length automatically.

## Directives

The current assembly parser includes directives for `aobScan`, `aobScanModule`,
`aobScanRegion`, `alloc`, `dealloc`, `label`, `registerSymbol`, `unregisterSymbol`, and
`createThread`. Validate each directive against the target architecture and the package lifecycle.

## Result Handling

When Auto Assembler is invoked through Lua, check the returned success flag and error text. When it
is invoked as a package runnable, surface the terminal result to the user and keep disable actions
available after a partial enable failure.

# Hooks and Auto Assembler

Status: current public technical guide.

Auto Assembler is the action-oriented assembly runtime. The current engine supports allocations,
labels, AOB/module/region scans, access changes, thread execution, jumps, and assembly chunks. Hook installation remains manual: the author
must choose a safe overwrite length and restore original bytes during disable.

See the [Auto Assembler API](auto-assembler-api.md) for directive syntax, argument rules, and
current limitations.

## Basic Assembly Action

```lua
function enable()
  local ok, errorMessage = autoAssemble([[
alloc(newmem, 0x1000)
label(returnhere)

newmem:
  nop
  jmp returnhere
]])
    if not ok then
        error(errorMessage or "assembly failed")
    end
end

function disable()
  local ok, errorMessage = autoAssemble([[
unregisterSymbol(newmem)
dealloc(newmem)
]])
    if not ok then
        error(errorMessage or "assembly cleanup failed")
    end
end
```

Use `autoAssemble(text)` from Lua. It returns `true` or `false, errorMessage`; check that result
and provide the matching explicit disable operation. CE's `targetself=true` and `disableInfo`
behaviors are unsupported. See the [Lua assembly API](lua-scanning-and-assembly-api.md#autoassemble).

## Scan-Driven Hook Shape

```text
aobScanModule(injection, game.exe, 48 8B ?? ?? 89)
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

Persistent names should be explicit script allocations or labels. Assembly labels created only in
one assembly pass do not automatically become persistent script labels.

## Disable Requirements

An enable implementation must record or deterministically derive the patched address, original bytes,
every allocation and label it owns, and every timer or symbol created by the hook. On disable, restore
original bytes before releasing the code cave, stop timers, and unregister published symbols.

When a Lua trainer manages several features inside one package, its adapter owns
their shared setup. Retain a prerequisite while any active consumer needs it;
restore consumers before releasing their providers. Exclusive modes should release
the old mode before installing the new one. Failed activation must roll back its
acquired resources, and failed cleanup must retain enough ownership to retry.
Successful pointer actions may need to retain setup until package disable: the
target can be a flag inside an allocation used by the hook. Release it early only
when the target's lifetime is known. Freezes and saved values retain setup until
their timer stops or their original value is restored.

A prerequisite and consumer that both patch the same scan match need verified hook
composition. Preserve the dependency and reject the affected feature until its
patch order and restore behavior are established. Removing the dependency or
following an arbitrary installed jump cannot establish a correct hook.

## Limitations

The current engine does not automatically discover instruction overwrite length, relocate displaced
instructions, or synthesize a complete trampoline. Validate assembled code against a compatible game
build before distributing it.

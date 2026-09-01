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
    -- Restore the patched site and deallocate newmem.
end
```

Use `autoAssemble(text, targetself?, disableInfo?)` from Lua. Always check its result and make the
disable path symmetric with enable.

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

## Limitations

The current engine does not automatically discover instruction overwrite length, relocate displaced
instructions, or synthesize a complete trampoline. Validate assembled code against a compatible game
build before distributing it.
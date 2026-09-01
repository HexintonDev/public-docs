# Auto Assembler API

Status: current public directive and assembly reference.

Auto Assembler parses a source string in the current package script context. It executes control
directives and assembles instruction chunks into the target process. It is synchronous under the
process execution lane: a successful call means that the requested chunks were assembled and flushed
before the call returned.

There are two different kinds of lines:

1. **Directives** such as `alloc(...)`, `aobScanModule(...)`, and `dealloc(...)` change runtime
   state and are handled by Hexinton Engine.
2. **Assembly lines** such as `mov`, `lea`, `cmp`, `jmp`, `call`, `nop`, and `ret` are parsed by the
   x86/x64 text assembler and written to the current target chunk.

Directive names are case-insensitive. Names and expressions are resolved in the current script and
session context. The instruction dialect is x86/x64 AsmJit/AsmTK syntax; this page documents engine
behavior, not every instruction available in that assembler.

## Directive Reference

| Directive | Syntax | Behavior | Failure |
| --- | --- | --- | --- |
| `alloc` | `alloc(name[, size[, nearAddress]])` | Allocates memory owned by the current script; omitted size is `0x1000` bytes | Invalid count, size, address, name, or allocation conflict |
| `globalalloc` | `globalalloc(name[, size[, nearAddress]])` | Allocates session-global memory | Same validation and allocation failures as `alloc` |
| `dealloc` | `dealloc(name)` | Releases a local or global allocation and linked names | Name is missing or cannot be released |
| `aobScan` | `aobScan(name, pattern)` | Scans the process and binds the first hit to `name` | Invalid pattern or zero hits |
| `aobScanModule` | `aobScanModule(name, module, pattern)` | Scans one module and binds the first hit | Invalid module/pattern or zero hits |
| `aobScanRegion` | `aobScanRegion(name, start, stop, pattern)` | Scans the half-open address range selected by `start` and `stop` | Invalid expressions/range/pattern or zero hits |
| `fullAccess` | `fullAccess(address, size)` | Requests writable/executable access for a target range | Invalid range or protection failure |
| `createThread` | `createThread(address)` | Executes the resolved address through the session code-execution facility | Unresolved address or execution failure |
| `label` | `label(name)` | Declares a script-visible label that can be bound by a later `name:` line | Invalid count or name |
| `registerSymbol` | `registerSymbol(name)` | Explicitly publishes a script label or allocation | Unresolved name or symbol conflict |
| `unregisterSymbol` | `unregisterSymbol(name)` | Removes a symbol published by this script | Invalid count or symbol failure |

The parser accepts the lower-case spellings `registersymbol` and `unregistersymbol` as well.
Unknown function-like lines are not silently treated as directives; they are passed to the assembler
and normally fail as invalid assembly.

## Source Layout and Current Address

An address header starts a new assembly chunk when its expression resolves immediately:

```asm
alloc(codeCave, 0x1000)

codeCave:
  xor eax, eax
  ret

game.exe+0x123456:
  nop
```

The first chunk is written at `codeCave`; the second is written at the raw process address. A raw
instruction cannot appear before an address or allocation header. An allocation-backed chunk is
bounded by the allocation size. A raw target has no invented capacity limit, so the author must
ensure the patch fits the intended region.

An address header is not the same as an internal label. If `loop:` does not resolve as a script,
allocation, module, or session symbol, it is an assembler label inside the current chunk:

```asm
alloc(codeCave, 0x100)

codeCave:
  xor eax, eax
loop:
  inc eax
  cmp eax, 10
  jl loop
  ret
```

Internal labels are usable by instructions in that chunk only and are not persistent script symbols.
To create a label that can connect separate chunks, declare it first with `label(name)`.

## Allocation and Symbol Ownership

`alloc` belongs to the current script context. `globalalloc` belongs to the engine session and should
only be used when the storage must outlive one script context. Neither directive publishes a symbol
for other scripts automatically.

Enable:

```asm
alloc(codeCave, 0x1000)
registerSymbol(codeCave)
```

Disable, in this order:

```asm
unregisterSymbol(codeCave)
dealloc(codeCave)
```

Unregister published names before releasing their backing allocation. Package disable logic remains
responsible for restoring patched bytes and releasing every package-owned resource.

## AOB Scans

Use `??` for a wildcard byte. The scan directives bind only the first result; they do not enforce
uniqueness. A pattern with multiple hits is therefore not automatically safe to patch.

```asm
aobScan(processWideHit, 48 8B ?? ?? 89)
aobScanModule(moduleHit, game.exe, 48 8B ?? ?? 89)
aobScanRegion(regionHit, game.exe+0x1000, game.exe+0x9000, 48 8B ?? ?? 89)

moduleHit:
  nop
```

Zero hits are errors. Before shipping a package, verify that the selected pattern is unique enough
for the supported game build and that the matched instruction boundary is safe to overwrite.

## Access, Calls, and Threads

`fullAccess` takes an address expression and a positive byte count. It does not assemble bytes or
restore the previous protection automatically:

```asm
fullAccess(game.exe+0x200000, 0x1000)
```

`createThread` resolves one entry address and immediately asks the session to execute it. It does
not return a thread handle and does not provide join, cancellation, or automatic allocation cleanup:

```asm
alloc(threadEntry, 0x100)

threadEntry:
  xor eax, eax
  ret

createThread(threadEntry)
```

The entry code must match the target architecture and calling/return expectations. Treat this
directive as an advanced operation; keep the allocation alive for as long as the target can execute
the entry point.

## Complete Manual Hook Shape

This is the shape of a manual hook, not an automatic trampoline generator:

```asm
aobScanModule(injection, game.exe, 48 8B 05 ?? ?? ?? ??)
alloc(newmem, 0x1000, injection)
label(returnhere)
registerSymbol(injection)

newmem:
  ; Re-emit displaced instructions when required by the chosen overwrite length.
  mov rax, [rip+0x1234]
  jmp returnhere

injection:
  jmp newmem
  nop

returnhere:
```

The engine does not calculate overwrite length, disassemble and relocate displaced instructions,
create a complete trampoline, save original bytes, or restore the hook site during `dealloc`.
Those operations must be implemented by the package's enable/disable design. `nop` padding must
match the number of bytes deliberately overwritten; do not guess it from the visible source lines.

## Lua Invocation

Lua calls Auto Assembler with the source string and receives `true` on success or `false, error`
on failure:

```lua
function enable()
    local ok, errorMessage = autoAssemble([[
alloc(codeCave, 0x1000)
registerSymbol(codeCave)

codeCave:
  xor eax, eax
  ret
]])
    if not ok then
        error(errorMessage or "assembly failed")
    end
end
```

The optional `targetself=true` argument is not supported. The optional `disableInfo` argument is
also not supported, so use a separate disable action and explicit cleanup. The Lua function does not
automatically undo a successful enable when a later package operation fails.

## Errors and Unsupported Directives

Failures include the source line number for directive errors and assembly-chunk context for parser
or flush errors. A failed scan, unresolved label, invalid instruction, protection change, or write
failure must be handled as an enable failure.

`assert(...)` is not a supported directive. Use a constrained AOB scan, explicit address checks, and
the package error/disable path instead. The engine also does not provide automatic rollback for all
side effects, so keep enable actions ordered and make disable actions idempotent.

See [Assembly and Labels](assembly-and-labels.md), [Hooks and Auto Assembler](hooks-and-auto-assembler.md),
and [Errors and Results](errors-and-results.md).

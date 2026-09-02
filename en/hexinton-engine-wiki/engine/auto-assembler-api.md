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

The engine implements the following 11 directives. Directive names are case-insensitive, and
arguments are comma-separated while bracketed address expressions remain one argument. Every
directive error includes its source line number.

### `alloc`

```asm
alloc(name[, size[, nearAddress]])
```

Allocates executable/read-write memory owned by the current script context and binds `name` to the
allocation. `size` is an address-sized integer expression and defaults to `0x1000` bytes. When
`nearAddress` is supplied, the allocator uses it as the placement hint for the allocation. The
allocation can then be used as a segment target or address expression.

The name must be valid and must not conflict with an existing allocation or incompatible label. A
missing name, more than three arguments, invalid size/address expression, zero/invalid size, or
allocation failure is an error. `alloc` does not register the name as a public symbol automatically.

```asm
alloc(codeCave, 0x1000, game.exe+0x123456)

codeCave:
  xor eax, eax
  ret
```

### `globalalloc`

```asm
globalalloc(name[, size[, nearAddress]])
```

Allocates memory in the engine session's global allocation store rather than the current script's
local allocation store. It uses the same default size of `0x1000`, optional near-address hint,
name validation, and maximum of three arguments as `alloc`. A global allocation can outlive one
script context, so it must only be used for storage intentionally shared or intentionally persistent
across script lifecycles.

An invalid name, size, near address, allocation conflict, or allocation failure is an error. The
directive does not automatically publish a symbol to other scripts.

```asm
globalalloc(sharedState, 0x100)
```

### `dealloc`

```asm
dealloc(name)
```

Releases the allocation identified by `name`. The lookup covers a local allocation and the session's
global allocation store as supported by the script context. The directive accepts exactly one
argument and fails when the name cannot be found or released. Deallocation does not restore bytes
written into the allocation's consumers and does not replace symbol cleanup.

```asm
unregistersymbol(codeCave)
dealloc(codeCave)
```

Unregister published names before deallocating their backing allocation. For a `globalalloc`, make
the owner and final cleanup path explicit because its lifetime is not limited to one script.

### `aobScan`

```asm
aobScan(name, pattern)
```

Scans the attached process for a byte pattern and binds the first match to the script label `name`.
`??` is the wildcard byte token supported by the underlying AOB scanner. The directive accepts
exactly two arguments. Invalid pattern text, scan failure, zero matches, an invalid label name, or
a conflict with a previously bound label is an error.

When a pattern has multiple matches, the directive still binds the first hit. It does not enforce
uniqueness, so package validation must make the pattern specific enough for every supported build.

```asm
aobScan(injection, 48 8B ?? ?? 89)

injection:
  nop
```

### `aobScanModule`

```asm
aobScanModule(name, module, pattern)
```

Scans only the named loaded module and binds the first match to `name`. The directive accepts exactly
three arguments: result name, module expression/name, and AOB pattern. Invalid module or pattern
input, scan failure, zero matches, invalid name, or label conflict is an error.

Module scoping reduces accidental matches but does not guarantee uniqueness. Confirm the selected
instruction boundary and expected bytes before using the bound label as a hook target.

```asm
aobScanModule(injection, game.exe, 48 8B 05 ?? ?? ?? ??)
```

### `aobScanRegion`

```asm
aobScanRegion(name, start, stop, pattern)
```

Resolves `start` and `stop`, scans the address region selected by those bounds, and binds the first
match to `name`. It accepts exactly four arguments. Address expressions can refer to modules,
symbols, allocations, or supported pointer chains.

Invalid bounds, an invalid range, invalid pattern text, zero matches, invalid name, or a label
conflict is an error. The directive does not enforce unique matches within the region. Keep the
range and pattern aligned with the target build and intended instruction boundary.

```asm
aobScanRegion(regionHit, game.exe+0x1000, game.exe+0x9000, 48 8B ?? ?? 89)
```

### `fullAccess`

```asm
fullAccess(address, size)
```

Resolves `address` and requests writable/executable access for a positive byte range of `size` in
the target process. It accepts exactly two arguments. `size` is resolved as an address-sized
integer expression and must be greater than zero.

Invalid address or size expressions, a non-positive range, unavailable process state, or a
protection failure is an error. `fullAccess` does not emit instructions, write patch bytes, record
the previous protection, or restore protection automatically.

```asm
fullAccess(game.exe+0x200000, 0x1000)
```

### `createThread`

```asm
createThread(address)
```

Resolves `address` and immediately passes it to the session code-execution facility. It accepts
exactly one argument, returns no assembly-level value, and does not return a thread handle. An
unresolved address or execution failure is an error.

The entry point must match the target architecture and the session's code-execution/calling
expectations. The directive does not provide join, cancellation, completion status, or automatic
cleanup of memory used by the entry point. Keep any allocation alive for as long as the target can
execute it.

```asm
alloc(threadEntry, 0x100)

threadEntry:
  xor eax, eax
  ret

createThread(threadEntry)
```

### `label`

```asm
label(name)
```

Declares a script-visible label that can be bound by a later `name:` assembly header. The directive
accepts exactly one argument and resets the label's pending binding. If `name` already identifies a
local or global allocation, the directive leaves that allocation target available instead of
replacing it.

A missing/extra argument or invalid label name is an error. A declaration alone does not assign an
address; the label becomes useful after a matching assembly segment binds it.

```asm
alloc(codeCave, 0x100)
label(returnhere)

codeCave:
  jmp returnhere

returnhere:
```

### `registerSymbol`

```asm
registerSymbol(name)
```

Publishes a previously resolvable script label or allocation under `name` in the current script
context. It accepts exactly one argument. The target must be resolvable through a bound label or
allocation at the time the directive executes; symbol conflicts or unresolved names are errors.

Registration does not allocate memory or validate that a target is safe to execute. Keep the symbol
registered only while its backing label/allocation exists.

```asm
alloc(codeCave, 0x1000)
registerSymbol(codeCave)
```

### `unregisterSymbol`

```asm
unregisterSymbol(name)
```

Removes the symbol identified by `name` from the current script's published symbols. It accepts
exactly one argument. The implementation makes removal idempotent for a missing symbol, but invalid
argument count or symbol-storage failure is still an error.

```asm
unregisterSymbol(codeCave)
dealloc(codeCave)
```

The parser accepts the lower-case spellings `registersymbol` and `unregistersymbol` as well. Unknown
function-like lines are not silently treated as directives; they are passed to the assembler and
normally fail as invalid assembly.

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

# Auto Assembler API

Status: current public directive reference.

Auto Assembler is an action-oriented assembly runtime. The parser accepts directives and sends
assembly chunks to the current package script context. Auto Assembler is not a Lua global runtime;
`createThread` is an assembly directive, not a Lua function.

## Directive Reference

| Directive | Syntax | Behavior | Failure |
| --- | --- | --- | --- |
| `alloc` | `alloc(name, size?, nearAddress?)` | Allocates local script memory; default size is `0x1000` | Wrong argument count, invalid size/address, or allocation conflict fails with a line error |
| `globalalloc` | `globalalloc(name, size?, nearAddress?)` | Allocates session-global memory | Same allocation and name failures as `alloc` |
| `dealloc` | `dealloc(name)` | Releases a local or global allocation and linked names | Missing name or release failure fails with a line error |
| `aobScan` | `aobScan(name, pattern)` | Binds the first process-wide AOB match as a script label | Wrong count or no match fails |
| `aobScanModule` | `aobScanModule(name, module, pattern)` | Binds the first match in one module | Wrong count or no match fails |
| `aobScanRegion` | `aobScanRegion(name, start, stop, pattern)` | Binds the first match in an address range | Wrong count, invalid range, or no match fails |
| `fullAccess` | `fullAccess(address, size)` | Changes access for a target range | Invalid address/size or protection failure fails |
| `createThread` | `createThread(address)` | Executes code at a resolved target address | Wrong count, unresolved address, or execution failure fails |
| `label` | `label(name)` | Declares/resets a script label for later binding | Wrong count or invalid name fails |
| `registersymbol` | `registersymbol(name)` | Publishes a script label/allocation as a symbol | Wrong count or unresolved name fails |
| `unregistersymbol` | `unregistersymbol(name)` | Removes a script-owned symbol | Wrong count or symbol failure fails |

Directive names are parsed case-insensitively. User-defined names are validated by the engine.

## Allocation Example

```asm
alloc(exampleCave, 0x1000)
registersymbol(exampleCave)
```

Matching cleanup:

```asm
unregistersymbol(exampleCave)
dealloc(exampleCave)
```

`globalalloc` uses session-global allocation ownership and should be used only when the allocation
must outlive one script context. Do not use it to bypass package ownership.

## Scan Examples

```asm
aobScan(injection, 48 8B ?? ?? 89)
aobScanModule(moduleInjection, game.exe, 48 8B ?? ?? 89)
aobScanRegion(regionInjection, game.exe+0x1000, game.exe+0x9000, 48 8B ?? ?? 89)
```

Each scan binds the first match to a script label. Zero matches are errors. The current directive
implementation does not require uniqueness when multiple matches exist, so authors should choose a
pattern and scope that are known to produce one compatible result and verify it before distributing a
package.

## `createThread`

```asm
alloc(threadStart, 0x1000)

threadStart:
  ret

createThread(threadStart)
```

The directive resolves the entry address and invokes the session code-execution facility. The script
author owns the code at that address and must ensure it is valid for the target architecture and safe
to execute. The directive does not provide a managed thread handle, join operation, cancellation
protocol, or automatic allocation lifetime management.

## Hook Shape

```asm
alloc(newmem, 0x1000, injection)
label(returnhere)

newmem:
  ; preserve displaced instructions when required
  jmp returnhere

injection:
  jmp newmem
  nop
returnhere:
```

The engine does not calculate overwrite length, relocate displaced instructions, synthesize a full
trampoline, or restore the hook site automatically. Use a separate disable action to restore original
bytes and release all package-owned resources.

## About `assert`

`assert(...)` is not currently one of the parsed assembly directives. Do not document it as an
engine-supported directive or rely on it for precondition checking. Use a validated AOB result,
explicit address checks, and the package's supported error/cleanup path instead.

## Lifecycle

Auto Assembler action failure returns a line-specific error through the package command result. Pair
enable and disable actions, unregister symbols before deallocating their backing memory, and stop or
release any thread-related package resources before destroying the process context.

See [Assembly and Labels](assembly-and-labels.md), [Hooks and Auto Assembler](hooks-and-auto-assembler.md),
and [Errors and Results](errors-and-results.md).

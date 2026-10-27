# Assembly and Labels

Status: current public label and allocation model.

## Three Name Scopes

1. **Script labels** belong to a `ScriptContext` and include allocation-backed names and explicit
   labels. They live as long as the script context.
2. **Session symbols** are published explicitly and can be resolved at session scope.
3. **Assembler labels** exist only during one assembly pass, for example a local loop label.

An assembler-pass label does not automatically become a persistent script symbol. This prevents stale
names after disable and re-enable.

## Multi-Origin Scripts

A CE-style assembly script can contain allocations, scan results, labels, and address-bound chunks.
An expression ending in `:` starts a new chunk when it resolves to an address; otherwise it is an
internal assembler label.

```asm
aobScanModule(injection, game.exe, 48 8B ?? ?? 89)
alloc(newmem, 0x1000, injection)
label(returnhere)

newmem:
  nop
  jmp returnhere

injection:
  jmp newmem
  nop
returnhere:
```

## Ownership

Pair `alloc` with `dealloc`, and `registersymbol` with `unregistersymbol`. Package disable must
release allocations, symbols, patches, hooks, and timers it created. Do not consume another package's
private labels or allocations.

See [Visibility and Namespaces](visibility-and-namespaces.md) for the package and session scope of
these names.

## Forward offsets and imported symbols

Within one assembly chunk, `jmp later+0x2`, `jne later-0x2` and `call later+2`
resolve after `later:` binds. Offsets use instruction number rules: bare numbers
are decimal, and `0x` marks hexadecimal. The resulting aliases stay private to
that assembly pass. See the [Auto Assembler API](auto-assembler-api.md#names-with-offsets-far-jumps-and-quoted-modules).

An already visible registered symbol can be an address header (`flag:`) that
writes to that address. A consumer in another script context must declare its
provider dependency. Declaring a fresh unbound local label with the same name
does not import the provider's symbol.

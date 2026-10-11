# Patching

Status: current public patch behavior.

A named patch stores the original bytes in the current engine session and can restore them later.
The patch service is currently a native `EngineSession` API; it is not a registered Lua global.
Package authors using the package command path should use Auto Assembler or the supported package
runtime APIs instead of assuming these C++ methods are available in Lua.

## Byte Patch

```cpp
const auto record = session->applyPatch({
    .name = "player.hp",
    .address = hookAddress,
    .replacement = { std::byte{0x90}, std::byte{0x90}, std::byte{0x90}, std::byte{0x90} },
    .expected = { std::byte{0x89}, std::byte{0x54}, std::byte{0x24}, std::byte{0x10} },
});
```

For package scripts, write the replacement through an Auto Assembler action at the resolved site:

```asm
game.exe+0x1234:
  nop
  nop
  nop
  nop
```

This writes the same four NOP bytes, but does not create the native named patch record or
automatically preserve its precondition. `assert(...)` is not an AA directive. If an operation
requires an expected-byte check, use a source-qualified AOB/address check or the supported Lua
memory APIs before the write.

Restore the named patch from the native host:

```cpp
session->restorePatch("player.hp");
```

For a package, its explicit off action restores the original instruction. In this example the
original four bytes encode:

```asm
game.exe+0x1234:
  mov dword ptr [rsp+0x10], edx
```

The native patch service reads current bytes, verifies the optional precondition, temporarily changes page
protection when needed, writes replacement bytes, and stores the original bytes. NOP patches are a
common special case, but the same ownership and restore rules apply.

AA uses the AsmTK dialect. For raw replacement bytes, `.db 0x90, 0x90, 0x90, 0x90` also emits the
four NOPs; CE's unprefixed `db 90 90 90 90` is not equivalent syntax. See the
[assembler syntax notes](auto-assembler-api.md#instruction-and-data-syntax).

## Scope and Limitations

Patch records are session-scoped. They do not automatically install hooks, relocate instructions,
create trampolines, or make multi-write transactions. Assembly code written directly to a hook site
still requires the author to preserve displaced instructions, choose a safe overwrite length, and
restore the site on disable.

Use validated AOB results and target architecture checks before patching. Never patch a guessed or
ambiguous address.

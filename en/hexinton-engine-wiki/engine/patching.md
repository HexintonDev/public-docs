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

For package scripts, the equivalent lifecycle is expressed with an Auto Assembler action:

```asm
assert(injection, 89 54 24 10)
db 90 90 90 90
```

Restore the patch from the native host:

```cpp
session->restorePatch("player.hp");
```

Or restore the package-owned assembly site with its disable action.

```lua
local address = getAddressSafe("game.exe+0x1234")
if not address then error("patch site unavailable") end
```

The patch service reads current bytes, verifies the optional precondition, temporarily changes page
protection when needed, writes replacement bytes, and stores the original bytes. NOP patches are a
common special case, but the same ownership and restore rules apply.

## Scope and Limitations

Patch records are session-scoped. They do not automatically install hooks, relocate instructions,
create trampolines, or make multi-write transactions. Assembly code written directly to a hook site
still requires the author to preserve displaced instructions, choose a safe overwrite length, and
restore the site on disable.

Use validated AOB results and target architecture checks before patching. Never patch a guessed or
ambiguous address.

# Hexinton Engine Documentation

Status: public technical wiki framework and navigation draft.

This section is for script authors, tool integrators, and AI assistants that need to create or
explain HexMod engine scripts. The information architecture is defined in
[WIKI-FRAMEWORK.md](WIKI-FRAMEWORK.md). The pages below are topic drafts being reconciled against
the implementation and test fixtures.

## Start Here

Read the pages in this order:

1. [Wiki framework](WIKI-FRAMEWORK.md)
2. [Script packages and lifecycle](script-packages.md)
3. [Lua scripting](lua-scripting.md)
4. [Memory and address resolution](memory-and-addresses.md)
5. [AOB scanning](aob-scanning.md)
6. [Hooks and Auto Assembler](hooks-and-auto-assembler.md)
7. [Trainer controls and UI bindings](trainer-controls.md)
8. [Lua API reference](lua-api-reference.md)
9. [Testing and failure handling](testing.md)
10. [Sample library](samples/README.md)

## Capability Map

| Task | Page |
| --- | --- |
| Create a runnable script package | [Script packages and lifecycle](script-packages.md) |
| Read or write a game value | [Memory and address resolution](memory-and-addresses.md) |
| Find a stable address from bytes | [AOB scanning](aob-scanning.md) |
| Install a patch or hook | [Hooks and Auto Assembler](hooks-and-auto-assembler.md) |
| Configure trainer controls and custom surfaces | [Trainer controls and UI bindings](trainer-controls.md) |
| Find an API function | [Lua API reference](lua-api-reference.md) |
| Publish live values to a UI | [Lua scripting](lua-scripting.md) |
| Verify a script and diagnose errors | [Testing and failure handling](testing.md) |

## Related References

- [Package format and runnable declarations](script-packages.md)
- [Lua runtime API reference](lua-api-reference.md)
- [Trainer controls and UI bindings](trainer-controls.md)
- [Testing and failure handling](testing.md)

## Sample Library

The sample library is planned as a separate indexed section. Each sample will have a stable ID and
fixed GitHub source/test URLs so pages can link to examples without duplicating large code blocks.

## Important Boundary

Hexinton Engine executes Lua and Auto Assembler runnables. Application JavaScript is a
separate, capability-gated runtime owned by the desktop host. Do not use JavaScript APIs in a native
Lua runnable.
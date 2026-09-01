# Technical Wiki

Status: current public technical Wiki.

This section is for script authors, tool integrators, and AI assistants that need to create or explain HexMod engine scripts. The information architecture is defined in [WIKI-FRAMEWORK.md](wiki-framework.md). Current behavior is labeled and kept separate from planned or internal-only material.

## Start Here

Read the pages in this order:

1. [Wiki framework](wiki-framework.md)
2. [Script packages and lifecycle](script-packages.md)
3. [Lua scripting](lua-scripting.md)
4. [Visibility and namespaces](visibility-and-namespaces.md)
5. [Memory and address resolution](memory-and-addresses.md)
6. [AOB scanning](aob-scanning.md)
7. [Hooks and Auto Assembler](hooks-and-auto-assembler.md)
8. [Auto Assembler API](auto-assembler-api.md)
9. [Lua API reference](lua-api-reference.md)
10. [Thread model](thread-model.md)
11. [View Models and State Feeds](view-models-and-feeds.md)
12. [Application and Engine Architecture](application-and-engine-architecture.md)
13. [Testing and failure handling](testing.md)
14. [Sample library](samples/)

## Capability Map

| Task                                           | Page                                                     |
| ---------------------------------------------- | -------------------------------------------------------- |
| Create a runnable script package               | [Script packages and lifecycle](script-packages.md)      |
| Read or write a game value                     | [Memory and address resolution](memory-and-addresses.md) |
| Find a stable address from bytes               | [AOB scanning](aob-scanning.md)                          |
| Install a patch or hook                        | [Hooks and Auto Assembler](hooks-and-auto-assembler.md)  |
| Understand labels, symbols, and package visibility | [Visibility and Namespaces](visibility-and-namespaces.md) |
| Configure trainer controls and custom surfaces | [Trainer controls and UI bindings](trainer-controls.md)  |
| Find an API function                           | [Lua API reference](lua-api-reference.md)                |
| Use Auto Assembler directives                  | [Auto Assembler API](auto-assembler-api.md)              |
| Publish live values to a UI                    | [Lua scripting](lua-scripting.md)                        |
| Understand command, session, and feed boundaries | [Application and Engine Architecture](application-and-engine-architecture.md) |
| Understand workers, locks, and cleanup         | [Thread Model](thread-model.md)                          |
| Verify a script and diagnose errors            | [Testing and failure handling](testing.md)               |

## Related References

* [Package format and runnable declarations](script-packages.md)
* [Lua runtime API reference](lua-api-reference.md)
* [Lua process and address API](lua-process-and-address-api.md)
* [Lua memory API](lua-memory-api.md)
* [Lua scanning and assembly API](lua-scanning-and-assembly-api.md)
* [Lua runtime utilities API](lua-runtime-utilities-api.md)
* [Auto Assembler API](auto-assembler-api.md)
* [View Models and State Feeds](view-models-and-feeds.md)
* [Application integration](../application/README.md)
* [Trainer controls and UI bindings](trainer-controls.md)
* [Testing and failure handling](testing.md)

## Sample Library

The sample library is a separate indexed section. Each sample has a stable topic and is kept small enough to explain package lifecycle, Lua modules, process handles, service feeds, and Auto Assembler action pairs without duplicating the full API references.

## Important Boundary

Hexinton Engine executes Lua and Auto Assembler runnables. Application JavaScript is a separate, capability-gated runtime owned by the desktop host. Do not use JavaScript APIs in a native Lua runnable.

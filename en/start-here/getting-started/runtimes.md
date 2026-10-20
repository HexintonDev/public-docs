# Runtime Selection

Status: current public runtime guide. UE4SS describes the implemented integration contract; client/framework availability is stated in its guide.

| Runtime        | Runs in                   | Use it for                                                            |
| -------------- | ------------------------- | --------------------------------------------------------------------- |
| Lua            | Native engine worker      | Attached process memory, scans, symbols, timers, and services         |
| UE4SS Lua (`ue4ss`) | Inside the attached Unreal game | Reflected objects/properties/functions, managed hooks and game-thread services |
| Auto Assembler | Native engine assembly path | Assembly patches, allocations, and hooks                              |
| JavaScript     | C# host through Jint      | File/window/process discovery, launch, and approved host capabilities |

## Rules

Choose from the operation's behavior rather than the source provider or file extension. A fixed
instruction replacement may fit two short AA operations; a pointer/value edit or conditional logic
may fit Lua; a hook that captures an object pointer for a parameterized action may use both. Reuse
the engine's symbols, allocations and typed memory APIs instead of constructing a second runtime.

Use UE4SS Lua for Unreal object/function access; it requires an attached PID and the UE4SS framework installed with the client.
Use application JavaScript for capability-gated host-side work. Use a service plus a ViewModel/feed
when the interface needs continuously observed game state. A one-time patch does not require a
polling service.

AA supports `enable`, `disable` and `action` entries, with `entryFile` and no `entrySymbol`.
It cannot return a query value, start a service, or consume action arguments. Lua and UE4SS can
provide those operations. Package preparation may happen before an initial query, so put a
user-selected gameplay change in an action, with explicit lifecycle cleanup. See
[package lifecycle](../../hexinton-engine-wiki/engine/script-packages.md) and
[trainer bindings](../../hexinton-engine-wiki/engine/trainer-controls.md).

Host Lua and UE4SS Lua use separate interpreters and APIs. Keep existing Lua logic and call a declared UE provider through [dependency runnables](../../hexinton-engine-wiki/engine/dependency-runnable-api.md) when it needs Unreal data or an operation. You do not need to duplicate the package dependency/lifecycle rules or create another UI command format. Files remain `.lua`; the manifest runtime selects the interpreter.

UE4SS loads on demand from the framework installed with the client. Read [UE4SS loading prerequisites](../../hexinton-engine-wiki/engine/ue4ss-runtime.md#loading-and-prerequisites) before trying the [location sample](../../hexinton-engine-wiki/engine/samples/ue4ss-player-location.md).

Application JavaScript is not the same as a WebView surface script. Both are host-side concerns, but the surface uses declared bindings and the application JavaScript runtime uses the capability-gated `hex` object. Neither receives unrestricted Lua or CLR access.

Auto Assembler labels and allocations must have an explicit lifecycle. A successful command is one operation result; ongoing state must come from a service-backed feed.

HexMod AA resembles CE but uses the AsmTK instruction/data dialect. Check the
[syntax differences](../../hexinton-engine-wiki/engine/auto-assembler-api.md#instruction-and-data-syntax)
before copying a script, and follow the [Auto Assembler rules](../../hexinton-engine-wiki/engine/engine-essentials.md#auto-assembler-rules). Static validation does not assemble AA; exercise the actual operation in
a controlled integration target and distinguish that result from real-game compatibility.

See [Application JavaScript capabilities](../../hexinton-engine-wiki/application/application-js-host-capabilities.md), [Lua API reference](../../hexinton-engine-wiki/engine/lua-api-reference.md), and [Auto Assembler examples](../../hexinton-engine-wiki/engine/assembly-api-examples.md).

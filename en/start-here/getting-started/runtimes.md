# Runtime Selection

Status: current public runtime guide. UE4SS describes the implemented integration contract; client/framework availability is stated in its guide.

| Runtime        | Runs in                   | Use it for                                                            |
| -------------- | ------------------------- | --------------------------------------------------------------------- |
| Lua            | Native engine worker      | Attached process memory, scans, symbols, timers, and services         |
| UE4SS Lua (`ue4ss`) | Inside the attached Unreal game | Reflected objects/properties/functions, managed hooks and game-thread services |
| Auto Assembler | Native engine action path | Assembly patches, allocations, and hooks                              |
| JavaScript     | C# host through Jint      | File/window/process discovery, launch, and approved host capabilities |

## Rules

Use host Lua for process memory and address-based logic. Use UE4SS Lua for Unreal object/function access; it also requires an attached PID and prepared framework. Use Auto Assembler for assembly actions. Use application JavaScript for host-side work. Use a service plus a ViewModel/feed for continuously displayed game state.

Host Lua and UE4SS Lua use separate interpreters and APIs. Keep existing Lua logic and call a declared UE provider through [dependency runnables](../../hexinton-engine-wiki/engine/dependency-runnable-api.md) when it needs Unreal data or an operation. You do not need to duplicate the package dependency/lifecycle rules or create another UI command format. Files remain `.lua`; the manifest runtime selects the interpreter.

UE4SS loads on demand, but automatic application framework delivery is incomplete. Read [UE4SS loading prerequisites](../../hexinton-engine-wiki/engine/ue4ss-runtime.md#loading-and-prerequisites) before trying the [location sample](../../hexinton-engine-wiki/engine/samples/ue4ss-player-location.md).

Application JavaScript is not the same as a WebView surface script. Both are host-side concerns, but the surface uses declared bindings and the application JavaScript runtime uses the capability-gated `hex` object. Neither receives unrestricted Lua or CLR access.

Auto Assembler labels and allocations must have an explicit lifecycle. A successful command is one operation result; ongoing state must come from a service-backed feed.

See [Application JavaScript capabilities](../../hexinton-engine-wiki/application/application-js-host-capabilities.md), [Lua API reference](../../hexinton-engine-wiki/engine/lua-api-reference.md), and [Auto Assembler examples](../../hexinton-engine-wiki/engine/assembly-api-examples.md).

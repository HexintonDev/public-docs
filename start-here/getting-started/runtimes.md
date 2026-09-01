# Runtime Selection

Status: current public runtime guide.

| Runtime        | Runs in                   | Use it for                                                            |
| -------------- | ------------------------- | --------------------------------------------------------------------- |
| Lua            | Native engine worker      | Attached process memory, scans, symbols, timers, and services         |
| Auto Assembler | Native engine action path | Assembly patches, allocations, and hooks                              |
| JavaScript     | C# host through Jint      | File/window/process discovery, launch, and approved host capabilities |

## Rules

Use Lua when a pid and native session are required. Use Auto Assembler when the operation is an assembly action. Use application JavaScript for host-side work that does not require native memory access. Use a Lua service plus a ViewModel/feed for changing game state.

Application JavaScript is not the same as a WebView surface script. Both are host-side concerns, but the surface uses declared bindings and the application JavaScript runtime uses the capability-gated `hex` object. Neither receives unrestricted Lua or CLR access.

Auto Assembler labels and allocations must have an explicit lifecycle. A successful command is one operation result; ongoing state must come from a service-backed feed.

See [Application JavaScript capabilities](../../hexinton-engine-wiki/application/application-js-host-capabilities.md), [Lua API reference](../../hexinton-engine-wiki/engine/lua-api-reference.md), and [Auto Assembler examples](../../hexinton-engine-wiki/engine/assembly-api-examples.md).

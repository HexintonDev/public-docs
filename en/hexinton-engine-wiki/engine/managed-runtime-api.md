# Managed Runtime API

Status: implemented host-Lua API, reviewed against the engine on 2026-11-02.

Use runtime metadata to resolve managed classes, methods and fields in Unity Mono, Unity IL2CPP,
CoreCLR and .NET Framework CLR v4 games. These calls use the package's attached process; the client
owns process attachment. They do not require a whole-process byte scan.

## Attach and inspect the runtime

```lua
local rt = managed.attach()
-- rt.kind: "mono", "il2cpp", "coreclr" or "clr"
-- rt.module: the detected runtime module name
```

Attach detects the loaded runtime and prepares the appropriate adapter. A failed attach is not
cached, so a later explicit attempt can retry after the game has initialized. For .NET, the engine
loads its helper into the already-running runtime. CLR hosting supports CLR v4; the CoreCLR path
requires the game's original `hostfxr.dll` to remain loaded. A detected runtime can therefore still
fail with `managed_runtime_not_ready`. Detection alone does not prove that a game is supported.

Class paths are `[Assembly::]Namespace.Class[/Nested]`. Specify the assembly when an unqualified
type is ambiguous. Constructed generic types use names such as ``Game.Box`1[System.Int32]`` (the
backtick and arity are part of the type name).

## Class and method lookup

```lua
local playerClass = rt:class('Game.Player')
local method = rt:method('Game.Player', 'Heal', {'System.Int32'})
local signatures = rt:methods('Game.Player', 'Heal')
```

| Operation | Result |
| --- | --- |
| `rt:class(classPath)` | Runtime class/type handle |
| `rt:methods(classPath, name)` | Array of overload signature strings |
| `rt:method(classPath, name, options?)` | `{address, handle, signature, size?}` |

Method options may be a parameter count, a list of exact parameter type names, or an options table:

```lua
local heal = rt:method('Game.Player', 'Heal', 1)
local generic = rt:method('Game.Player', 'Echo', {
    params = {'System.Int32'},
    generic = {'System.Int32'}
})
```

`count` can also be supplied in the options table. If multiple overloads remain, lookup raises
`ambiguous_overload`; use parameter types rather than selecting an arbitrary method. Lookup searches
the named class and its nearest declaring parent. A concrete .NET method can be compiled on demand
with the runtime's preparation API; the game need not have called it first. Abstract methods and
open generic methods have no concrete native entry point.

## Entry points, body offsets and CE symbols

`address` is the runtime's native **entry point**. On CoreCLR it can be a precode jump stub rather
than the compiled body. `size` is optional: the Mono adapter can obtain it when JIT metadata is
available; the current CLR and CoreCLR adapters do not return a code size.

Do not apply a source's body-relative patch offset to the entry point, guess a method length, or
scan an invented range. Body patches require separately verified native body/range information and
expected bytes for the supported game build. Tiered compilation can replace generated code.

Ordinary address lookup does not automatically parse CE managed names such as
`Terraria.Main::get_LocalPlayer+10`. Resolve the method explicitly instead:

```lua
local rt = managed.attach()
local getter = rt:method('Terraria.Main', 'get_LocalPlayer', 0)
-- getter.address is an entry point, not proof that getter.address + 0x10 is the source patch site.
```

Resolving an address does not provide general managed invocation. The API has no `rt:invoke` or
managed-string constructor. Native `executeCode`/`executeCodeEx` do not automatically marshal managed
objects, attach runtime threads, establish a managed calling convention, or handle managed
exceptions. Qualify those requirements separately before attempting a game-specific call.

## Fields and static storage

```lua
local health = rt:field('Game.Player', 'Health')
-- health: {name, type, static, offset, address?}
local countAddress = rt:static('Game.Player', 'Count')
```

Instance `offset` is measured from the beginning of the managed object, including its header.
Obtain and validate the actual object pointer separately. A class handle is not an object pointer;
the garbage collector may move objects. Lookup does not pin arbitrary game objects.

`rt:static` requires a static field and returns its storage address. For .NET the helper exposes
stable primitive, enum and pointer-sized scalar static storage. Literal constants, thread/context
statics, reference statics and other storage that cannot be safely exposed fail with
`field_has_no_storage`. This is not a promise to expose every field type or movable boxed static.
Runtime-specific initialization failures remain errors.

## Errors and lifetime

Failures raise catchable Lua errors containing a stable `[code]` prefix. Common codes include
`managed_runtime_not_found`, `managed_runtime_not_ready`, `managed_export_missing`,
`class_not_found`, `method_not_found`, `ambiguous_overload`, `method_not_compiled`,
`field_not_found`, `field_has_no_storage`, `field_not_static`, `class_not_initialized`,
`generic_instance_failed` and `invalid_type_name`. Preserve unfamiliar failures; do not substitute
an address from an unrelated scan.

Runtime handles and lookup caches belong to the Lua runtime's attached process. Do not persist them
across detach, restart or a different process. Resolution and test-fixture success do not establish
that a live-game patch or call is compatible. See [testing](testing.md),
[address resolution](lua-process-and-address-api.md) and [game calls](lua-game-calls-and-worker-api.md).

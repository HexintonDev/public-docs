# Visibility and Namespaces

Status: current public runtime behavior.

This page defines what is visible between Lua files, packages, dependencies, Auto Assembler passes,
and process sessions. Visibility is an ownership rule, not a convenience global namespace.

## The Main Scopes

| Scope | Owned by | Visible to | Lifetime |
| --- | --- | --- | --- |
| Lua local | Current Lua chunk/function | That lexical scope | Normal Lua lifetime |
| Lua script environment | One package `scriptId` | That package's Lua runnables | Current script context |
| Dependency-visible Lua global | A declared dependency package | The dependent package and its declared dependents | While the dependency context exists |
| Script label/allocation | One `ScriptContext` | That script context and explicit resolution callers | Script context lifetime |
| Session symbol | `EngineSession` symbol repository | Session-level address resolution | Current process session |
| Assembly-pass label | One assembler pass | Chunks in that pass | Until that pass ends |

An unrelated package cannot see another package's Lua globals, labels, allocations, timers, hooks, or
private symbols merely because both packages are enabled in the same process context.

## Lua Globals

A global assignment belongs to the current package environment:

```lua
counter = (counter or 0) + 1

function readCounter()
    return counter
end
```

The value persists across runnables for the same package/script context. It is not process-wide and it
is not visible to an unrelated package. `local` keeps normal Lua lexical scope. `_G.value` also refers
to the current script environment; it does not publish a process-wide global.

The current package's globals win during dependency lookup. Globals from declared transitive
dependencies are considered next. If two visible dependencies provide the same global name, lookup
fails as ambiguous. If a declared dependency context is missing, lookup fails with
`dependency_context_missing`.

## Package-Local `require`

A hosted Lua package receives a package-local `require` loader. For this package:

```text
<scriptsRoot>/trainer.player/lua/helpers/inventory.lua
```

this import is valid:

```lua
local inventory = require("helpers.inventory")
return inventory.read(1)
```

Module IDs use dot-separated segments. Segments may contain letters, numbers, `_`, and `-`. The
loader rejects absolute paths, path separators, empty segments, `..`, and a `.lua` suffix.

Required files execute in the same script environment as the entry file. Their module return values
are cached once per `ScriptContext`. A module that returns no value caches and returns `true`.

Typical layout:

```text
trainer.player/
  package.json
  lua/
    main.lua
    helpers/inventory.lua
```

```lua
-- lua/helpers/inventory.lua
local module = {}

function module.read(page)
    return { page = page }
end

return module
```

```lua
-- lua/main.lua
local inventory = require("helpers.inventory")

function getInventory(arguments)
    return inventory.read(arguments.page)
end
```

Current module failures are `lua_module_invalid_id`, `lua_module_not_found`, and
`lua_module_cycle`. A Lua load/runtime error includes the helper path in its traceback.

Cross-package imports do not use `require`. A package consumes dependency globals through the enabled
dependency chain declared in `package.json`.

`runGlobal` is a host-level Lua operation and uses the normal Lua global loader. It is separate from a
package runnable and does not establish package-local visibility.

## Dependency Visibility

Declare a dependency explicitly:

```json
{
  "dependencies": [
    { "id": "trainer.shared", "compatible": "^1.0.0", "tested": "1.0.2" }
  ]
}
```

The host creates dependency context IDs and enables dependencies before the dependent package. The
dependent package may use the dependency's declared public Lua globals, but not its implementation
state. Keep dependency contracts stable and avoid relying on accidental names.

## Symbols and Labels

### Script labels

`ScriptContext` owns persistent labels and local allocations. An allocation such as `newmem` creates
a same-name local label. Explicit labels can be declared and bound by the script context. They are
not automatically visible to unrelated packages.

### Session symbols

A package may explicitly publish a symbol with `registerSymbol`. Session-level resolution can then
find it, subject to the package and dependency visibility rules. Pair it with `unregisterSymbol` in
cleanup.

### Assembly-pass labels

A label inside one text assembly pass, such as a loop label, is private to that pass:

```asm
loop:
  dec ecx
  jne loop
```

It does not become a persistent `ScriptContext` label automatically. Use explicit script labels or
registered symbols when a name must survive the assembly pass.

## Auto Assembler Allocation Names

`alloc(name, size)` creates an allocation-backed name in the owning script context. `label(name)`
defines an explicit script label. `registersymbol(name)` publishes a name, and
`unregistersymbol(name)` removes that publication. Pair `alloc` with `dealloc` and symbols with
`unregistersymbol` during the package disable path.

Allocation and label names are not a substitute for a public package API. A dependent package should
consume documented public values, not reach into another package's private code cave or hook.

## Process and Session Scope

Bare Lua memory/address APIs use the command's default process session. `openProcess(pid)` returns an
explicit handle for another process session. Script environments and script-local labels remain
scoped to the runtime/script context; they do not become shared merely because the same package ID
is used with a different pid.

A new process attachment has a new native context and new lifecycle-owned runtime state. Package
cleanup must release names and resources before the context is destroyed.

## Visibility Checklist

Before publishing a package contract, verify:

- package-local modules use `require` only within the package;
- every cross-package dependency is declared;
- public globals and symbols are documented;
- private labels, allocations, timers, and hooks are not consumed externally;
- assembly-pass labels are not assumed to survive;
- symbols and allocations have disable cleanup;
- a different pid is treated as a different process/session context.

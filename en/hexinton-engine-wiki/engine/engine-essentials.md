# Engine Essentials

Status: current behavior of the shipped engine. This page is the compact set of rules for writing
HexMod packages, for people and for coding agents. Every complete example under
[Verified examples](#verified-examples) is assembled and run against the engine's controlled test
target by an automated check. Details live in the linked reference pages.

## How a package runs

A player selects a game, the client attaches to the running game process, and the player uses
toggles, buttons and value fields in a trainer panel. Each control calls a **runnable** in a
**package**: a folder with `package.json`, Lua and/or Auto Assembler (AA) files and a hosted UI
declaration. The engine runs that runnable against the attached process.

Packages implement gameplay operations only. They never find or attach to a process, launch the
game, open their own windows, or manage sessions.

## Package contract

- `package.json` declares `runtime.files`, `runtime.runnables`, optional dependencies and
  `frontend.hosted`. The folder name equals `id`. Every package has an `enable` and a `disable`
  runnable.
- Runnable kinds are `enable`, `disable`, `action`, `query` and `service`. AA supports `enable`,
  `disable` and `action` with `entryFile` only: no `entrySymbol`, parameters or return values. Lua
  and UE4SS support every kind and name a function with `entrySymbol`.
- The engine may run `enable` before the first query or action, so `enable` only prepares. The
  player's change happens in an `action`. `disable` cleans up and must work even when the feature
  was never turned on.
- A hosted toggle uses `write.enableAction` / `write.disableAction` (action IDs). A number widget
  uses `write.action` plus `valueParameter`. See [trainer controls](trainer-controls.md).
- Lua action return values are discarded. An action fails only by raising `error(...)`.

## Choosing a runtime

| Behavior | Usually fits |
| --- | --- |
| Fixed byte patch, AOB patch, code-cave hook | AA files, one per operation |
| Hook whose "off" must not run before "on", or a hook plus a flag | AA text run from a short Lua file that checks the hook site's bytes |
| Pointer or value write, calculation, condition | Host Lua |
| Repeating write or check, frozen value | Host Lua repeating timer |
| Hook captures a pointer that a parameterized action uses | AA hook plus Lua action |
| Unreal reflection | `ue4ss` runtime, when its prerequisites are available |

Pick the smallest fit. A five-byte patch is two tiny AA files and a toggle, not a Lua framework.
See [runtime selection](../../start-here/getting-started/runtimes.md).

## Auto Assembler rules

HexMod AA is its own dialect: AsmTK instruction syntax plus a small set of engine directives. Follow
these rules exactly. Text written for other assemblers does not run here unchanged.

### Numbers: always write `0x`

The engine reads a bare number differently depending on where it appears:

| Position | Bare `1000` means | Example |
| --- | --- | --- |
| Instructions and data (`mov`, `cmp`, `[reg+offset]`, `.db` ...) | **decimal** 1000 | `[rcx+170]` is offset 170 (`0xAA`) |
| Address headers and directive arguments (`game.exe+1234:`, `alloc(...)`, `fullAccess(...)` ...) | **hexadecimal** `0x1000` | `alloc(cave, 1000)` reserves `0x1000` bytes |

Write every number of 10 or more with a `0x` prefix, in every position. Then both readings agree.
Numbers 0 to 9 are the same either way. Package validation reports a bare number of 10 or more in an
`.aa` file as the warning `aa_ambiguous_number`, with what the engine actually read. AOB scan
patterns are byte pairs (`48 8B ?? 05`) and are not affected.

### Data

- `.db`, `.dw`, `.dd` and `.dq` emit little-endian 1-, 2-, 4- and 8-byte values, comma-separated:
  `.db 0x90, 0x90, 0x90`. Undotted `db` and GNU `.byte` are not accepted.
- Write floats as their IEEE-754 bit pattern: `1.0` is `.dd 0x3F800000`, `100.0` is
  `.dd 0x42C80000`, `-2000.0` is `.dd 0xC4FA0000`.

### Directives

The engine handles `alloc(name[, size[, near]])`, `globalAlloc`, `dealloc(name)`, `label(name)`,
`registerSymbol(name)`, `unregisterSymbol(name)`, `aobScan`, `aobScanModule`, `aobScanRegion`,
`fullAccess` and `createThread`. Names are case-insensitive. Signatures and failure behavior are in
the [Auto Assembler API](auto-assembler-api.md).

Not part of HexMod AA: `define`, `readmem`, `assert`, `{$lua}`/`{$asm}` blocks, `{$try}`,
`reassemble`, `loadlibrary`, `struct`, anonymous `@@` labels and `[ENABLE]`/`[DISABLE]` sections.
Instead:

- one AA file performs one operation, so "on" and "off" are separate files;
- name an address with a header or a label bound at it;
- write known original bytes with `.db`;
- put conditions and checks in a Lua runnable, which can run AA text with `autoAssemble`.

### Addresses and labels

- Module expressions such as `game.exe+0x1234:` work as chunk headers, not as instruction operands.
  To `call` or `jmp` to a module address, bind a label there:

  ```asm
  label(originalCall)
  game.exe+0x4801e0:
  originalCall:

  game.exe+0x4948ae:
    call originalCall
  ```

  This writes only the `call`; nothing is written at `originalCall`.
- Declare `label(name)` only for names that connect separate chunks, such as a return address or a
  hook site the "off" file needs. Leave labels used inside one chunk, such as inline data after cave
  code, undeclared.
- Allocation and symbol names persist between a package's operations, so "off" can refer to what
  "on" created. That is all the state most hooks need.
- A bound label stays resolvable after `unregisterSymbol` and `dealloc`, so `getAddressSafe(name)`
  cannot tell whether a hook is installed. Read the hook site's bytes instead, as the hook example
  below does.

### Hooks and restoration

The engine does not compute overwrite length, relocate displaced instructions or save original
bytes. Keep the jump, padding `nop`s, displaced instructions and return label exactly as the design
requires. The "off" operation writes the original bytes back and releases the allocation and
symbols. `dealloc` alone does not restore the hook site.

## Lua rules

- `autoAssemble(text)` returns `true`, or `false, message`. Check it and raise the message.
- `getAddress("[[game.exe+0x10]+0x20]+0x4")` resolves a pointer chain and raises when a hop is
  unreadable. `getAddressSafe` returns `nil` instead.
- `createTimer(false)` makes a disabled repeating timer: set `Interval` and `OnTimer`, then
  `Enabled = true`. Destroy timers in the "off" action and in `disable`.
- The host Lua API is the registered set in the [Lua API reference](lua-api-reference.md). There are
  no UI forms, address lists, file-version or module-enumeration functions. Do not assume a function
  exists because another tool has one.

## Game versions

Packages are not version-gated. Record the game build a package was written or tested for in its
`docs/` notes. Do not refuse an operation because the running build cannot be identified, and do not
add a version reader. A package may still check something it directly depends on, such as the
original bytes at a patch site before writing. A wrong address shows up as a failed scan, a failed
action or no effect, and is fixed by updating the package.

## Testing

Static validation checks the manifest, bindings and Lua syntax; it does not assemble AA. One
integration case is usually enough: run the production "on" operation against the controlled test
target, check the bytes or value it wrote, run "off" and check the original state is back. Add unit
cases only for real logic. See [testing](testing.md) and [test helpers](package-test-helpers.md).

To land a module-relative address in the test target, register the module name as a symbol in the
test before assembling. Never change the package's real offsets to fit the test target. A passing
test proves the operation assembles and writes what it should; it does not prove the address matches
a given game build.

## Verified examples

Each example below is a complete package. The automated check writes these files and runs the
declared integration test on the engine. Addresses are illustrative.

### Example: fixed patch

"On" replaces a five-byte call with `nop`s; "off" writes the original call back. The call's
displacement, `0x4801e0 - (0x4948ae + 5)`, does not depend on where the module is loaded.

{% code title="package.json" %}
```json
{
  "id": "example.fixed-patch", "version": "1.0.0", "displayName": "Fixed patch",
  "runtime": {
    "files": [
      { "path": "aa/prepare.aa", "runtime": "aa" },
      { "path": "aa/on.aa", "runtime": "aa" },
      { "path": "aa/off.aa", "runtime": "aa" }
    ],
    "runnables": [
      { "id": "prepare", "kind": "enable", "runtime": "aa", "entryFile": "aa/prepare.aa" },
      { "id": "cleanup", "kind": "disable", "runtime": "aa", "entryFile": "aa/off.aa" },
      { "id": "on", "kind": "action", "runtime": "aa", "entryFile": "aa/on.aa" },
      { "id": "off", "kind": "action", "runtime": "aa", "entryFile": "aa/off.aa" }
    ]
  },
  "frontend": { "hosted": { "schemaVersion": 1, "groups": [{
    "id": "player", "label": "Player", "widgets": [{
      "id": "fixed-patch", "kind": "toggle", "label": "Fixed patch",
      "write": { "enableAction": "on", "disableAction": "off" }
    }]
  }]}}
}
```
{% endcode %}

{% code title="aa/prepare.aa" %}
```asm
// Nothing to prepare; "off" only rewrites fixed bytes, so lifecycle disable can reuse it.
```
{% endcode %}

{% code title="aa/on.aa" %}
```asm
Game.exe+0x4948ae:
  .db 0x90, 0x90, 0x90, 0x90, 0x90
```
{% endcode %}

{% code title="aa/off.aa" %}
```asm
Game.exe+0x4948ae:
  .db 0xE8, 0x2D, 0xB9, 0xFE, 0xFF
```
{% endcode %}

{% code title="tests/integration.lua" %}
```lua
return function(ctx)
  local h = dofile(ctx.helpers.luaFile)
  local site = ctx.fixture.baseAddress + 192
  local function run(path)
    local file = assert(io.open(ctx.packageRoot .. path, "rb"))
    local text = file:read("*a"); file:close()
    local ok, message = autoAssemble(text); assert(ok, message)
  end
  h.withMemory({ { address = site, size = 5 } }, function()
    registerSymbol("Game.exe", site - 0x4948ae) -- test-only address substitution
    run("/aa/on.aa")
    for _, byte in ipairs(readBytes(site, 5, true)) do h.equal(byte, 0x90) end
    run("/aa/off.aa")
    h.equal(readBytes(site, 1), 0xE8)
    h.equal(site + 5 + readInteger(site + 1, true), getAddress("Game.exe+0x4801e0"))
  end)
end
```
{% endcode %}

{% code title="tests/studio.tests.json" %}
```json
{ "schemaVersion": 1, "cases": [
  { "id": "on-off", "kind": "integration", "runtime": "lua", "entryFile": "tests/integration.lua", "timeoutMs": 15000 }
] }
```
{% endcode %}

### Example: hook with inline data

"On" allocates a cave near the hook site, replaces a seven-byte `movaps xmm6, [rcx+0x170]` with a
jump, loads a constant vector into `xmm6` in the cave and jumps back. "Off" writes the original
seven bytes back and releases the cave. Because "off" needs the cave from "on", a short Lua file
runs both AA texts and decides from the hook site's first byte (`0xE9`, the `jmp`) whether the hook
is installed, so the toggle works for any number of cycles and "off" is safe before "on". `Spoof`
stays undeclared because only the cave chunk uses it.

{% code title="package.json" %}
```json
{
  "id": "example.hook-inline-data", "version": "1.0.0", "displayName": "Hook with inline data",
  "runtime": {
    "files": [{ "path": "lua/main.lua", "runtime": "lua" }],
    "runnables": [
      { "id": "enable", "kind": "enable", "runtime": "lua", "entryFile": "lua/main.lua", "entrySymbol": "enable" },
      { "id": "disable", "kind": "disable", "runtime": "lua", "entryFile": "lua/main.lua", "entrySymbol": "disable" },
      { "id": "on", "kind": "action", "runtime": "lua", "entryFile": "lua/main.lua", "entrySymbol": "turnOn" },
      { "id": "off", "kind": "action", "runtime": "lua", "entryFile": "lua/main.lua", "entrySymbol": "turnOff" }
    ]
  },
  "frontend": { "hosted": { "schemaVersion": 1, "groups": [{
    "id": "player", "label": "Player", "widgets": [{
      "id": "hook", "kind": "toggle", "label": "Hook with inline data",
      "write": { "enableAction": "on", "disableAction": "off" }
    }]
  }]}}
}
```
{% endcode %}

{% code title="lua/main.lua" %}
```lua
local ON = [[
label(HookSite)
Game.exe+0x9C83CA:
HookSite:
alloc(cave, 0x1000, HookSite)
label(returnHere)
registerSymbol(HookSite)
cave:
  movups xmm6, [Spoof]
  jmp returnHere
Spoof:
  .dd 0, 0xC4FA0000, 0, 0x3F800000
HookSite:
  jmp cave
  nop
  nop
returnHere:
]]

local OFF = [[
HookSite:
  .db 0x0F, 0x28, 0xB1, 0x70, 0x01, 0x00, 0x00
unregisterSymbol(HookSite)
dealloc(cave)
]]

local function assemble(text)
  local ok, message = autoAssemble(text)
  if not ok then error(message) end
end

local function installed()
  return readBytes(getAddress("Game.exe+0x9C83CA"), 1) == 0xE9
end

function enable() end
function turnOn() if not installed() then assemble(ON) end end
function turnOff() if installed() then assemble(OFF) end end
function disable() turnOff() end
```
{% endcode %}

{% code title="tests/integration.lua" %}
```lua
return function(ctx)
  local h = dofile(ctx.helpers.luaFile)
  dofile(ctx.packageRoot .. "/lua/main.lua")
  local site = ctx.fixture.baseAddress + 200
  local original = { 0x0F, 0x28, 0xB1, 0x70, 0x01, 0x00, 0x00 }
  h.withMemory({ { address = site, size = 7 } }, function()
    registerSymbol("Game.exe", site - 0x9C83CA) -- test-only address substitution
    writeBytes(site, original)
    turnOff() -- safe before "on"
    for _ = 1, 2 do
      turnOn()
      turnOn() -- already on: no second hook
      h.equal(readBytes(site, 1), 0xE9)
      turnOff()
      for index, byte in ipairs(readBytes(site, 7, true)) do h.equal(byte, original[index]) end
    end
  end)
end
```
{% endcode %}

{% code title="tests/studio.tests.json" %}
```json
{ "schemaVersion": 1, "cases": [
  { "id": "on-off", "kind": "integration", "runtime": "lua", "entryFile": "tests/integration.lua", "timeoutMs": 15000 }
] }
```
{% endcode %}

### Example: pointer write

"On" writes a value through a pointer chain; "off" writes the normal value. `getAddress` raises if a
pointer in the chain is unreadable, which fails the action as it should.

{% code title="package.json" %}
```json
{
  "id": "example.pointer-write", "version": "1.0.0", "displayName": "Pointer write",
  "runtime": {
    "files": [{ "path": "lua/main.lua", "runtime": "lua" }],
    "runnables": [
      { "id": "enable", "kind": "enable", "runtime": "lua", "entryFile": "lua/main.lua", "entrySymbol": "enable" },
      { "id": "disable", "kind": "disable", "runtime": "lua", "entryFile": "lua/main.lua", "entrySymbol": "disable" },
      { "id": "on", "kind": "action", "runtime": "lua", "entryFile": "lua/main.lua", "entrySymbol": "turnOn" },
      { "id": "off", "kind": "action", "runtime": "lua", "entryFile": "lua/main.lua", "entrySymbol": "turnOff" }
    ]
  },
  "frontend": { "hosted": { "schemaVersion": 1, "groups": [{
    "id": "player", "label": "Player", "widgets": [{
      "id": "pointer", "kind": "toggle", "label": "Pointer write",
      "write": { "enableAction": "on", "disableAction": "off" }
    }]
  }]}}
}
```
{% endcode %}

{% code title="lua/main.lua" %}
```lua
local TARGET = "[[Game.exe+0x4768E78]+0x20]+0x4"

function enable() end
function turnOn() writeInteger(getAddress(TARGET), 999) end
function turnOff() writeInteger(getAddress(TARGET), 100) end
function disable() end
```
{% endcode %}

{% code title="tests/integration.lua" %}
```lua
return function(ctx)
  local h = dofile(ctx.helpers.luaFile)
  dofile(ctx.packageRoot .. "/lua/main.lua")
  -- The test target holds a pointer to the player at signature+16 and to stats at player+0x20.
  registerSymbol("Game.exe", ctx.fixture.signatureAddress + 16 - 0x4768E78) -- test-only substitution
  local value = ctx.fixture.statsAddress + 4
  h.withMemory({ { address = value, size = 4 } }, function()
    turnOn()
    h.equal(readInteger(value), 999)
    turnOff()
    h.equal(readInteger(value), 100)
  end)
end
```
{% endcode %}

{% code title="tests/studio.tests.json" %}
```json
{ "schemaVersion": 1, "cases": [
  { "id": "on-off", "kind": "integration", "runtime": "lua", "entryFile": "tests/integration.lua", "timeoutMs": 15000 }
] }
```
{% endcode %}

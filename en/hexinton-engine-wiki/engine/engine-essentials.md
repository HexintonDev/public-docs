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
  uses `write.action` plus `valueParameter`. Lua actions receive one JSON parameter object, for
  example `function addCurrency(params)` reads `params.amount`; they do not receive positional
  amounts, addresses or scan strings. See [trainer controls](trainer-controls.md).
- Lua action return values are discarded. An action fails only by raising `error(...)`.

## Choosing a runtime

| Behavior | Usually fits |
| --- | --- |
| Fixed byte patch, AOB patch, code-cave hook | AA files, one per operation |
| Prepared hook with a flag initially zero; actions only write fixed flag values | AA prepare/on/off/cleanup files; prepare keeps the gameplay effect inactive |
| Hook whose "off" needs resources created by "on", or an operation needing an explicit condition | Short Lua orchestration when the chosen AA lifecycle cannot express that behavior |
| Pointer or value write, calculation, condition | Host Lua |
| Repeating write or check, frozen value | Host Lua repeating timer |
| Hook captures a pointer that a parameterized action uses | AA hook plus Lua action |
| Unreal reflection | `ue4ss` runtime, when its prerequisites are available |

Pick the smallest fit. A five-byte patch is two tiny AA files and a toggle, not a Lua framework.
See [runtime selection](../../start-here/getting-started/runtimes.md).

Reuse a shared helper when several features need the same table lookup, item spawning or restoration
logic. Implement the needed operations once and keep the feature actions small. Large item or
destination lists are data: extract and preserve their IDs and names instead of rewriting them as
code. Use ordinary controls for simple features; use a custom searchable interface when the task
needs a catalogue, several related inputs or richer results.

For a custom HTML interface, reuse the supplied `surface-sdk.js` module and the public `hm-*` UI
kit. The installed author-reference folder contains `_code/surface-sdk.js`,
`_code/surface-ui-kit.css` and an internal `_code/surface-sdk.manifest.json` with source hashes.
Copy/import the module with ordinary file tools; do not rewrite MessagePort request routing,
pending requests, errors or subscriptions. The host injects the UI kit. The same SDK is supplied to
the conversion worker and the in-client authoring agent. Read the
[custom interface quickstart](../application/custom-interface-quickstart.md#reuse-the-supplied-sdk).
Keep the embedded document, main container and form background transparent; the host owns the page
background. Use the kit's control sizing, colors, spacing, focus and selected-state classes, without
an arbitrary page-wide card or custom theme. Top-align form fields and keep optional helper text
below its control; see the [aligned form example](../application/custom-interface-quickstart.md#transparent-documents-and-aligned-fields).

Player-facing labels, progress and errors describe the game operation in plain language. Use
"Waiting to enable mods..." or "Item spawning is paused.", not "Waiting for the host" or
"Surface paused". An inactive SDK alone does not establish that the game is absent; say
"Waiting for the game" only when the authoritative game state says so. Show a useful task-level
failure and preserve the player's draft; put API names, error codes and exception text in
diagnostics or optional technical details. Follow the client's existing language conventions.
Give cause-specific advice only when the actual error or authoritative state supports it. A missing
byte pattern in an attached game does not mean the game is absent: show a task failure and optional
technical details without telling the player to launch it or inventing another remedy.

## Choosing an input control

Setting a value and repeating a command are different interactions. An ordinary number/text
control submits an edited value according to its normal commit behavior. For Add Money, Grant XP,
Add Souls or a similar repeatable action, use **one row with a separate input and button**:

```json
{
  "id": "add-currency", "kind": "number", "label": "Add currency", "valueType": "integer",
  "constraints": { "minimum": 1, "maximum": 1000000, "step": 1 },
  "presentation": { "submitMode": "explicit", "buttonLabel": "Add", "defaultValue": 500 },
  "write": { "action": "add-currency", "valueParameter": "amount" }
}
```

The input and button have separate boundaries and a gap; they still belong to one widget and action.
In explicit mode, typing and blur do not execute anything. Clicking the button or pressing Enter
sends the current value once. Repeating either with an unchanged value repeats the command. The
action receives `{ "amount": 500 }`. Do not add a second amount-storage widget or make `enable`
perform the command. Omitting `submitMode` preserves the existing control behavior.
See the complete [input and button example](trainer-controls.md#input-with-an-explicit-button).

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

`aobScanModule` accepts module names with spaces, either unquoted or enclosed in double quotes,
for example `aobScanModule(site, "Tactics Ogre Reborn.exe", 66 41 01 86 4A 01 00 00)`.

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
  Independent hooks in one package need distinct persistent names (`healthCave`, `manaCave`); each
  hook's on/off reuse its names. Local labels within one chunk and intentionally shared setup need no prefixes.
- Bound labels, allocations and visible symbols work directly in memory operands: `mov dword ptr
  [flag], 1`, `lea rax, [flag]` and `mov rax, qword ptr [flag]`. A declared data label in a later
  chunk resolves before the earlier code chunk is written. Preserve these symbolic operands.
  On x64, nearby data can use RIP-relative addressing; for an address too far for that instruction,
  use `mov rax, distantData` followed by `mov dword ptr [rax], 1`. See the
  [cross-chunk data example](auto-assembler-api.md#instruction-and-data-syntax).
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

Packages are not version-gated unless the user requests a gate. Record game-build and verification
information in research or publication records outside the generated runtime package. Do not refuse
an operation because the running build cannot be identified, and do not
add a version reader. A package may still check something it directly depends on, such as the
original bytes at a patch site before writing. A wrong address shows up as a failed scan, a failed
action or no effect, and is fixed by updating the package.

## Testing

Create a minimal manifest and smoke-test declaration early when delegating runtime work. Give a
runtime helper the agreed runnable IDs and actual parameter object, and let it use the supplied
engine tests. Another Lua/WASM interpreter with mocked host functions cannot establish HexMod
compatibility. Test a small assembly or game-call case on the real engine before investigating its
internals. When runtime and interface files have separate owners, build them in parallel and run
the combined check after both are ready.
When delegating a hook smoke test, give the helper the complete [verified test example](#example-hook-with-inline-data),
including byte reads and relative-jump/data-operand calculations, not only the runtime assembly.
Adapt its actual instruction layout rather than guessing offsets.

For a custom interface, one focused UI smoke check should run the authored HTML/entry module with
the supplied SDK: edit/select an input, submit it, and check readiness, suspension and failure
handling. Testing an argument helper and the SDK separately does not exercise the interface's own
event handlers. Pair this with a `surfaceCommand` integration step through the declared alias and
actual JSON arguments; see [custom-interface testing](testing.md#custom-surface-command-steps).

Static validation checks the manifest, bindings and Lua syntax; it does not assemble AA. One
integration case is usually enough: run the production "on" operation against the controlled test
target, check the bytes or value it wrote, run "off" and check the original state is back. Add unit
cases only for real logic. See [testing](testing.md) and [test helpers](package-test-helpers.md).
Reuse passing receipts while the tested source files, manifest, tests and engine are unchanged;
rerun after relevant edits or a new concern about the result.

Declare named game memory with `fixture.modules`. The test target loads real modules, so setup
seeds `ctx.fixture.modules[index].base + sourceOffset` for module-relative addresses. Keep the
package's original offsets and pointer traversal. Use host `toggle` steps for the declared control
and host `action` steps for input/button controls; both use the app's actual bindings and parameter
object. Repeat an explicit input action with the same value to prove it remains repeatable. Use
`cleanup` for lifecycle disable.
Match the source's bitness: `fixture.architecture` accepts `"x86"` or `"x64"` (default). An available
x86 fixture is a real 32-bit process with matching modules. A byte check in an x64 fixture does not
prove x86 execution. If the installed test build lacks that target, report the test limitation;
it does not establish that the production engine cannot implement the feature.
Compute fixture sizes with the shell, and cover the source offset plus the accessed byte width.
For example, PowerShell `0x19B8900 / 1MB` is about 25.72 MiB; a 32 MiB module
(`size: 33554432`) can cover an eight-byte pointer at that offset. Keep that root at
`module.base + 0x19B8900`, and seed intermediate pointer cells at nonoverlapping positions relative
to `module.dataAddress`, allowing for each hop and field width. Never replace a production offset
with a smaller test address. If a reported allocation limit really prevents the source address,
record the test limitation instead of changing what the package patches.

Lua steps seed or inspect memory. A passing test proves the actual actions assemble and write what
they should; it does not prove the addresses match a given game build.

## Generated code and reference use

Keep comments technical: explain field widths, pointer layouts, patch behavior or cleanup. Keep
credits, source headers, table entry IDs, license notices and provenance in separate research or
publication records, not in newly generated runtime files or package documentation. Preserve the
original acquired resources separately.

Read the exact installed API contract and use its examples. Do not invent a host function because
another tool provides one. If a requested feature needs an unavailable capability, explain the
specific missing operation instead of inserting an unconditional error into a supposed working
action. A smoke test establishes only the behavior it actually exercised.

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
local ORIGINAL = {0xE8, 0x2D, 0xB9, 0xFE, 0xFF}
local function site(ctx) return ctx.fixture.modules[1].base + 0x4948AE end
local function check(ctx, expected)
  local actual = readBytes(site(ctx), 5, true)
  for index, byte in ipairs(expected) do assert(actual[index] == byte) end
end
return {
  setup = function(ctx) writeBytes(site(ctx), ORIGINAL) end,
  on = function(ctx) check(ctx, {0x90, 0x90, 0x90, 0x90, 0x90}) end,
  off = function(ctx)
    check(ctx, ORIGINAL)
    assert(site(ctx) + 5 + readInteger(site(ctx) + 1, true)
      == ctx.fixture.modules[1].base + 0x4801E0)
  end,
}
```
{% endcode %}

{% code title="tests/studio.tests.json" %}
```json
{ "schemaVersion": 1, "cases": [{
  "id": "on-off", "kind": "integration", "runtime": "lua", "timeoutMs": 15000,
  "fixture": {"modules": [{"name": "Game.exe", "size": 5242880}]},
  "steps": [
    {"kind": "lua", "entryFile": "tests/integration.lua", "entrySymbol": "setup"},
    {"kind": "toggle", "widgetId": "example.fixed-patch/fixed-patch", "enabled": true},
    {"kind": "lua", "entryFile": "tests/integration.lua", "entrySymbol": "on"},
    {"kind": "toggle", "widgetId": "example.fixed-patch/fixed-patch", "enabled": false},
    {"kind": "lua", "entryFile": "tests/integration.lua", "entrySymbol": "off"},
    {"kind": "toggle", "widgetId": "example.fixed-patch/fixed-patch", "enabled": true},
    {"kind": "cleanup"},
    {"kind": "lua", "entryFile": "tests/integration.lua", "entrySymbol": "off"}
  ]
}] }
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
local ORIGINAL = {0x0F, 0x28, 0xB1, 0x70, 0x01, 0x00, 0x00}
local function site(ctx) return ctx.fixture.modules[1].base + 0x9C83CA end
return {
  setup = function(ctx) writeBytes(site(ctx), ORIGINAL) end,
  on = function(ctx)
    local hook = site(ctx)
    assert(readBytes(hook, 1) == 0xE9)
    local cave = hook + 5 + readInteger(hook + 1, true)
    local bytes = readBytes(cave, 2, true)
    assert(bytes[1] == 0x0F and bytes[2] == 0x10, "movups in cave")
    local vector = cave + 7 + readInteger(cave + 3, true)
    assert(readFloat(vector) == 0 and readFloat(vector + 4) == -2000)
    assert(readFloat(vector + 8) == 0 and readFloat(vector + 12) == 1)
  end,
  off = function(ctx)
    local actual = readBytes(site(ctx), 7, true)
    for index, byte in ipairs(ORIGINAL) do assert(actual[index] == byte) end
  end,
}
```
{% endcode %}

{% code title="tests/studio.tests.json" %}
```json
{ "schemaVersion": 1, "cases": [{
  "id": "on-off", "kind": "integration", "runtime": "lua", "timeoutMs": 15000,
  "fixture": {"modules": [{"name": "Game.exe", "size": 16777216}]},
  "steps": [
    {"kind": "lua", "entryFile": "tests/integration.lua", "entrySymbol": "setup"},
    {"kind": "toggle", "widgetId": "example.hook-inline-data/hook", "enabled": false},
    {"kind": "toggle", "widgetId": "example.hook-inline-data/hook", "enabled": true},
    {"kind": "lua", "entryFile": "tests/integration.lua", "entrySymbol": "on"},
    {"kind": "toggle", "widgetId": "example.hook-inline-data/hook", "enabled": false},
    {"kind": "lua", "entryFile": "tests/integration.lua", "entrySymbol": "off"},
    {"kind": "toggle", "widgetId": "example.hook-inline-data/hook", "enabled": true},
    {"kind": "lua", "entryFile": "tests/integration.lua", "entrySymbol": "on"},
    {"kind": "cleanup"},
    {"kind": "lua", "entryFile": "tests/integration.lua", "entrySymbol": "off"}
  ]
}] }
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
return {
  setup = function(ctx)
    writeQword(ctx.fixture.modules[1].base + 0x4768E78, ctx.fixture.playerAddress)
    writeQword(ctx.fixture.playerAddress + 0x20, ctx.fixture.statsAddress)
    writeInteger(ctx.fixture.statsAddress + 4, 100)
  end,
  on = function(ctx) assert(readInteger(ctx.fixture.statsAddress + 4) == 999) end,
  off = function(ctx) assert(readInteger(ctx.fixture.statsAddress + 4) == 100) end,
}
```
{% endcode %}

{% code title="tests/studio.tests.json" %}
```json
{ "schemaVersion": 1, "cases": [{
  "id": "on-off", "kind": "integration", "runtime": "lua", "timeoutMs": 15000,
  "fixture": {"modules": [{"name": "Game.exe", "size": 83886080}]},
  "steps": [
    {"kind": "lua", "entryFile": "tests/integration.lua", "entrySymbol": "setup"},
    {"kind": "toggle", "widgetId": "example.pointer-write/pointer", "enabled": true},
    {"kind": "lua", "entryFile": "tests/integration.lua", "entrySymbol": "on"},
    {"kind": "toggle", "widgetId": "example.pointer-write/pointer", "enabled": false},
    {"kind": "lua", "entryFile": "tests/integration.lua", "entrySymbol": "off"},
    {"kind": "cleanup"},
    {"kind": "lua", "entryFile": "tests/integration.lua", "entrySymbol": "off"}
  ]
}] }
```
{% endcode %}

### Example: game call with separate code and data

This small package demonstrates the part used by an item spawner: prepare one code allocation and
one data allocation, pass an input to a target function, and repeat the same command. It is an
illustrative call convention, not a signature for a real game. The test seeds an executable target
function and invokes the unchanged package through its actual input/button binding.

For data in another allocation, loading its address into a register and using `[register+offset]`
is a straightforward x64 form. The test waits for the target thread to finish before cleanup.

{% code title="package.json" %}
```json
{
  "id": "example.game-call", "version": "1.0.0", "displayName": "Game call",
  "runtime": {
    "files": [{"path": "lua/main.lua", "runtime": "lua"}],
    "runnables": [
      {"id": "enable", "kind": "enable", "runtime": "lua", "entryFile": "lua/main.lua", "entrySymbol": "enable"},
      {"id": "disable", "kind": "disable", "runtime": "lua", "entryFile": "lua/main.lua", "entrySymbol": "disable"},
      {"id": "grant", "kind": "action", "runtime": "lua", "entryFile": "lua/main.lua", "entrySymbol": "grant"}
    ]
  },
  "frontend": {"hosted": {"schemaVersion": 1, "groups": [{
    "id": "resources", "label": "Resources", "widgets": [{
      "id": "grant", "kind": "number", "label": "Grant", "valueType": "integer",
      "constraints": {"minimum": 1, "maximum": 1000000, "step": 1},
      "presentation": {"submitMode": "explicit", "buttonLabel": "Give", "defaultValue": 500},
      "write": {"action": "grant", "valueParameter": "amount"}
    }]
  }]}}
}
```
{% endcode %}

{% code title="lua/main.lua" %}
```lua
local PREPARE = [[
aobScanModule(grantCall, "Game.exe", 01 11 49 B8 ?? ?? ?? ?? ?? ?? ?? ??)
alloc(grantCode, 0x1000, grantCall)
alloc(grantData, 0x1000, grantCall)
registerSymbol(grantCode)
registerSymbol(grantData)
grantData:
  .dd 0, 0, 0
grantCode:
  sub rsp, 0x28
  mov rcx, grantData
  mov edx, [rcx+4]
  mov rax, grantCall
  call rax
  mov rcx, grantData
  add dword ptr [rcx+8], 1
  add rsp, 0x28
  ret
]]

local function assemble(text)
  local ok, message = autoAssemble(text)
  if not ok then error(message) end
end

function enable() assemble(PREPARE) end
function grant(params)
  writeInteger(getAddress("grantData") + 4, params.amount)
  assemble("createThread(grantCode)")
end
function disable()
  assemble("unregisterSymbol(grantData)\nunregisterSymbol(grantCode)\ndealloc(grantData)\ndealloc(grantCode)")
end
```
{% endcode %}

{% code title="tests/integration.lua" %}
```lua
local function waitForCall(ctx, count, expected)
  for attempt = 1, 100 do
    local data = readQword(ctx.fixture.statsAddress + 8)
    if data ~= 0 and readInteger(data + 8) == count then
      assert(readInteger(data) == expected, "target function received the requested amount")
      return
    end
    sleep(10)
  end
  error("target function did not complete")
end

return {
  setup = function(ctx)
    local target = ctx.fixture.modules[1].dataAddress
    fullAccess(target, 0x1000)
    writeQword(ctx.fixture.statsAddress + 8, 0)
    -- Observe the real call's data pointer without reaching into another package's symbols.
    local code = string.format([[
0x%X:
  add dword ptr [rcx], edx
  mov r8, 0x%X
  mov qword ptr [r8+8], rcx
  xor eax, eax
  ret
]], target, ctx.fixture.statsAddress)
    local ok, message = autoAssemble(code)
    assert(ok, message)
  end,
  first = function(ctx) waitForCall(ctx, 1, 500) end,
  second = function(ctx) waitForCall(ctx, 2, 1000) end,
}
```
{% endcode %}

{% code title="tests/studio.tests.json" %}
```json
{"schemaVersion": 1, "cases": [{
  "id": "repeated-call", "kind": "integration", "runtime": "lua", "timeoutMs": 15000,
  "fixture": {"modules": [{"name": "Game.exe", "size": 65536}]},
  "steps": [
    {"kind": "lua", "entryFile": "tests/integration.lua", "entrySymbol": "setup"},
    {"kind": "action", "widgetId": "example.game-call/grant", "value": 500},
    {"kind": "lua", "entryFile": "tests/integration.lua", "entrySymbol": "first"},
    {"kind": "action", "widgetId": "example.game-call/grant", "value": 500},
    {"kind": "lua", "entryFile": "tests/integration.lua", "entrySymbol": "second"},
    {"kind": "cleanup"}
  ]
}]}
```
{% endcode %}

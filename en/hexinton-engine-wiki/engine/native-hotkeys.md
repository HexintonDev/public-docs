# Native Hotkeys for Continuous Input

Status: current behavior of the desktop client and engine kits that include the Lua `hotkeys`
API (engine kit `conversion-wave6` and later). Older clients load such a package but cannot run
its hotkeys; see [Compatibility](#compatibility).

Ordinary features use **UI hotkeys**: the player binds a key to a trainer control (toggle, set
value, execute) in the trainer's hotkey editor. That model cannot express continuous input such as
flying, where the script must know whether a key is *held*. **Native hotkeys** let a package's Lua
code read declared keys directly while the feature that owns them is running.

## When to use them

| Source feature | Package shape |
| --- | --- |
| Fly Mode / Fly Hack with Fly Up and Fly Down keys | One `Fly Mode` toggle with `hold` hotkeys `fly-up` and `fly-down` |
| No Clip / Free Camera that moves with direction keys | One toggle with `hold` hotkeys for each direction it needs |
| Speed boost while a key is held | One toggle with a `hold` hotkey `boost` |
| Unlimited Health, Add Money, Teleport to Waypoint | Ordinary toggle, input/button or action; the player may bind a UI hotkey |

Rules:

- Continuous movement is **one toggle** that owns its keys. Never split it into separate "Fly Up"
  / "Fly Down" action widgets, and never ask the player to bind UI hotkeys to make it usable.
- Give every native hotkey a sensible default chord so the feature works without setup. Common
  choices: `Space` up, `Control` down, `W`/`A`/`S`/`D` or the arrow keys for horizontal direction,
  `Shift` for a boost. Two hotkeys in one package cannot share a default chord.
- Do not turn one-shot actions into native hotkeys. A one-shot effect stays an `action` that the
  player can bind with a UI hotkey.

## Declaring hotkeys

`runtime.hotkeys` declares the keys, because the Lua runtime enforces them. A toggle widget's
`nativeHotkeys` array says which trainer row shows and owns them.

```json
{
  "id": "example.movement",
  "version": "1.0.0",
  "runtime": {
    "hotkeys": [
      { "id": "fly-up", "label": "Fly up", "mode": "hold", "defaultChord": { "key": "Space", "modifiers": [] } },
      { "id": "fly-down", "label": "Fly down", "mode": "hold", "defaultChord": { "key": "Control" } }
    ],
    "runnables": [
      { "id": "enable", "kind": "enable", "runtime": "lua", "entryFile": "lua/main.lua", "entrySymbol": "enable" },
      { "id": "disable", "kind": "disable", "runtime": "lua", "entryFile": "lua/main.lua", "entrySymbol": "disable" },
      { "id": "fly-on", "kind": "action", "runtime": "lua", "entryFile": "lua/main.lua", "entrySymbol": "startFly" },
      { "id": "fly-off", "kind": "action", "runtime": "lua", "entryFile": "lua/main.lua", "entrySymbol": "stopFly" }
    ]
  },
  "frontend": { "hosted": { "schemaVersion": 1, "groups": [{
    "id": "movement", "label": "Movement",
    "widgets": [{
      "id": "fly-mode", "kind": "toggle", "label": "Fly Mode",
      "write": { "enableAction": "fly-on", "disableAction": "fly-off" },
      "nativeHotkeys": ["fly-up", "fly-down"]
    }]
  }] } }
}
```

`runtime.hotkeys[]` (at most 16 per package):

| Field | Rules |
| --- | --- |
| `id` | Stable identifier, `[A-Za-z0-9][A-Za-z0-9._-]{0,63}`, unique in the package. Never translated. |
| `label` | Short action text shown on the row, at most 64 bytes ("Fly up"). Package locales may translate it. |
| `mode` | `"hold"`: press and release edges plus `isDown`; extra modifiers are ignored. `"press"`: one edge per press; modifiers must match exactly. |
| `defaultChord` | `{ "key": <key name>, "modifiers": ["control" \| "shift" \| "alt" \| "meta"] }`; `modifiers` may be omitted. |

Key names are case-insensitive: `A`–`Z`, `0`–`9`, `F1`–`F24`, `Numpad0`–`Numpad9`, `NumpadAdd`,
`NumpadSubtract`, `NumpadMultiply`, `NumpadDivide`, `NumpadDecimal`, `Space`, `Enter`, `Tab`,
`Escape`, `Backspace`, `Insert`, `Delete`, `Home`, `End`, `PageUp`, `PageDown`, `CapsLock`,
`ArrowUp`, `ArrowDown`, `ArrowLeft`, `ArrowRight`, `Control`, `Shift`, `Alt` (a modifier may be
the held key itself), `MouseMiddle`, `Mouse4` and `Mouse5`. Gamepad buttons are not supported.

Widget `nativeHotkeys` (at most 8) is allowed only on `toggle` widgets and must reference declared
ids; a hotkey may appear on at most one widget. Always reference every declared hotkey from its
toggle: an unreferenced hotkey still works but the player cannot see or rebind it.

Package validation rejects malformed declarations with the manifest line and column.

## Lua API

Every package environment has a `hotkeys` table.

| Call | Behavior |
| --- | --- |
| `hotkeys.activate(id [, { onPress = fn, onRelease = fn }])` | Starts sampling the declared hotkey for this package. Callbacks receive the hotkey id. `onRelease` requires mode `hold`. Re-activating replaces callbacks. |
| `hotkeys.deactivate(id)` | Stops sampling and releases callbacks. Returns whether it was active. Does not call `onRelease`. |
| `hotkeys.isDown(id)` | Mode `hold` only. `true` while active, the game window is in front and the chord is held. |
| `hotkeys.isActive(id)` | Whether this package currently has the hotkey active. |

Undeclared ids, another package's ids, wrong callback types and mode misuse raise Lua errors.

Fly Mode with held up/down keys, polled from a timer:

```lua
-- Space rises and Ctrl descends while Fly Mode is on.
local HEIGHT = "[[game.exe+0x3D5DF38]+0x10]+0x6B8"
local STEP = 0.35
local timer

local function tick()
  local delta = 0
  if hotkeys.isDown("fly-up") then delta = delta + STEP end
  if hotkeys.isDown("fly-down") then delta = delta - STEP end
  if delta ~= 0 then writeFloat(HEIGHT, readFloat(HEIGHT) + delta) end
end

function enable() end

function startFly()
  if timer then return end
  timer = createTimer(false)
  timer.Interval = 16
  timer.OnTimer = tick
  timer.Enabled = true
  -- Activate last, so a failed setup never leaves keys active.
  hotkeys.activate("fly-up")
  hotkeys.activate("fly-down")
end

function stopFly()
  hotkeys.deactivate("fly-up")
  hotkeys.deactivate("fly-down")
  if timer then timer:destroy(); timer = nil end
end

function disable() stopFly() end
```

An edge-driven alternative uses callbacks instead of polling:

```lua
hotkeys.activate("fly-up", {
  onPress = function() writeFloat(VELOCITY_Y, 6.0) end,
  onRelease = function() writeFloat(VELOCITY_Y, 0.0) end,
})
```

If the source keeps the feature's own gravity or collision patch, apply it in the toggle's "on"
action and restore it in "off" and `disable`, exactly as without hotkeys. The hotkeys only replace
the source's separate up/down key handling.

## Semantics

- **Sampling.** While at least one hotkey is active, the engine samples input every 15 ms on the
  same thread as timers and package calls, so callbacks never run concurrently with package code.
  Nothing is hooked or consumed: the game still receives every key.
- **Focus.** Keys count only while the attached game's window is in front. Typing in Hexinton Mod,
  the overlay or another application never drives a hotkey. Losing focus counts as releasing every
  key, so a held key reports `onRelease` instead of staying stuck.
- **Hold.** `onPress` fires on the first sample that sees the chord, including a key already held
  at activation; `onRelease` fires when it is released or focus is lost.
- **Press.** One `onPress` per physical press, without auto-repeat.
- **Ownership.** An activation ends on `deactivate`, when the package's script state is destroyed
  (disable, rollback, shutdown), when a reload drops the id or changes its mode, or when one of its
  callbacks raises an error (the error is logged and the hotkey released).

## What the player sees

The owning trainer row shows the keys while it is listed, for example
"While on [Space] Fly up [Ctrl] Fly down". The player can rebind them in that row's hotkey editor
under "Feature keys", and the client warns about conflicts with UI hotkeys or other rows. Packages
do not implement rebinding or conflict handling themselves.

## Testing

Package integration cases drive hotkeys in the owned test process with a `hotkey` step:

```json
{
  "id": "fly-mode-hold", "kind": "integration", "runtime": "lua",
  "fixture": { "modules": [{ "name": "game.exe", "size": 67108864 }] },
  "steps": [
    { "kind": "lua", "entryFile": "tests/fly.lua", "entrySymbol": "setup" },
    { "kind": "toggle", "widgetId": "example.movement/fly-mode", "enabled": true },
    { "kind": "hotkey", "hotkeyId": "fly-up", "state": "down", "waitMs": 300 },
    { "kind": "hotkey", "hotkeyId": "fly-up", "state": "up", "waitMs": 50 },
    { "kind": "lua", "entryFile": "tests/fly.lua", "entrySymbol": "assertRose" },
    { "kind": "toggle", "widgetId": "example.movement/fly-mode", "enabled": false },
    { "kind": "cleanup" }
  ]
}
```

`hotkeyId` must be declared in the tested package's `runtime.hotkeys`; an undeclared id makes the
test invalid. `state` is `down` or `up`. `waitMs` (0–5000, default 100) lets the engine sample and
the package's timers run before the next step. The simulated state replaces the keyboard and focus
for that hotkey only and ends with the case. Assert the effect (the height rose while held and
stopped after release), not only that the toggle ran.

## Compatibility

Packages cannot declare a minimum client version. A client without this API still loads and
validates a native-hotkey package and shows its toggle without feature keys, but the toggle fails
when it runs (`hotkeys` is `nil`). Native hotkeys therefore require a desktop client with this API.

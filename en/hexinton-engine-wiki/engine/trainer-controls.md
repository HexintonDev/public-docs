# Trainer Controls and UI Bindings

Status: current public technical guide.

Lua functions become usable trainer features through package declarations. The runtime defines the
operation; the hosted trainer view defines how that operation is presented to a user. The live value
path is explained in [View Models and State Feeds](view-models-and-feeds.md).

## Choose the Contract

| User need | Package runnable | UI binding | What comes back |
| --- | --- | --- | --- |
| Perform one operation | `action` | `command` | One correlated command result |
| Read once | `query` | `query` | One response for that request |
| Keep observing a value | `service` | `viewModel` | Repeated host state updates |

Use an `action` for a button, toggle write, patch, or explicit refresh. Use a `query` for a
one-shot search or read. Use a `service` when the value can change without the current UI action.

## Complete hosted toggle

Create `example.toggle/package.json`:

```json
{
  "id": "example.toggle",
  "version": "1.0.0",
  "displayName": "Example Toggle",
  "runtime": {
    "files": [{ "path": "lua/main.lua", "runtime": "lua" }],
    "runnables": [
      { "id": "enable", "kind": "enable", "runtime": "lua", "entryFile": "lua/main.lua", "entrySymbol": "enable" },
      { "id": "disable", "kind": "disable", "runtime": "lua", "entryFile": "lua/main.lua", "entrySymbol": "disable" },
      { "id": "turn-on", "kind": "action", "runtime": "lua", "entryFile": "lua/main.lua", "entrySymbol": "turnOn" },
      { "id": "turn-off", "kind": "action", "runtime": "lua", "entryFile": "lua/main.lua", "entrySymbol": "turnOff" },
      { "id": "state", "kind": "query", "runtime": "lua", "entryFile": "lua/main.lua", "entrySymbol": "readState" }
    ]
  },
  "frontend": {
    "hosted": {
      "schemaVersion": 1,
      "groups": [{
        "id": "player",
        "label": "Player",
        "widgets": [{
          "id": "example-mode",
          "kind": "toggle",
          "label": "Example Mode",
          "description": "Demonstrates a control and package state; it has no gameplay effect.",
          "write": { "enableAction": "turn-on", "disableAction": "turn-off" },
          "read": { "query": "state", "resultPointer": "/enabled" },
          "tags": [{ "id": "example", "label": "Example" }]
        }]
      }]
    }
  }
}
```

Create `example.toggle/lua/main.lua`:

```lua
local state = { enabled = false }

function enable()
    -- Prepare package resources only; displaying or reading this control must not turn it on.
end

function turnOn()
    state.enabled = true
end

function turnOff()
    state.enabled = false
end

function readState()
    return { enabled = state.enabled }
end

function disable()
    turnOff()
end
```

The write IDs must name `action` runnables, not the package's `enable`/`disable` hooks. The read
ID must name a `query`. Package preparation happens before an action or query when needed:
put a user-selected gameplay change in the action, because even the initial state query can
prepare the package. Keep cleanup of owned resources in the lifecycle `disable` hook.

A Lua query must return exactly one detached JSON-compatible value; objects and arrays are
supported. Here that value is `{ "enabled": false }`. Native transport wraps it in a `result`
field, but the hosted `resultPointer` is relative to the query value, so use `/enabled`, not
`/result/enabled`. Use an empty pointer when reading a scalar query value.

An action's Lua return values are discarded. Returning `false` or `{ ok = false }` does not
fail the command. Raise an error when a required check or cleanup fails; otherwise a toggle
may commit the requested state despite an unsuccessful operation. For a non-service-backed
toggle, command success can commit that requested boolean; it does not establish an independently
observed game value. Use a service/feed when other code can change the actual state.

## Groups and control tags

Author groups live in `frontend.hosted.groups`. Each group has a stable `id`, a readable `label`,
and `widgets`. Choose groups around the user's task, such as Player, Combat or Inventory. Group
IDs must not change when labels are translated.

Each widget can additionally declare an optional `tags` array:

```json
"tags": [{ "id": "health", "label": "Health" }, { "id": "survival", "label": "Survival" }]
```

This is a fragment inside a widget, not a top-level manifest field. The current compiler accepts
at most 32 tags per widget, with unique IDs and IDs/labels no longer than 64 characters. Repeated
tag IDs across widgets in one package must use the same label. The host prefixes tag identity
with the package ID. These control tags are separate from the backend package page's string tags.
Personal trainer organization may override layout without rewriting the author manifest.

## Phone remote sharing

Toggle, action, number, and select widgets of published packages are shared with the player's phone
remote by default. The phone receives only the widget's label, description, options, constraints, and
current value, never its bindings, scripts, or hotkeys. To keep a widget on the PC only, declare
`"remoteAllowed": false` on that widget:

```json
{ "id": "unlock-all", "kind": "action", "label": "Unlock all", "remoteAllowed": false }
```

Use it for controls that need the player to look at the game screen first. Other widget kinds and
locally installed (unpublished) packages are never shared. Custom interfaces are not shown on the phone
remote.

An action result answers “did this operation finish?”. It is not a live value and does not replace a
later Service observation. If another source can change the value, display the ViewModel updated by
the Service rather than the last value returned by the action.

## Input with an explicit button

A repeatable command such as Add Souls, Add Money or Grant XP needs one row with a separate input
and button, each with its own boundary and a gap between them. They are one declared widget and
one action. Use the existing `number` or `text` widget with `presentation.submitMode` set to
`explicit`. The entered value goes directly to the action; it does not need a second widget or a
separate amount-storage action.

| Presentation field | Contract |
| --- | --- |
| `submitMode` | `explicit` submits only from the button or Enter. Omit it for the existing number/text commit behavior. |
| `buttonLabel` | Optional button label, such as `Add` or `Give item`. |
| `defaultValue` | Optional initial draft value when no read value is available: a number for a number widget, or a string for a text widget. Used by explicit mode only; it does not execute the action. |

Typing, blur and number adjustments only edit the draft in explicit mode. Clicking the button or
pressing Enter submits the current valid value once, including when it is unchanged from the last
submission. Validation and pending execution prevent another submission until the control is ready.
Keep `write.action` and `write.valueParameter` on the same widget. A Lua action receives one object:
`{ "amount": 500 }` becomes a table read as `params.amount`, not the positional number `500`.

The following complete package is an engine fixture example. `game.exe+0x2100` is illustrative
memory in a named test module; it is not a verified currency address for any game.

{% code title="package.json" %}
```json
{
  "id": "example.add-currency", "version": "1.0.0", "displayName": "Add currency",
  "runtime": {
    "files": [{ "path": "lua/main.lua", "runtime": "lua" }],
    "runnables": [
      { "id": "enable", "kind": "enable", "runtime": "lua", "entryFile": "lua/main.lua", "entrySymbol": "enable" },
      { "id": "disable", "kind": "disable", "runtime": "lua", "entryFile": "lua/main.lua", "entrySymbol": "disable" },
      { "id": "add-currency", "kind": "action", "runtime": "lua", "entryFile": "lua/main.lua", "entrySymbol": "addCurrency",
        "parameterSchema": {
          "type": "object", "required": ["amount"], "additionalProperties": false,
          "properties": { "amount": { "type": "integer", "minimum": 1, "maximum": 1000000 } }
        }
      }
    ]
  },
  "frontend": { "hosted": { "schemaVersion": 1, "groups": [{
    "id": "resources", "label": "Resources", "widgets": [{
      "id": "add-currency", "kind": "number", "label": "Add currency", "valueType": "integer",
      "constraints": { "minimum": 1, "maximum": 1000000, "step": 1 },
      "presentation": { "submitMode": "explicit", "buttonLabel": "Add", "defaultValue": 500 },
      "write": { "action": "add-currency", "valueParameter": "amount" }
    }]
  }]}}
}
```
{% endcode %}

{% code title="lua/main.lua" %}
```lua
function enable() end
function disable() end

function addCurrency(params)
  local address = getAddress("game.exe+0x2100")
  writeInteger(address, readInteger(address) + params.amount)
end
```
{% endcode %}

The lifecycle functions prepare and clean up package resources. This one-time operation owns no
timer or reversible patch, so they have no work to do. A real package replaces the sample address
with its supported resolver or game-function call. A confirmed additive operation is not undone by
disabling the package.

Use a host-driven `action` step to check the same binding and parameter object as the app. This
test repeats the unchanged amount and inspects the resulting memory after each command.

{% code title="tests/integration.lua" %}
```lua
local function address(ctx) return ctx.fixture.modules[1].base + 0x2100 end
return {
  setup = function(ctx) writeInteger(address(ctx), 0) end,
  first = function(ctx) assert(readInteger(address(ctx)) == 500) end,
  second = function(ctx) assert(readInteger(address(ctx)) == 1000) end,
}
```
{% endcode %}

{% code title="tests/studio.tests.json" %}
```json
{ "schemaVersion": 1, "cases": [{
  "id": "repeated-add", "kind": "integration", "runtime": "lua", "timeoutMs": 15000,
  "fixture": { "modules": [{ "name": "game.exe", "size": 16384 }] },
  "steps": [
    { "kind": "lua", "entryFile": "tests/integration.lua", "entrySymbol": "setup" },
    { "kind": "action", "widgetId": "example.add-currency/add-currency", "value": 500 },
    { "kind": "lua", "entryFile": "tests/integration.lua", "entrySymbol": "first" },
    { "kind": "action", "widgetId": "example.add-currency/add-currency", "value": 500 },
    { "kind": "lua", "entryFile": "tests/integration.lua", "entrySymbol": "second" },
    { "kind": "cleanup" }
  ]
}] }
```
{% endcode %}

## Hosted View Models

Declare a ViewModel in `frontend.hosted` and connect it to a Service:

```json
{
  "frontend": {
    "hosted": {
      "schemaVersion": 1,
      "viewModels": [{ "id": "health", "service": "watch-health", "autoStart": true }],
      "groups": []
    }
  }
}
```

The Service calls `publishEvent`; it does not update a control directly. The host matches the event
to the declared Service, stores the payload as Feed state, updates the ViewModel, and sends a state
update to the UI. Publish a small initial JSON payload and later publish only meaningful changes.
The host adds package scope and revision metadata. Do not use a query as a replacement for a live
Service.

## Binding an Action

The complete bindings object for a surface can contain both the current ViewModel value and the
Command that changes it:

```json
{
  "bindings": {
    "health.current": {
      "type": "viewModel",
      "viewModel": "health"
    },
    "health.set": {
      "type": "command",
      "action": "set-health"
    }
  }
}
```

The binding alias is local to the surface. The command's arguments are validated against the action
`parameterSchema` in the package manifest.

## Commands, Queries, and Feeds

These bindings are shown as one complete object so the JSON can be copied and then nested under a
surface declaration:

```json
{
  "bindings": {
    "inventory.search": {
      "type": "query",
      "query": "search-inventory"
    },
    "inventory.state": {
      "type": "viewModel",
      "viewModel": "resources"
    }
  }
}
```

Queries are appropriate for search and pagination. A Command is a request such as `set-health`. A
Feed is the host route for values published by a Service. The ViewModel is the current host
projection that controls bind to. The UI receives a command result separately from a ViewModel state
update.

## Custom Surface Declaration

```json
{
  "frontend": {
    "surfaces": [
      {
        "id": "health-panel",
        "kind": "web-inline",
        "label": "Health",
        "entry": "frontend/health-panel/index.html",
        "presentation": { "mode": "panel", "size": "large" },
        "bindings": {
          "health": { "type": "viewModel", "viewModel": "health" },
          "setHealth": { "type": "command", "action": "set-health" }
        }
      }
    ]
  }
}
```

Supported placements are `web-inline` with `compact` or `panel` mode, and `web-app` with `app` mode.
The current host loads one package-relative HTML document and enforces a 1 MiB document limit.

Surface code runs in a sandbox. It receives only declared bindings and must use the provided surface
SDK. It must not call the raw WebView bridge, access credentials, or assume unrestricted filesystem,
network, or native access.

## UI Design Rules

Use a toggle for a reversible enable/disable feature, a command button for a one-time action, a
number/text input with `submitMode: "explicit"` for a repeatable command that takes one entered
value, a query for transient results, and a read-only ViewModel value for the current value observed by the
Service. Keep local draft input separate from the committed ViewModel value. A delta is only the
transport message that changes the ViewModel; it is not the ViewModel itself.

Continuous movement (fly mode with up/down keys, noclip, free camera, a held speed boost) is one
toggle that owns script-read [native hotkeys](native-hotkeys.md) through `nativeHotkeys`, with
default keys. Do not add separate "Fly Up"/"Fly Down" actions for the player to bind as UI
hotkeys; UI hotkeys are for ordinary toggles, inputs and actions.

For an item catalogue or several related parameters, consider a custom searchable surface instead
of a long collection of unrelated inputs. Keep IDs and names as data and send one parameter object
to the declared action. A native input/button is sufficient for a single amount.

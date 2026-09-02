# View Models and State Feeds

Status: current public trainer integration reference.

This page answers one practical question: **how does a value in the target process become the value
shown by a trainer control?** Read the terms below in execution order. They are different objects and
must not be used interchangeably.

## The Four Pieces

| Piece | What it is | When it runs | What the UI receives |
| --- | --- | --- | --- |
| Command | One request from the UI, such as `set-health` | When the user clicks, toggles, or submits | One correlated success or error result |
| Service | A long-lived Lua runnable that repeatedly observes the target | After it is started, until disable or teardown | It emits values with `publishEvent` |
| Feed | The host-side route for events from one service to one named state stream | Whenever the service publishes an event | A scoped value and an ordered revision |
| ViewModel | The host's current structured projection used by controls and surfaces | It is updated when a matching feed event is accepted | The current value, plus state updates/deltas |

The **Service produces observations**. The **Feed transports them**. The **host updates the
ViewModel**. The **UI binds to the ViewModel**. A `delta` is only the message describing a change to
the ViewModel; it is not another name for the ViewModel and it is not the original service event.

## Which One Should I Use?

Start with the value's lifetime:

| Requirement | Declare | Example |
| --- | --- | --- |
| The user asks for one result and waits for it | `query` | Read the current inventory once |
| The user asks the engine to perform one operation | `action`/command | Set health to `100` or teleport |
| The value can change without a button click | `service` + `viewModel` | Health changes because the game continues running |
| The UI needs a changing value from a service | `viewModel` bound to that service | A read-only health label or slider value |
| The UI needs to start that observation | `command` for the service, or `autoStart` | Start `watch-health` |

Do not use a command result as the current value. For example, `set-health` may return `{ health =
100 }`, but that only proves what the operation returned. The game, another package, or an external
event may change health immediately afterward. The `watch-health` service observes the target and
publishes the value that the host accepts into the ViewModel.

## Complete Health Example

The following example shows the whole path. It uses a module expression only as a placeholder;
replace it with an address or shared dependency that you have verified for your target.

### 1. Lua service and command

Put this in `lua/main.lua`:

```lua
local HEALTH_ADDRESS = "game.exe+0x1234"
local healthTimer
local lastHealth

local function readHealth()
    return readInteger(HEALTH_ADDRESS, true)
end

local function publishHealth()
    local health = readHealth()
    if health == lastHealth then
        return
    end

    lastHealth = health
    publishEvent("health_changed", { health = health })
end

function enable()
end

function disable()
    if healthTimer then
        healthTimer:destroy()
        healthTimer = nil
    end
    lastHealth = nil
end

function setHealth(parameters)
    writeInteger(HEALTH_ADDRESS, parameters.health)
    publishHealth()
    return { ok = true, health = readHealth() }
end

function watchHealth(parameters)
    if healthTimer then
        return
    end

    publishHealth()
    healthTimer = createTimer(false)
    healthTimer.Interval = (parameters and parameters.interval) or 250
    healthTimer.OnTimer = function()
        publishHealth()
    end
    healthTimer.Enabled = true
end
```

What this code does:

1. `watchHealth` is the Service. It reads the target immediately, then reads it again every 250 ms.
2. `publishEvent` sends an observation named `health_changed`; it does not update the UI directly.
3. The comparison with `lastHealth` prevents identical observations from producing unnecessary state
   updates.
4. `setHealth` is a Command entry point. Its return table is the command result, not the Feed.
5. `disable` destroys the timer, so the Service stops producing observations.

### 2. Package manifest

Declare the runnables and connect the Service to a ViewModel in `package.json`:

```json
{
  "id": "example.health",
  "version": "1.0.0",
  "displayName": "Health Example",
  "runtime": {
    "runnables": [
      {
        "id": "enable",
        "kind": "enable",
        "runtime": "lua",
        "entryFile": "lua/main.lua",
        "entrySymbol": "enable"
      },
      {
        "id": "disable",
        "kind": "disable",
        "runtime": "lua",
        "entryFile": "lua/main.lua",
        "entrySymbol": "disable"
      },
      {
        "id": "watch-health",
        "kind": "service",
        "runtime": "lua",
        "entryFile": "lua/main.lua",
        "entrySymbol": "watchHealth",
        "parameterSchema": {
          "type": "object",
          "properties": {
            "interval": { "type": "number" }
          }
        }
      },
      {
        "id": "set-health",
        "kind": "action",
        "runtime": "lua",
        "entryFile": "lua/main.lua",
        "entrySymbol": "setHealth",
        "parameterSchema": {
          "type": "object",
          "required": ["health"],
          "properties": {
            "health": { "type": "number" }
          }
        }
      }
    ]
  },
  "frontend": {
    "hosted": {
      "schemaVersion": 1,
      "viewModels": [
        {
          "id": "health",
          "service": "watch-health",
          "autoStart": true
        }
      ],
      "groups": [
        {
          "id": "player",
          "label": "Player",
          "widgets": [
            {
              "id": "health",
              "kind": "number",
              "label": "Health",
              "valueType": "integer",
              "write": {
                "action": "set-health",
                "valueParameter": "health"
              }
            }
          ]
        }
      ]
    }
  }
}
```

The important connections are:

| Manifest field | Connects |
| --- | --- |
| `watch-health.entrySymbol` | The manifest to `watchHealth(parameters)` |
| `viewModels[0].service` | The ViewModel to the `watch-health` Service |
| `widgets[0].write.action` | A user edit to the `set-health` Command |
| `widgets[0].id` | The `health` property in `health_changed` to the widget value |

`autoStart: true` asks the host to start the Service when the package and surface are allowed to
run. If it is not enabled, bind a command or another supported lifecycle mechanism that starts the
Service before expecting live values.

### 3. What the UI sees

After `publishEvent("health_changed", { health = 80 })`, the host performs the following work:

```text
Lua watchHealth
  -> publishEvent("health_changed", { health = 80 })
  -> host matches package + service + event to the health feed
  -> host stores the feed value and increments its revision
  -> host applies health = 80 to the current ViewModel
  -> host emits a state delta for the ViewModel path
  -> UI accepts the newer revision and renders 80
```

The UI does not call `readInteger`, `publishEvent`, or the Lua runtime. It receives a bootstrap
snapshot and later state updates through the declared surface binding. A command completion travels
on a separate correlated channel:

```text
click Set Health
  -> command { runnableId: "set-health", parameters: { health: 100 } }
  -> command result { ok: true, health: 100 }
  -> later service observation { health: 100 }
  -> ViewModel update and UI render
```

The command result and the service observation may arrive at different times. The second is what
keeps a live value correct after changes that did not originate from the button.

## Binding Syntax

A hosted surface binds to the declared ViewModel, not to Lua:

```json
{
  "bindings": {
    "health.current": {
      "type": "viewModel",
      "viewModel": "health"
    },
    "health.set": {
      "type": "command",
      "action": "set-health",
      "executionPolicy": "write"
    }
  }
}
```

The `viewModel` binding identifies the current host projection. The `command` binding identifies a
one-shot operation. They are separate because a successful operation does not itself establish the
next value observed by the Service and accepted by the host.

## Ordered Delivery and Lifecycle

```text
service -> native event queue -> host event pump -> pid/session validation
  -> GameSession worker -> feed projection -> ViewModel update
  -> stateVersion delta -> checkpoint -> UI
```

The event pump drains queued events after the process wait handle signals; it is not a fixed UI
polling timer. The host rejects stale or out-of-scope events. A delta is sent before its matching
checkpoint. The UI should apply only a newer `stateVersion` for its current package and surface.

On disable, the Service stops, timers are destroyed, and no newer values should be expected from
that Service. The UI must show loading before the first value, unavailable when the package or
session cannot provide it, stale when updates stop according to the feature's policy, and error when
the Service reports a read or publication failure.

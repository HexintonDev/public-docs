# Complete Custom Interface Example

Status: current protocol-v1 example.

This package is a complete, copyable custom interface. It uses an in-memory counter instead of a
real game address, so it can demonstrate the full protocol without inventing a target-specific
pointer. Replace the `nextHealth` and `setHealth` bodies with verified target reads and writes only
after this example works in the host.

## What This Example Proves

```text
UI button
  -> surface binding health.set
  -> set-health action
  -> command result

watch-health service
  -> publishEvent
  -> host Feed
  -> health ViewModel
  -> feed.updated state delta
  -> UI value
```

The command result and the ViewModel update are separate messages. The page waits for the ViewModel
update before treating the displayed health as the latest observed game value.

## File Tree

```text
example.health-surface/
  package.json
  lua/main.lua
  ui/health.html
```

## Complete Files

Create `example.health-surface/package.json`:

```json
{
  "id": "example.health-surface",
  "version": "1.0.0",
  "displayName": "Health Surface Example",
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
            "interval": { "type": "integer", "minimum": 50 }
          },
          "additionalProperties": false
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
            "health": { "type": "integer", "minimum": 0, "maximum": 100 }
          },
          "additionalProperties": false
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
      "groups": []
    },
    "surfaces": [
      {
        "id": "health-panel",
        "kind": "web-inline",
        "label": "Health",
        "description": "Observe and set the example health value.",
        "order": 10,
        "entry": "ui/health.html",
        "presentation": {
          "mode": "panel",
          "size": "large"
        },
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
    ]
  }
}
```

Create `example.health-surface/lua/main.lua`:

```lua
local health = 50
local maximumHealth = 100
local healthTimer = nil

local function publishHealth()
  publishEvent("health_changed", {
    health = health,
    maxHealth = maximumHealth
  })
end

local function nextHealth()
  health = health + 1
  if health > maximumHealth then
    health = 0
  end
end

function enable()
end

function disable()
  if healthTimer ~= nil then
    healthTimer:destroy()
    healthTimer = nil
  end
end

function setHealth(parameters)
  health = math.floor(tonumber(parameters.health) or health)
  health = math.max(0, math.min(health, maximumHealth))
  return {
    health = health,
    maxHealth = maximumHealth
  }
end

function watchHealth(parameters)
  if healthTimer ~= nil then
    return
  end

  publishHealth()

  healthTimer = createTimer(false)
  healthTimer.Interval = (parameters and parameters.interval) or 1000
  healthTimer.OnTimer = function(_timer)
    nextHealth()
    publishHealth()
  end
  healthTimer.Enabled = true
end
```

Create `example.health-surface/ui/health.html`:

```html
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <meta http-equiv="Content-Security-Policy" content="default-src 'none'; script-src 'unsafe-inline'; style-src 'unsafe-inline'; connect-src 'none'; base-uri 'none'; form-action 'none'">
  <title>Health</title>
  <style>
    :root { color-scheme: dark; font: 14px "Segoe UI", sans-serif; color: #f5f5f4; }
    * { box-sizing: border-box; }
    body { margin: 0; padding: 16px; background: transparent; }
    main { display: grid; gap: 12px; max-width: 360px; padding: 16px; border: 1px solid rgba(255,255,255,.16); border-radius: 8px; background: rgba(28,29,31,.86); }
    h1 { margin: 0; font-size: 17px; }
    .value { font-size: 30px; font-weight: 700; }
    .muted { min-height: 20px; color: #b8b6b2; }
    .row { display: flex; gap: 8px; align-items: center; }
    input, button { min-height: 36px; border: 1px solid rgba(255,255,255,.18); border-radius: 6px; color: inherit; font: inherit; }
    input { width: 100px; padding: 0 10px; background: rgba(255,255,255,.08); }
    button { padding: 0 14px; background: #e7833c; color: #21130a; font-weight: 700; cursor: pointer; }
    button:disabled { cursor: not-allowed; opacity: .45; }
    :focus-visible { outline: 2px solid #ffb27b; outline-offset: 2px; }
  </style>
</head>
<body>
  <main>
    <h1>Health</h1>
    <div id="value" class="value" aria-live="polite">-- / --</div>
    <div id="status" class="muted" role="status" aria-live="polite">Waiting for the host...</div>
    <div class="row">
      <label for="health">Set health</label>
      <input id="health" type="number" min="0" max="100" value="50">
      <button id="apply" type="button" disabled>Apply</button>
    </div>
  </main>
  <script>
    (() => {
      const valueElement = document.getElementById("value");
      const statusElement = document.getElementById("status");
      const input = document.getElementById("health");
      const apply = document.getElementById("apply");
      let port;
      let lifecycle = "suspended";
      let committedHealth = null;
      let committedMaximum = null;
      let acceptedRevision = 0;
      let subscriptionId = null;
      let requestSequence = 0;
      const pending = new Map();

      function post(type, payload, requestId) {
        port.postMessage({ protocolVersion: 1, type, payload, requestId });
      }

      function isDirty() {
        return committedHealth !== null && Number(input.value) !== committedHealth;
      }

      function render() {
        const ready = lifecycle === "active";
        apply.disabled = !ready || committedHealth === null || pending.size > 0 || !isValidInput();
      }

      function isValidInput() {
        const number = Number(input.value);
        return Number.isInteger(number) && number >= 0 && number <= 100;
      }

      function subscribeHealth() {
        if (subscriptionId !== null || lifecycle !== "active") {
          return;
        }
        acceptedRevision = 0;
        subscriptionId = "health-view-" + (++requestSequence);
        post("feed.subscribe", { binding: "health.current" }, subscriptionId);
      }

      function unsubscribeHealth() {
        if (subscriptionId === null) {
          return;
        }
        post("feed.unsubscribe", {}, subscriptionId);
        subscriptionId = null;
      }

      function showError(error) {
        statusElement.textContent = "Request failed: " + (error.code || "unexpected_failure");
      }

      window.addEventListener("message", (event) => {
        const bootstrap = event.data;
        const receivedPort = event.ports && event.ports[0];
        if (!receivedPort || !bootstrap || bootstrap.type !== "hexmod.surface.bootstrap" ||
            !bootstrap.payload || bootstrap.payload.protocolVersion !== 1) {
          return;
        }

        port = receivedPort;
        port.addEventListener("message", (portEvent) => {
          const message = portEvent.data;
          if (!message || message.protocolVersion !== 1) {
            return;
          }

          if (message.type === "lifecycle.changed") {
            lifecycle = message.payload.state;
            if (lifecycle === "active") {
              statusElement.textContent = "Connected. Waiting for health state...";
              subscribeHealth();
            } else {
              unsubscribeHealth();
              statusElement.textContent = lifecycle === "disposed" ? "Surface closed." : "Surface paused.";
            }
            render();
            return;
          }

          if (message.type === "request.succeeded" && message.requestId) {
            pending.delete(message.requestId);
            statusElement.textContent = "Command completed. Waiting for health state...";
            render();
            return;
          }

          if (message.type === "request.failed" && message.requestId) {
            pending.delete(message.requestId);
            showError(message.payload.error);
            render();
            return;
          }

          if (message.type === "feed.updated" && message.requestId === subscriptionId) {
            if (message.payload.revision <= acceptedRevision) {
              return;
            }
            const next = message.payload.value;
            if (!next || !Number.isInteger(next.health) || !Number.isInteger(next.maxHealth)) {
              statusElement.textContent = "Health state is invalid.";
              render();
              return;
            }
            const hadLocalDraft = isDirty();
            acceptedRevision = message.payload.revision;
            committedHealth = next.health;
            committedMaximum = next.maxHealth;
            if (!hadLocalDraft) {
              input.value = String(committedHealth);
            }
            valueElement.textContent = committedHealth + " / " + committedMaximum;
            statusElement.textContent = "Observed revision " + acceptedRevision + ".";
            render();
          }
        });
        port.start();
        post("surface.ready", {});
      }, { once: true });

      input.addEventListener("input", () => {
        post("lifecycle.setDirty", { isDirty: isDirty() });
        render();
      });

      apply.addEventListener("click", () => {
        if (apply.disabled) {
          return;
        }
        const requestId = "health-command-" + (++requestSequence);
        pending.set(requestId, true);
        statusElement.textContent = "Command submitted...";
        post("command.execute", {
          binding: "health.set",
          arguments: { health: Number(input.value) }
        }, requestId);
        render();
      });
    })();
  </script>
</body>
</html>
```

## Run It

1. Create the three files at the paths shown above.
2. Open the package in Studio and wait for it to save.
3. Apply the package changes, start the target session, and wait for the trainer to connect.
4. Open the `Health` surface.
5. The Service publishes an initial value, so the surface receives a `feed.updated` message without a
   button click.
6. Enter a value and select `Apply`. The page receives `request.succeeded` first; the next Service
   observation arrives separately and updates the displayed value.
7. Disable the package and verify that the timer stops publishing.

## What to Replace for a Real Game

- Replace the `health` variable with a verified process read.
- Replace the assignment in `setHealth` with a validated process write.
- Keep the `health.current` ViewModel binding and `health.set` Command binding.
- Keep the initial Service publication and idempotent timer cleanup.
- Recheck bounds, process attachment, architecture, address validity, and failure handling.

This example intentionally has no `stateFeeds` array and no `type: "feed"` surface binding. The
current contract uses `frontend.hosted.viewModels` and `type: "viewModel"`; the wire update remains
`feed.updated` because the host delivers ViewModel state through a Service-backed feed.

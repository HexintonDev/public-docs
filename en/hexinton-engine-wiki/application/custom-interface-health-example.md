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
  frontend/health-panel/index.html
  frontend/health-panel/surface-sdk.js
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
        "entry": "frontend/health-panel/index.html",
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
            "action": "set-health"
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

Copy the supplied `docs/_code/surface-sdk.js` from the installed author-reference bundle to
`example.health-surface/frontend/health-panel/surface-sdk.js`. Keep its build manifest outside this package. The host
injects the public `hm-*` UI kit; the page adds only its small layout rules. The SDK owns port
requests, responses, errors and subscriptions.

Create `example.health-surface/frontend/health-panel/index.html`:

```html
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>Health</title>
  <style>
    body { margin: 0; padding: var(--hm-space-4); }
    main { max-width: 360px; }
    #health { width: 100px; }
  </style>
</head>
<body class="hm-surface">
  <main class="hm-stack">
    <h1 class="hm-heading">Health</h1>
    <div id="value" class="hm-heading" aria-live="polite">-- / --</div>
    <div id="status" class="hm-status" role="status" aria-live="polite">Waiting to enable health controls...</div>
    <div class="hm-cluster">
      <label for="health">Set health</label>
      <input id="health" class="hm-input" type="number" min="0" max="100" value="50">
      <button id="apply" class="hm-button hm-button-secondary" type="button" disabled>Apply</button>
    </div>
  </main>
  <script type="module">
    import { SurfaceSdkClient, isSurfaceBootstrapMessageV1 } from "./surface-sdk.js";

    const valueElement = document.getElementById("value");
    const statusElement = document.getElementById("status");
    const input = document.getElementById("health");
    const apply = document.getElementById("apply");
    let sdk;
    let lifecycle = "suspended";
    let observedHealth = null;
    let acceptedRevision = 0;
    let unsubscribeHealth;
    let stopLifecycle;
    let busy = false;

    function isDirty() {
      return observedHealth !== null && Number(input.value) !== observedHealth;
    }

    function render() {
      const number = Number(input.value);
      const valid = input.value.trim() !== "" && Number.isInteger(number) && number >= 0 && number <= 100;
      apply.disabled = lifecycle !== "active" || observedHealth === null || busy || !valid;
    }

    function observeHealth(next, revision) {
      if (revision <= acceptedRevision) return;
      if (!next || !Number.isInteger(next.health) || !Number.isInteger(next.maxHealth)) {
        statusElement.textContent = "Health could not be read.";
        return;
      }
      // Live observations must not overwrite an amount the player is still editing.
      const hadLocalDraft = isDirty();
      acceptedRevision = revision;
      observedHealth = next.health;
      if (!hadLocalDraft) input.value = String(observedHealth);
      valueElement.textContent = observedHealth + " / " + next.maxHealth;
      statusElement.textContent = "Health updated.";
      render();
    }

    window.addEventListener("message", event => {
      if (sdk || !isSurfaceBootstrapMessageV1(event.data) || !event.ports[0]) return;
      sdk = new SurfaceSdkClient(event.ports[0], event.data.payload);
      stopLifecycle = sdk.lifecycle.onStateChange(state => {
        lifecycle = state;
        if (state === "active" && !unsubscribeHealth) {
          acceptedRevision = 0;
          statusElement.textContent = "Reading health...";
          unsubscribeHealth = sdk.viewModels.subscribe("health.current", observeHealth);
        } else if (state !== "active") {
          unsubscribeHealth?.();
          unsubscribeHealth = undefined;
          statusElement.textContent = state === "disposed" ? "Health view closed." : "Health view paused.";
        }
        render();
      });
      sdk.lifecycle.ready();
    });

    input.addEventListener("input", () => {
      sdk?.lifecycle.setDirty(isDirty());
      render();
    });

    apply.addEventListener("click", async () => {
      if (apply.disabled) return;
      busy = true;
      statusElement.textContent = "Updating health...";
      render();
      try {
        await sdk.commands.execute("health.set", { health: Number(input.value) });
        statusElement.textContent = "Health updated. Waiting for the latest value...";
      } catch {
        statusElement.textContent = "Health could not be updated. Try again.";
      } finally {
        busy = false;
        render();
      }
    });

    window.addEventListener("pagehide", () => {
      unsubscribeHealth?.();
      stopLifecycle?.();
      sdk?.dispose();
    }, { once: true });
  </script>
</body>
</html>
```

## Run It

1. Create the three source files and copy `surface-sdk.js` at the paths shown above.
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

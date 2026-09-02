# Sample: Service Timer Feed

Sample ID: `service/timer-feed`

This sample is a complete hosted package with a deterministic Service, a host Feed, and a ViewModel
that displays the published value. It is a lifecycle demonstration, not a real memory observer.

The Service starts with `health = 1`, publishes immediately, and increments the value on every timer
tick. Replace the body of `nextHealth` with verified memory reads when adapting it to a target game.

## File Tree

```text
example.service-health/
	package.json
	lua/main.lua
```

## Complete Files

Create `example.service-health/package.json`:

```json
{
	"id": "example.service-health",
	"version": "1.0.0",
	"displayName": "Example Service Health",
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
					"id": "status",
					"label": "Status",
					"widgets": [
						{
							"id": "health",
							"kind": "number",
							"label": "Health",
							"valueType": "integer",
							"order": 10
						},
						{
							"id": "max-health",
							"kind": "number",
							"label": "Maximum Health",
							"valueType": "integer",
							"order": 20
						},
						{
							"id": "status",
							"kind": "text",
							"label": "State",
							"valueType": "string",
							"order": 30
						}
					]
				}
			]
		}
	}
}
```

Create `example.service-health/lua/main.lua`:

```lua
local timer = nil
local health = 0

local function nextHealth()
		health = health + 1
		return health
end

local function publishHealth()
		publishEvent("health_changed", {
				status = "ok",
				health = health,
				maxHealth = 100
		})
end

function enable()
end

function disable()
		if timer ~= nil then
				timer:destroy()
				timer = nil
		end
		health = 0
end

function watchHealth(parameters)
		if timer ~= nil then
				return
		end

		nextHealth()
		publishHealth()

		timer = createTimer(false)
		timer.Interval = (parameters and parameters.interval) or 250
		timer.OnTimer = function(_timer)
				nextHealth()
				publishHealth()
		end
		timer.Enabled = true
end
```

## Follow the Data

The manifest's `viewModels[0].service` connects the ViewModel to the `watch-health` Service. The
Service calls `publishEvent`; there is no `feed` runnable in the manifest. The host uses the service
identity and event metadata to route `health_changed` into the Feed, stores the payload with an
ordered revision, projects matching payload properties into widgets, and sends a state delta to the
UI.

```text
watch-health -> publishEvent -> health Feed -> health ViewModel -> state delta -> UI
```

The payload properties map to widget IDs after the package prefix and camel-case conversion:
`health` maps to `example.service-health/health`, `maxHealth` maps to
`example.service-health/max-health`, and `status` maps to `example.service-health/status`.

## Run It

With an interval of `250`, the first accepted event is approximately:

```json
{ "status": "ok", "health": 1, "maxHealth": 100 }
```

Later events contain `2`, `3`, and so on. The event is not a command result: it is an observation
from a long-lived Service. After disable, the timer is destroyed, its Lua reference is cleared, and
no further values should be published.

## Adapt It to a Game

Keep the event name and payload shape, but replace `nextHealth` with a verified read such as
`readInteger("game.exe+0x1234", true)`. Publish only when a meaningful value changes, handle read
errors explicitly, and keep the timer destruction in `disable`.

## Reuse Rules

- Publish an initial value before starting periodic observation.
- Publish later values only when the meaningful value changes.
- Keep one timer per service unless parallel behavior is required and documented.
- Destroy the timer in `disable` and set its variable to `nil`.
# Sample: Explicit Process Handles

Sample ID: `process/explicit-handles`

This sample is a complete package for writing a verified address through an explicit process handle.
The package also shows how to discover processes and the process that owns a window.

Do not run this against an unknown process. Supply a PID and address from a disposable target, and
confirm that the address is writable before invoking the action.

## File Tree

```text
example.process-handles/
	package.json
	lua/main.lua
```

## Complete Files

Create `example.process-handles/package.json`:

```json
{
	"id": "example.process-handles",
	"version": "1.0.0",
	"displayName": "Example Process Handles",
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
				"id": "write-explicit",
				"kind": "action",
				"runtime": "lua",
				"entryFile": "lua/main.lua",
				"entrySymbol": "writeExplicit",
				"parameterSchema": {
					"type": "object",
					"required": ["pid", "address", "value"],
					"properties": {
						"pid": { "type": "integer", "minimum": 1 },
						"address": { "type": "integer", "minimum": 1 },
						"value": { "type": "integer" }
					},
					"additionalProperties": false
				}
			},
			{
				"id": "discover-target",
				"kind": "action",
				"runtime": "lua",
				"entryFile": "lua/main.lua",
				"entrySymbol": "discoverTarget",
				"parameterSchema": {
					"type": "object",
					"required": ["processName", "windowClass", "windowTitle"],
					"properties": {
						"processName": { "type": "string" },
						"windowClass": { "type": "string" },
						"windowTitle": { "type": "string" }
					},
					"additionalProperties": false
				}
			}
		]
	}
}
```

Create `example.process-handles/lua/main.lua`:

```lua
function enable()
end

function disable()
end

function writeExplicit(parameters)
		if parameters.pid <= 0 then
				error("pid must be greater than zero")
		end

		local target = openProcess(parameters.pid)
		local before = target:readInteger(parameters.address)
		target:writeInteger(parameters.address, parameters.value)
		local after = target:readInteger(parameters.address)

		return {
				pid = target:getPid(),
				address = parameters.address,
				before = before,
				after = after
		}
end

function discoverTarget(parameters)
		local firstPid = findProcess(parameters.processName)
		local allPids = findProcesses(parameters.processName)
		local window = findWindow(parameters.windowClass, parameters.windowTitle)
		local windowPid = nil
		if window ~= nil then
				windowPid = getWindowProcessID(window)
		end

		return {
				firstPid = firstPid,
				processCount = #allPids,
				windowPid = windowPid,
				titlePid = findWindowProcessID(nil, parameters.windowTitle),
				classPid = findWindowProcessID(parameters.windowClass, nil)
		}
end
```

`openProcess(pid)` returns a handle whose methods operate on that PID. Bare functions such as
`writeInteger(address, value)` use the command's default attached process instead. This sample uses
the explicit handle for every read and write so the process choice is visible in the code.

## Run It

For `write-explicit`, provide values from your target:

```json
{
	"pid": 12345,
	"address": 140737488355328,
	"value": 77
}
```

The command result has the shape below; `before` depends on the target:

```json
{
	"pid": 12345,
	"address": 140737488355328,
	"before": 12,
	"after": 77
}
```

For `discover-target`, replace `ExampleGame.exe`, `ExampleGameWindow`, and `Example Game` with
values that exist on your machine:

```json
{
	"processName": "ExampleGame.exe",
	"windowClass": "ExampleGameWindow",
	"windowTitle": "Example Game"
}
```

A missing process or window returns `nil` for the corresponding result rather than identifying a
different target. Check those values before opening a handle or writing memory.

## Reuse Rules

- Use bare functions only when the command's attached target is intentional.
- Use `local target = openProcess(pid)` when operating on another process.
- Prefer `target:method(...)`; the dot-call form is also supported.
- Validate `pid` before calling `openProcess`; `0` is an invalid process ID.
# Sample: Package Enable, Action, and Disable

Sample ID: `package/enable-action-disable`

This sample shows two complete packages. The player package declares a dependency, enables after the
shared package, exposes one Lua action, and removes everything it owns during disable.

## File Tree

```text
example.shared/
	package.json
	lua/main.lua
example.player/
	package.json
	lua/main.lua
```

## Complete Files

## Shared Package

Create `example.shared/package.json`:

```json
{
	"id": "example.shared",
	"version": "1.0.0",
	"displayName": "Example Shared Contract",
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
			}
		]
	}
}
```

Create `example.shared/lua/main.lua`:

```lua
function enable()
		registerSymbol("example_shared_enabled", 0x1000)
end

function disable()
		unregisterSymbol("example_shared_enabled")
end

function sharedRuneBonus()
		return 42
end
```

`0x1000` is only a visible marker for this lifecycle sample. Replace it with an address that is
valid for your target, or remove the marker if the shared package does not need one.

## Player Package

Create `example.player/package.json`:

```json
{
	"id": "example.player",
	"version": "1.0.0",
	"displayName": "Example Player",
	"dependencies": [
		{ "id": "example.shared", "version": "^1.0.0" }
	],
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
				"id": "give-runes",
				"kind": "action",
				"runtime": "lua",
				"entryFile": "lua/main.lua",
				"entrySymbol": "giveRunes",
				"parameterSchema": {
					"type": "object",
					"required": ["amount"],
					"properties": {
						"amount": { "type": "integer", "minimum": 0 }
					},
					"additionalProperties": false
				}
			}
		]
	}
}
```

Create `example.player/lua/main.lua`:

```lua
function enable()
		local ok, errorMessage = autoAssemble([[
alloc(exampleRuneBuffer, 16)
registersymbol(exampleRuneBuffer)
]])
		if not ok then
				error(errorMessage or "could not allocate exampleRuneBuffer")
		end
end

function disable()
		unregisterSymbol("example_action_amount")
		local ok, errorMessage = autoAssemble([[
unregistersymbol(exampleRuneBuffer)
dealloc(exampleRuneBuffer)
]])
		if not ok then
				error(errorMessage or "could not release exampleRuneBuffer")
		end
end

function giveRunes(parameters)
		local amount = math.floor(tonumber(parameters.amount) or 0)
		local processHandle = openProcess(getDefaultPid())
		processHandle:writeInteger("exampleRuneBuffer", amount)
		registerSymbol("example_action_amount", processHandle:readInteger("exampleRuneBuffer"))
		return {
				ok = true,
				amount = amount,
				dependencyBonus = sharedRuneBonus()
		}
end
```

The allocation name and the symbol name are package resources: both are released by `disable`.

## Run It

Enable `example.player`. The host enables `example.shared` first because of the declared dependency.
Invoke `give-runes` with:

```json
{ "amount": 25 }
```

The command result is:

```json
{ "ok": true, "amount": 25, "dependencyBonus": 42 }
```

The action also writes `25` to the allocated `exampleRuneBuffer`. Disable the player package and
verify that `example_action_amount`, `exampleRuneBuffer`, and `example_shared_enabled` are gone.

## Replace Before Use

The marker address `0x1000` is target-specific. The allocated buffer is safe only as an example
resource and does not grant the package a gameplay effect. Use a verified target address and an
explicit write policy before turning this into a real trainer action.

## Reuse Rules

- Declare every dependency in the manifest; do not rely on load order.
- Validate action arguments with `parameterSchema`.
- Treat every symbol and allocation created by `enable` as an owned resource.
- Reverse enable work during `disable`, even when an action has never run.
# Sample: Lua Multi-File Require

Sample ID: `lua/multifile-require`

This sample is a complete package that loads one package-local module from `lua/main.lua`, loads it
again during an action, and reports whether the second call used the same cached module value.

## File Tree

```text
example.multifile/
	package.json
	lua/main.lua
	lua/helpers/inventory.lua
```

## Complete Files

Create `example.multifile/package.json`:

```json
{
	"id": "example.multifile",
	"version": "1.0.0",
	"displayName": "Example Lua Modules",
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
				"id": "inspect-inventory",
				"kind": "action",
				"runtime": "lua",
				"entryFile": "lua/main.lua",
				"entrySymbol": "inspectInventory",
				"parameterSchema": {
					"type": "object",
					"required": ["amount"],
					"properties": {
						"amount": { "type": "integer" }
					},
					"additionalProperties": false
				}
			}
		]
	}
}
```

Create `example.multifile/lua/main.lua`:

```lua
local inventory = require("helpers.inventory")

function enable()
end

function disable()
end

function inspectInventory(parameters)
		local secondReference = require("helpers.inventory")
		return {
				sameModule = inventory == secondReference,
				moduleLoadCount = inventory.loadCount(),
				total = inventory.add(parameters.amount)
		}
end
```

Create `example.multifile/lua/helpers/inventory.lua`:

```lua
local loadCount = 1

return {
		add = function(amount)
				return amount + 31
		end,
		loadCount = function()
				return loadCount
		end
}
```

`require("helpers.inventory")` maps to the package-local file `lua/helpers/inventory.lua`. The
module is evaluated once for the current Lua script context. Later calls with the same module ID
return the cached table.

## Run It

Enable the package and invoke `inspect-inventory` with:

```json
{ "amount": 5 }
```

The command result is:

```json
{ "sameModule": true, "moduleLoadCount": 1, "total": 36 }
```

The cache belongs to the current script context. A fresh context after package teardown evaluates
the module again. This is different from a dependency: another package must be declared in
`dependencies`; it must not be reached with a cross-package `require`.

## Reuse Rules

- Use dot-separated module IDs only, such as `helpers.inventory`.
- Do not use absolute paths, path separators, `..`, or a `.lua` suffix in a module ID.
- Use `require` only for files within the current package.
- Consume another package through a declared dependency contract, not cross-package `require`.
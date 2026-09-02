# Sample: Auto Assembler Allocation Action Pair

Sample ID: `aa/allocation-action-pair`

This sample is a complete package with two Auto Assembler actions: one allocates a named block and
the other releases it. It demonstrates resource pairing only. Allocating memory alone is not a hook
and does not change gameplay.

## File Tree

```text
example.aa-pair/
	package.json
	lua/main.lua
	aa/enable-cave.aa
	aa/disable-cave.aa
```

## Complete Files

Create `example.aa-pair/package.json`:

```json
{
	"id": "example.aa-pair",
	"version": "1.0.0",
	"displayName": "Example AA Allocation",
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
				"id": "allocate-cave",
				"kind": "action",
				"runtime": "aa",
				"entryFile": "aa/enable-cave.aa"
			},
			{
				"id": "release-cave",
				"kind": "action",
				"runtime": "aa",
				"entryFile": "aa/disable-cave.aa"
			}
		]
	}
}
```

Create `example.aa-pair/lua/main.lua`:

```lua
function enable()
end

function disable()
end
```

Create `example.aa-pair/aa/enable-cave.aa`:

```asm
alloc(exampleCave, 32)
registersymbol(exampleCave)
```

Create `example.aa-pair/aa/disable-cave.aa`:

```asm
unregistersymbol(exampleCave)
dealloc(exampleCave)
```

The name `exampleCave` is shared by both files. That shared name is the resource contract between
the two actions. The Lua `enable` and `disable` functions are required package lifecycle entries;
they do not replace the explicit AA allocation and release actions.

## Run It

Invoke `allocate-cave` once. The command succeeds and creates a named 32-byte allocation in the
current process. Invoke `release-cave` afterward. The command succeeds and removes the symbol and
allocation.

Do not invoke `release-cave` before allocation, and do not invoke `allocate-cave` repeatedly without
releasing the existing allocation. The exact behavior of duplicate allocation names depends on the
Auto Assembler runtime and should not be used as an ownership strategy.

## Reuse Rules

- Declare Auto Assembler runnables with `runtime: "aa"` and an `entryFile`.
- Declare Lua lifecycle runnables because Auto Assembler is action-oriented in this package model.
- Do not use Auto Assembler for `enable`, `disable`, or a continuously running service.
- Pair every `alloc` with a deterministic `dealloc` action or a Lua `disable` cleanup path.
- Build real hooks on top of validated scans and restoration logic; allocation alone is not a hook.
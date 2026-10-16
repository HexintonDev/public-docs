# Converting CE Game-Call Scripts

Status: current conversion guidance for the shipped engine.

Many Cheat Engine (CE) tables call the game's own functions instead of writing a value: item and
equipment spawners, warps and teleports, NPC or enemy spawns, chain events, flag setters and game
menus. Host Lua implements these with [`executeCode`, `executeCodeEx` and `allocateMemory`](lua-game-calls-and-worker-api.md).
Convert them; do not drop them because they call game code.

## Recognize the source shape

A typical CE spawner has three parts:

1. An AA script that scans for the game function, allocates a stub and registers a symbol such as
   `itemgib`. The stub loads the game's registers and calls the function.
2. A memory record with a dropdown list of item IDs and names, or a Lua table of them.
3. Lua that fills a request buffer and calls the stub:

```lua
local itembuf = allocateMemory(4 + 16)
writeInteger(itembuf, 1)
writeInteger(itembuf + 4, itemId)
executeCode("itemgib", itembuf)
```

Direct calls use `executeCodeEx(0, 100, address, ...)` with the game function's own arguments. CE
scripts often wrap address-list access in `synchronize`, and start delayed work with
`createNativeThread` or `createTimer(getMainForm())`.

## Convert it

**Keep the call code.** Port the AA that scans, allocates and registers the stub into the package's
prepare/`enable` step, or run it with `autoAssemble` from Lua. Keep the stub's instructions, AOB and
offsets exactly. Release its symbols and allocations in `disable`. The call itself keeps the same
function name or address, argument order and buffer layout as the source.

**Make the list data.** Extract the dropdown list or item table with its IDs and names unchanged; do
not retype or summarize it. A long catalogue belongs in a [custom interface](../application/custom-interfaces.md)
with search, category filters and a quantity input; a few fixed choices can be ordinary controls.
The interface sends the selected ID and quantity to one Lua action.

**Write one action per operation.** The action receives its parameters, writes the request buffer
exactly as the source did, calls the function with a bounded timeout and raises on failure:

```lua
local request

function spawnItem(params)
  if not request then
    local address, message = allocateMemory(0x100)
    if not address then error(message) end
    request = address
  end
  writeInteger(request, 1)
  writeInteger(request + 4, params.itemId)
  writeInteger(request + 8, params.quantity)
  writeInteger(request + 12, 0)
  writeInteger(request + 16, -1)
  local result, message = executeCode("itemgib", request, 1000)
  if result == nil then
    if message == "Execution timeout" then request = nil end  -- may still be in use
    error(message)
  end
end

function disable()
  if request then deAlloc(request) end
  request = nil
  -- also unregister the stub's symbols and release its allocation
end
```

A source loop that spawns every entry of a list becomes one action that loops over the IDs it
receives. Keep CE's short `sleep` between calls only when the source needs it.

**Replace CE-only scaffolding.**

| CE source | Package |
| --- | --- |
| `synchronize(function() ... end)` | The body inline, or keep `synchronize`; it calls the function directly |
| `getAddressList()`, memory records, `DropDownValue`, `record.Active` | Extracted data, package state and actions |
| A record that runs once and switches itself off with a timer | A button or input action |
| `createNativeThread(fn)` | The body in the action, or `createTimer(ms, fn)` for a delay |
| A thread that loops with `sleep` | A repeating timer that `disable` destroys |
| `"$process+7FF290"` | `process .. "+0x7FF290"`, using the attached module name |
| `executeCodeEx(0, 100, addr, "", x)` | The same call; `""` passes a pointer to an empty string |

Check CE argument types when porting `executeCodeEx`: integers stay integers, non-integer numbers
become 32-bit floats and strings are copied as text. A float parameter written as `1` in CE needs
`1.0` in Lua 5.4.

## Test it

An integration test cannot run the real game function. Seed a small stand-in function in a fixture
module that records the request it receives and returns a value, then run the production action
through its actual control. The Engine Essentials
[game call with executeCode](engine-essentials.md#example-game-call-with-executecode) example does
this. A pass proves the buffer layout, the call plumbing and repeatability; it does not prove the
game function's signature or the item IDs.

## When a feature cannot be converted

Report a specific, checkable reason, for example:

- the source's AOB for the game function is absent from the shipped executable;
- the stub depends on a register or structure that the source never initializes;
- the function must run on the game thread and the source reached it only through a hook that has
  not been ported yet.

"The engine has no `executeCode`" or "Lua has no threads" is not a reason. Those features exist as
described above.

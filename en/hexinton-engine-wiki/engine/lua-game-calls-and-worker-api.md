# Lua Game Calls, Allocation and Worker Helpers API

Status: current public API reference for host Lua (`runtime: "lua"`).

These globals are compatible with the Cheat Engine (CE) functions of the same names. They call game
functions, allocate target memory, and schedule Lua work on the runtime worker. To convert a CE
item spawner, warp, NPC spawn or event trigger, see
[converting CE game-call scripts](converting-ce-game-calls.md).

| Function | Result |
| --- | --- |
| [`executeCode(address, parameter?, timeout?)`](#executecode) | Call a function with one pointer-sized argument |
| [`executeCodeEx(callmethod, timeout, address, ...)`](#executecodeex) | Call a function with integer, float and string arguments |
| [`allocateMemory(size, base?, protection?)`](#allocatememory) | Allocate caller-owned target memory |
| [`deAlloc(address, size?)`](#dealloc) | Release an allocation |
| [`readByte(address)`, `writeByte(address, value)`](#readbyte-and-writebyte) | Read or write one byte (see the memory API) |
| [`synchronize(fn, ...)`](#synchronize) | Call `fn` now on the Lua worker |
| [`queue(fn, ...)`](#queue) | Call `fn` on a later Lua worker turn |
| [`isKeyPressed(vkey)`](#iskeypressed) | Current physical key state |

## How a game call runs

The engine writes a small call stub into the target, starts it with a new remote thread, and waits
for the function to return. The function runs on that new thread, not on the game's main thread.
That is the same thread model as CE, so a CE table that already calls the function this way is good
evidence that the call is safe. No Lua state or Lua callback is visible to the game code.

The calling Lua runnable waits until the function returns or the timeout expires. All host Lua for
a game shares one worker, so a long call delays timers, hotkeys and other actions. Use a bounded
timeout in actions, such as CE's usual `100` or `1000` milliseconds. Never pass `nil` or `-1`
(wait forever) from a timer callback.

Both call functions return failures instead of raising them:

| Outcome | Return values |
| --- | --- |
| Function returned | Integer return register (`RAX` on x64, `EAX` on x86) |
| Timeout expired | `nil, "Execution timeout"` |
| Bad argument, unresolved address, thread launch failure | `nil, message` |
| Thread ended without returning (for example, the game function crashed the thread) | `nil, message` |

Check the first value and raise the message from the action. A return value of `0` is a successful
call that returned zero.

After a timeout the remote thread may still be running. The engine leaves its own call stub and
copied strings in the target permanently, because freeing them while the thread runs would be unsafe.
Treat any buffer you passed in the same way: abandon it instead of calling `deAlloc`.

## `executeCode`

```lua
executeCode(address, parameter?, timeout?) -> result | nil, message
```

| Parameter | Meaning |
| --- | --- |
| `address` | Integer address or address expression, including registered symbols such as an AA `registersymbol(itemgib)` |
| `parameter` | Optional integer or address expression, passed as the single argument (`RCX` on x64, the stack on x86). Default `0` |
| `timeout` | Milliseconds. `nil` or `-1` waits without a limit; `0` returns at once unless the call already finished |

A string `parameter` is resolved as an address, not copied as text. Use `executeCodeEx` to pass
text. `executeCode(nil)` returns `nil` without calling anything.

```lua
local request = allocateMemory(0x100)
writeInteger(request, itemId)
writeInteger(request + 4, quantity)
local result, message = executeCode("itemgib", request, 1000)
if result == nil then error(message) end
```

## `executeCodeEx`

```lua
executeCodeEx(callmethod, timeout, address, ...) -> result | nil, message
```

The timeout comes **before** the address, unlike `executeCode`. `callmethod` is `0` (stdcall) or `1`
(cdecl). It matters only for 32-bit targets, where cdecl makes the stub remove the arguments after
the call. 64-bit targets always use the Windows x64 convention.

Each extra argument is converted by its Lua type:

| Lua value | Passed as |
| --- | --- |
| Integer, such as `5` or `0x1400` | Pointer-sized integer (`RCX`, `RDX`, `R8`, `R9`, then the stack on x64) |
| Float, such as `1.5` or `2.0` | 32-bit single-precision float (`XMM0`-`XMM3` for the first four x64 arguments, then a stack slot) |
| String | Pointer to a NUL-terminated copy of the bytes in target memory |

Strings are literal data: `""` passes a pointer to an empty string, and `"eldenring.exe"` passes
that text, never its resolved address. To pass a symbol's address, resolve it first with
`getAddress`. Text is copied byte-for-byte; for a UTF-16 argument, write it with
`writeString(buffer, text, true, true)` and pass `buffer`. Doubles and CE's `{type=..., value=...}`
argument tables are not supported and return `nil, message`. At most 256 arguments are accepted.

Lua 5.4 distinguishes `1` (integer) from `1.0` (float). A game function expecting a float needs a
float value: write `1.0`, not `1`.

```lua
local warp = getAddress(process .. "+0x7FF290")
local result, message = executeCodeEx(0, 1000, warp, "", destinationId)
if result == nil then error(message) end
```

## `allocateMemory`

```lua
allocateMemory(size, base?, protection?) -> address | nil, message
```

Allocates `size` bytes in the target. `base` is an optional address or expression to allocate near:
the block is placed within 2 GB of it, which keeps a 32-bit relative jump or call from that code in
range. `nil` or `0` lets the system choose. `protection` takes a Win32 `PAGE_*` value and defaults to
`PAGE_EXECUTE_READWRITE` (`0x40`). Supported values are `0x01`, `0x02`, `0x04`, `0x08`, `0x10`,
`0x20`, `0x40` and `0x80`, optionally combined with `PAGE_GUARD` (`0x100`). Other modifiers return
`nil, message` rather than silently dropping protection.

The allocation belongs to the caller. It is **not** released when a runnable returns, a service
stops or the package is disabled. Release it from `disable` with `deAlloc`, unless a call that used
it timed out.

## `deAlloc`

```lua
deAlloc(address, size?) -> boolean
```

Releases the whole allocation that starts at `address` and returns `true`, or `false` when it could
not be released. As in CE, `size` is accepted but does not limit the release.

## `readByte` and `writeByte`

```lua
readByte(address, signed?) -> integer | nil
writeByte(address, value) -> boolean
```

Read or write one byte. Like the other [typed memory functions](lua-memory-api.md#failures), a
failed read returns `nil` and a failed write returns `false`.

## `synchronize`

```lua
synchronize(fn, ...) -> fn's return values
```

Calls `fn(...)` immediately and returns its results. CE uses `synchronize` to reach its GUI thread;
host Lua has no GUI thread, and every package function already runs on the one Lua worker. Errors
raised by `fn` propagate to the caller. A converted script can call `synchronize` or simply run the
body inline.

## `queue`

```lua
queue(fn, ...) -> nothing
```

Runs `fn(...)` on a later turn of the Lua worker, after the current runnable returns. Arguments,
including `nil` values and tables, are retained as they were. The worker runs one queued callback
per turn, so a callback that queues another cannot recurse. The callback belongs to the package and
service that queued it: it is cancelled if the script is destroyed or the service stops first. An
error in a callback is logged and does not stop later callbacks.

## `isKeyPressed`

```lua
isKeyPressed(vkey) -> boolean
```

Returns whether Windows virtual key `vkey` (`0`-`255`) is held now. It reads the global keyboard
state, even when the game window is not in the foreground. For player-controlled input in a
package, declare [native hotkeys](native-hotkeys.md) instead: they are rebindable and are sampled
only while the game is in the foreground.

## Threads and timers

There is no `createNativeThread` or Lua `createThread`. Rewrite them by what the thread did:

| CE pattern | Host Lua |
| --- | --- |
| `createNativeThread(fn)` that runs once | Run the body in the action, or `createTimer(ms, fn)` for a one-shot delay |
| Thread loop with `sleep` | Repeating timer: `createTimer(false)`, then `Interval`, `OnTimer`, `Enabled = true` |
| `createTimer(getMainForm())` | `createTimer(false)` |
| `synchronize(function() ... end)` | The body itself, or keep `synchronize` |

`sleep` blocks the shared worker, so keep it short. See [`createTimer`](lua-runtime-utilities-api.md#createtimer)
and the [thread model](thread-model.md).

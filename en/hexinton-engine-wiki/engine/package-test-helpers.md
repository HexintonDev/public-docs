# Package Test Helpers v1

Studio supplies small Lua and JavaScript helper modules to declared package tests. They provide
assertions, temporary mocks and cleanup scopes; Lua also provides byte snapshot/restoration scopes.
They reduce boilerplate when Hex Assistant writes tests. They do not generate suites, choose game
addresses, Apply packages, invoke lifecycle hooks, or prove that a real game is compatible.

See [Testing](testing.md) for `tests/studio.tests.json`, the owned integration target and results.
Existing declarations remain `schemaVersion: 1`. The helper context is an additive contract.

## Loading and lifetime

Every Lua, JS or command case receives these fields in its existing context:

| Field | Contract |
| --- | --- |
| `helpers.version` | Integer `1`, the API contract described here. |
| `helpers.luaFile` | Absolute path to the Lua module. Load with `dofile`. |
| `helpers.jsModule` | Absolute `file:` URL to the ES module. Load with dynamic `import`. |

```lua
return function(ctx)
  assert(ctx.helpers and ctx.helpers.version == 1, 'Test helpers v1 required')
  local h = dofile(ctx.helpers.luaFile)
  h.equal(h.version, 1)
end
```

```javascript
export async function runTest(ctx) {
  if (ctx.helpers?.version !== 1) throw Error('Test helpers v1 required');
  const h = await import(ctx.helpers.jsModule);
  h.equal(h.version, 1);
}
```

Modules are shipped with Studio and materialized inside each worker's disposable package copy.
Paths are opaque, unique to the case and valid only while it runs. Do not persist them, build static
imports from them, or add the generated folder to a package. Saved files and their captured revision
are unchanged. Normal production runnables do not receive this context or an automatic helper global.
Older Studio builds without this feature do not have `helpers`; fail with the prerequisite above
rather than claim the case passed. v1 behavior is kept under that version; a future breaking contract
requires a new version. Custom helper code can still be kept in the package normally.

## Assertions

| API (Lua and JS) | Behavior |
| --- | --- |
| `equal(actual, expected, message?)` | Throws on unequal values; includes expected and actual in the error. Lua uses `==`; JS uses `Object.is`. Table/object equality is identity, not recursive comparison. JS treats `NaN` as equal to itself and distinguishes `-0` from `0`. |
| `ok(value, message?)` | Throws unless the value is truthy under that language's rules. Lua considers `0` and `''` truthy; JS does not. |
| `throws(fn, contains?)` | Requires `fn` to fail; optionally checks a literal, case-sensitive substring of its error text. Returns the original error. No error, or a substring mismatch, is a test failure. JS also awaits/requires rejected promises. |

`message` and `contains`, when supplied, are strings. **JS `throws` returns a promise and must be
awaited**, including when the callback throws synchronously. Successful Lua assertions return no
values; successful JS `equal`/`ok` return `undefined`. There is no assertion count or deep-equality API.
A body returning `false` is not an exception for `throws`; the runner separately treats a test entry
returning `false` as a failed case.

For a production Lua module at `lua/health.lua` exporting `clamp(value, maximum)` and rejecting
nonnumeric input, a unit test can be:

```lua
return function(ctx)
  local h = dofile(ctx.helpers.luaFile)
  local health = dofile(ctx.packageRoot .. '/lua/health.lua')
  h.equal(health.clamp(150, 100), 100, 'clamp above max')
  h.equal(health.clamp(-10, 100), 0, 'clamp below zero')
  h.equal(health.clamp(50, 100), 50, 'valid value')
  h.throws(function() health.clamp('bad', 100) end, 'numeric')
end
```

For equivalent production JS exports in `math.mjs`, `tests/unit.mjs` can use:

```javascript
import { clamp } from '../math.mjs';
export async function runTest(ctx) {
  const h = await import(ctx.helpers.jsModule);
  h.equal(clamp(150, 100), 100, 'clamp above max');
  h.equal(clamp(-10, 100), 0, 'clamp below zero');
  await h.throws(() => clamp('bad', 100), 'numeric');
}
```

These examples import the implementation under test. Reimplementing `clamp` inside a test would
test the duplicate rather than the package.

## Cleanup scopes

`withCleanup(body)` calls `body(defer)`. Register a callback with `defer(fn)` as soon as a resource
needs cleanup, preferably **before** an operation that can partially acquire or modify it.

On normal return or an ordinary thrown error, every registered cleanup is attempted once, in
reverse registration order. Registration closes when the body finishes; retaining `defer` and using
it later throws. A cleanup callback signals failure by throwing, not by returning `false`.

If the body fails and cleanup succeeds, its original error is rethrown. If any cleanup fails, the
error contains the original body failure (if any), `Cleanup failed:`, and all cleanup failures in
attempt order. Other cleanups still run. Successful scopes return the body's value; Lua preserves
all return values, including nil slots. **JS scopes await the body and callbacks, return a promise
and must be awaited.** Avoid detached work that continues after the scope has closed.

Lua example for a production module whose `enable(statsAddress, fail)` writes memory and whose
idempotent `disable()` restores it:

```lua
return function(ctx)
  local h = dofile(ctx.helpers.luaFile)
  local health = dofile(ctx.packageRoot .. '/lua/health.lua')
  local stats = ctx.fixture.statsAddress
  for _ = 1, 3 do
    h.withCleanup(function(defer)
      defer(health.disable) -- before enable: cleanup also runs if enable partially fails
      health.enable(stats, false)
      h.equal(readInteger(stats), 50, 'enabled effect')
    end)
    h.equal(readInteger(stats), 100, 'disable restored original health')
    health.disable()
    h.equal(readInteger(stats), 100, 'repeated disable is harmless')
  end
  h.throws(function()
    h.withCleanup(function(defer)
      defer(health.disable)
      health.enable(stats, true) -- this example module throws after its first write
    end)
  end, 'enable failed after writing')
  h.equal(readInteger(stats), 100, 'failed enable cleaned up')
end
```

Adapt the call signatures to the actual package; these are example module exports, not universal
host functions. The runner does not call production enable/disable automatically. Assert the effect
and the cleanup separately. For timers/hooks/services, register the actual destroy/unsubscribe
operation and assert the resulting state; helpers do not discover resources for you.

JS example using a production subscription handle:

```javascript
await h.withCleanup(async defer => {
  const subscription = await service.subscribe();
  defer(() => subscription.dispose());
  h.equal(await subscription.next(), expectedValue);
});
```

Here cleanup is registered immediately after acquisition. If `subscribe()` itself can fail after
acquiring something, its implementation must expose/perform that partial cleanup or the test must
register an appropriate owner cleanup before invoking it.

## Temporary mocks

`withMock(target, key, replacement, body)` temporarily replaces one slot, calls `body()` and restores
the original slot on normal return or ordinary error. It forwards the body's result and uses the
cleanup error rules above. Nested mocks restore to the enclosing mock first, then to the original.
A replacement of Lua `nil` removes the raw slot temporarily; JS `undefined` remains an own property.

Lua accepts a table and a non-nil valid table key. It uses `rawget`/`rawset`, preserving an absent raw
slot and leaving metatables untouched. An inherited value is visible again afterward. Prefer a
dependency table or deliberately mock a host function in the test's `_G` when production code loaded
with `dofile` uses that environment:

```lua
local written = 0
h.withMock(_G, 'readInteger', function(address)
  if address == 104 then return 30 end
  h.equal(address, 100, 'only expected reads')
  return written
end, function()
  h.withMock(_G, 'writeInteger', function(address, value)
    h.equal(address, 100, 'only expected writes')
    written = value
    return true
  end, function()
    h.equal(health.setHealth(100, 50), 30)
    h.equal(written, 30)
  end)
end)
```

Mock **every** host operation that would access real memory in a unit test, including writes
and reads of the written value as above. A mock does not redirect other host functions or
make a live address safe. Module load-time caches must be mocked before importing that module.

JS accepts an object/function and string/symbol key. It restores the exact original own-property
descriptor, including enumerability, or deletes a formerly absent own slot. Accessors and own
non-writable/non-configurable properties are rejected before invoking the body; getters are not
evaluated. Adding a slot to a non-extensible object also fails. Imported ES module namespaces are
not writable dependency objects. **Await JS `withMock`**:

```javascript
await h.withMock(dependency, 'lookup', async () => 123, async () => {
  h.equal(await production.resolve(dependency), 123);
});
```

Use ordinary dependency objects. Proxies or a body that freezes/redefines the mocked slot can
prevent restoration; this is reported as cleanup failure, not silently accepted.

## Lua memory scopes

`withMemory(ranges, body)` uses the native Lua `readBytes(address, size, true)` and
`writeBytes(address, bytes)` functions. `ranges` is a nonempty dense array of
`{ address = integer, size = integer }`. Addresses must be positive, lengths 1–65536 bytes,
and the total snapshot size at most 65536 bytes; an overflowing address range is rejected.
Overlapping ranges are supported: every snapshot is taken before the body starts.

All ranges must be captured successfully before calling `body()`. Capture failure means the body
does not run and no restoration writes occur. After ordinary success or failure, ranges restore
in reverse order. Each write is followed by a read verifying every original byte. A thrown write,
returned `false`, short/invalid read, or byte mismatch becomes a cleanup failure. Remaining ranges
still receive restoration attempts. The body cannot substitute different host functions for this
cleanup: the scope retains the read/write functions it captured at entry.

For the owned integration fixture and a production `setHealth(stats, value)`:

```lua
return function(ctx)
  local h = dofile(ctx.helpers.luaFile)
  local health = dofile(ctx.packageRoot .. '/lua/health.lua')
  local stats = ctx.fixture.statsAddress
  h.equal(readPointer(ctx.fixture.signatureAddress + 16), ctx.fixture.playerAddress)
  h.equal(readPointer(ctx.fixture.playerAddress + 32), stats)
  h.withMemory({ { address = stats, size = 8 } }, function()
    h.equal(health.setHealth(stats, 150), 100, 'clamped write')
    h.equal(health.setHealth(stats, 50), 50, 'requested write')
    h.equal(readInteger(stats + 4), 100, 'maximum is unchanged')
  end)
  h.equal(readInteger(stats), 100, 'original health restored')
end
```

Declare every range the code may modify, with the correct widths. This helper neither finds those
ranges nor restricts writes outside them. Avoid writes beyond the fixture allocation. Its known
pointers/offsets remain fixture contracts; this test cannot prove real-game AOBs or offsets.
There is no JS memory helper in v1; JS can use `withCleanup` with its actual declared APIs.

## Command cases and failures

Command cases read the same context via the JSON file in `HEXMOD_TEST_CONTEXT` or the
`{fixtureManifest}` placeholder. A Node ES module can use:

```javascript
import { readFileSync } from 'node:fs';
const ctx = JSON.parse(readFileSync(process.env.HEXMOD_TEST_CONTEXT, 'utf8'));
const h = await import(ctx.helpers.jsModule);
h.equal(h.version, 1);
```

The JS helper has no Node/DOM dependency. The Lua memory helper requires the native Lua host; an
external Lua interpreter does not acquire those host functions merely by loading the file. Framework
installation, TypeScript compilation and framework-specific assertion/fixture APIs remain explicit.

Uncaught helper assertions and body/cleanup failures follow the existing Tests/assistant result
path with their original text. Helpers do not convert failure to a passing boolean, skip a suite,
or enforce coverage. Generated helper files are not navigation targets in the saved package tree.

**Cleanup scopes cannot execute after timeout, Stop, process crash or forced termination.** The
runner terminates its owned worker/target tree and removes disposable files; it does not promise
Lua/JS finally callbacks in a killed process. External side effects outside the owned target still
need their own recovery. Unit mocks are not an integration check, and fixture tests are not a
real-game compatibility check. Keep meaningful unit and integration suites and report blocked
prerequisites accurately.

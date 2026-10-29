# Custom Interface API

Status: current protocol v1 reference.

The current host loads one package HTML document in a sandbox and transfers a private `MessagePort`.
The page can use only the aliases declared in its surface manifest. A page sends a binding alias such
as `health.set`; it never sends a package runnable ID such as `set-health` directly.

Author pages should import the supplied `SurfaceSdkClient` module as shown in the
[quickstart](custom-interface-quickstart.md#reuse-the-supplied-sdk). The wire envelopes below explain
what that existing SDK and the host exchange; they do not require another inline request client.

The manifest uses `type: "viewModel"` for a changing value. The wire message is named
`feed.updated` because the host delivers ViewModel changes through a Service-backed feed. There is no
current `type: "feed"` surface binding or `stateFeed` property in this contract.

## Bootstrap

After the document loads, the host sends one global message and transfers one port:

```ts
interface SurfaceBootstrapMessageV1 {
  type: "hexmod.surface.bootstrap";
  payload: {
    protocolVersion: 1;
    supportedProtocolVersions: number[];
    surfaceInstanceId: string;
    packageId: string;
    packageVersion: string;
    surfaceId: string;
    environment: {
      mode?: "recording"; // Absent in ordinary live hosts.
      theme: "dark" | "light";
      locale: string;
      reducedMotion: boolean;
      placement: "compact" | "panel" | "app";
      platform: "desktop" | "web" | "mobile";
    };
    capabilities: Record<string, {
      type: "command" | "query" | "viewModel";
    }>;
  };
}
```

Use the global `message` event only to receive this bootstrap. Normal traffic must use the transferred
port. The frame has an opaque origin, so origin-string checks are not the authorization mechanism.

Install the port listener before calling `surface.ready`. Call it within five seconds, then wait for
`lifecycle.changed` with `state: "active"`. Requests sent before ready or while suspended are
rejected; they are not queued.

## Recording mode

Inside the isolated Save Mods editor, `sdk.environment.getSnapshot().mode` is `"recording"`.
The field is absent in ordinary live hosts. The normal SDK methods still use the private port;
the recording host handles them without executing game commands, game queries, or live Service
subscriptions. Binding responses come from the optional
[`recording.result` manifest declaration](custom-interface-manifest.md#isolated-save-mods-recording).
Missing declarations do not become successful null responses.

The declared results are initial preview data. They are not live values, and a fixed query response
does not automatically implement a game-backed workflow. Keep local state inside the page when
recording mode needs it, for example updating a selected value after a recorded command:

```js
let displayedHealth = 100;
const recording = sdk.environment.getSnapshot().mode === "recording";
async function setHealth(health) {
  await sdk.commands.execute("health.set", { health });
  if (recording) {
    displayedHealth = health;
    render();
  }
  // The live page continues to get current values through its existing ViewModel subscription.
}
```

Use the declared initial response and packaged catalogue data to initialize the page. Avoid showing
recorded command completion as proof that the game changed. Live hosts ignore `recording.result`
and continue resolving the same binding aliases to their normal runnables and feeds.

## Localized Text

The host does not translate text inside a custom interface. The page owns its text and picks it from
`environment.locale`. Keep one flat JSON file per launch locale next to the page and give every file
the same keys:

```text
frontend/<surface-id>/index.html
frontend/<surface-id>/locales/en.json      {"title": "Item Spawner", "give": "Give"}
frontend/<surface-id>/locales/zh-Hans.json {"title": "物品生成器", "give": "给予"}
frontend/<surface-id>/locales/ja.json, ko.json, de.json, es.json
```

```js
const LOCALES = ["en", "zh-Hans", "ja", "ko", "de", "es"];
let text = {};
async function loadText(locale) {
  const code = LOCALES.includes(locale) ? locale : "en";
  const response = await fetch(`./locales/${code}.json`).catch(() => null);
  // A package may ship only some locales; fall back to English when a file is missing.
  text = await (response?.ok ? response : await fetch("./locales/en.json")).json();
  render();
}
// sdk is the SurfaceSdkClient created from the bootstrap message.
loadText(sdk.environment.getSnapshot().locale);
sdk.environment.subscribe((environment) => loadText(environment.locale));
```

Ship `en.json` always; the other five are optional for your packages and required for HexMod's converted packages ([package format](../../start-here/getting-started/package-format.md#user-facing-text-six-launch-locales)).
Catalogue data such as item names keeps its source names; translate the interface around it.

## Message Envelope

Every normal message uses this shape:

```ts
interface SurfaceEnvelopeV1<TPayload> {
  protocolVersion: 1;
  type: string;
  requestId?: string;
  payload: TPayload;
}
```

Commands, queries, and ViewModel subscriptions require a non-empty request ID unique for the live
surface instance. `surface.ready`, `lifecycle.setDirty`, and `surface.reportError` do not use one.
The maximum serialized message size is 256 KiB in either direction. Arguments and results must be
finite JSON values.

## Ready and Dirty State

```js
port.postMessage({
  protocolVersion: 1,
  type: "surface.ready",
  payload: {}
});

port.postMessage({
  protocolVersion: 1,
  type: "lifecycle.setDirty",
  payload: { isDirty: true }
});
```

Send `surface.ready` once after installing message and lifecycle listeners. Dirty state describes
unsaved local draft input; it does not mean that the game has uncommitted native work.

## Execute a Command

Suppose the manifest contains this binding:

```json
{
  "bindings": {
    "health.set": {
      "type": "command",
      "action": "set-health"
    }
  }
}
```

The page sends the surface alias and the arguments expected by the action's `parameterSchema`:

```js
port.postMessage({
  protocolVersion: 1,
  type: "command.execute",
  requestId: "health-command-1",
  payload: {
    binding: "health.set",
    arguments: { health: 120 }
  }
});
```

The host resolves `health.set` to the package-local `set-health` action, validates `{ health: 120 }`,
executes the Lua entry point, and returns one correlated completion. The page must not replace
`binding` with `runnableId` and must not send `parameters` instead of `arguments`.

At most four commands may be pending for one surface. A command result confirms one requested
operation. It is not the current game value; use a ViewModel subscription when the value can change
outside the command.

## Execute a Query

Queries use the same alias rule and can be cancelled:

```js
port.postMessage({
  protocolVersion: 1,
  type: "query.execute",
  requestId: "inventory-query-1",
  payload: {
    binding: "inventory.search",
    arguments: { search: "repair" }
  }
});

port.postMessage({
  protocolVersion: 1,
  type: "query.cancel",
  requestId: "inventory-query-1",
  payload: {}
});
```

Use a query for one transient read or search. Cancellation settles the local request with
`query_cancelled` where the host adapter supports cancellation. It does not undo a runtime operation
that has already become uncancellable.

## Subscribe to a ViewModel

Suppose the manifest contains:

```json
{
  "bindings": {
    "health.current": {
      "type": "viewModel",
      "viewModel": "health"
    }
  }
}
```

Subscribe using the surface alias, not the Service runnable ID:

```js
port.postMessage({
  protocolVersion: 1,
  type: "feed.subscribe",
  requestId: "health-view-1",
  payload: { binding: "health.current" }
});
```

The host sends the latest projected value when one exists and later sends updates:

```json
{
  "protocolVersion": 1,
  "type": "feed.updated",
  "requestId": "health-view-1",
  "payload": {
    "value": { "health": 80, "maxHealth": 100 },
    "revision": 3
  }
}
```

Apply only updates for the current subscription and accept only a revision newer than the last
accepted revision. Do not synthesize a revision from a command result or local draft.

Unsubscribe with the same subscription request ID:

```js
port.postMessage({
  protocolVersion: 1,
  type: "feed.unsubscribe",
  requestId: "health-view-1",
  payload: {}
});
```

## Host Responses

Successful command and query requests receive:

```json
{
  "protocolVersion": 1,
  "type": "request.succeeded",
  "requestId": "health-command-1",
  "payload": { "result": { "health": 120, "maxHealth": 100 } }
}
```

Failed requests receive a structured error:

```json
{
  "protocolVersion": 1,
  "type": "request.failed",
  "requestId": "health-command-1",
  "payload": {
    "error": {
      "severity": "error",
      "domain": "surface",
      "code": "capability_denied",
      "message": "The custom surface does not have this capability.",
      "details": {}
    }
  }
}
```

Branch on stable `domain` and `code`, not on human-facing `message`. Preserve local drafts after a
failure and show a task-oriented message.

## Lifecycle Changes

```json
{
  "protocolVersion": 1,
  "type": "lifecycle.changed",
  "payload": { "state": "active" }
}
```

- `active`: declared capabilities may be used.
- `suspended`: stop privileged work; current ViewModel subscriptions are removed.
- `disposed`: settle local requests, remove listeners, close the port, and release resources.

When the surface becomes active again, create fresh ViewModel subscriptions with fresh request IDs.
Protocol v1 does not replay old subscribe messages automatically.

## Supplied Client API

The bundled browser module exports `SurfaceSdkClient`, `SurfaceSdkError`,
`isSurfaceBootstrapMessageV1` and `SURFACE_PROTOCOL_VERSION`. The existing client exposes:

```ts
interface SurfaceSdkClient {
  lifecycle: {
    ready(): void;
    setDirty(isDirty: boolean): void;
    onStateChange(listener: (state: "active" | "suspended" | "disposed") => void): () => void;
  };
  commands: {
    execute<TResult>(binding: string, argumentsValue: JsonValue): Promise<TResult>;
    isAvailable(binding: string): boolean;
  };
  queries: {
    execute<TResult>(binding: string, argumentsValue: JsonValue, options?: { signal?: AbortSignal }): Promise<TResult>;
  };
  viewModels: {
    subscribe<TValue>(binding: string, listener: (value: TValue, revision: number) => void): () => void;
  };
  environment: {
    getSnapshot(): SurfaceEnvironmentV1;
    subscribe(listener: (environment: SurfaceEnvironmentV1) => void): () => void;
  };
  dispose(): void;
}
```

Copy the supplied `surface-sdk.js` into your UI folder and import it as an ES module. The SDK build
manifest records its source hashes; keep that metadata in authoring records rather than deliverable
packages. Use the actual client instead of duplicating request IDs, pending maps and port listeners.

The protocol is intentionally capability-limited: the page cannot call Lua, read process memory,
access credentials, call arbitrary host methods, or use the raw WebView bridge.

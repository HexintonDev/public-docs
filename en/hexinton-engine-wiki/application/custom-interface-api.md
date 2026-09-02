# Custom Interface API

Status: current protocol v1 reference.

The current host loads one package HTML document in a sandbox and transfers a private `MessagePort`.
The page can use only the aliases declared in its surface manifest. A page sends a binding alias such
as `health.set`; it never sends a package runnable ID such as `set-health` directly.

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
      "action": "set-health",
      "executionPolicy": "write"
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

## Minimal Client Shape

An inline client or future public SDK can expose this shape:

```ts
interface SurfaceClientV1 {
  lifecycle: {
    ready(): void;
    setDirty(isDirty: boolean): void;
    onStateChange(listener: (state: "active" | "suspended" | "disposed") => void): () => void;
  };
  commands: {
    execute<TResult>(binding: string, argumentsValue: JsonValue): Promise<TResult>;
  };
  queries: {
    execute<TResult>(binding: string, argumentsValue: JsonValue, signal?: AbortSignal): Promise<TResult>;
  };
  viewModels: {
    subscribe<TValue>(binding: string, listener: (value: TValue, revision: number) => void): () => void;
  };
  dispose(): void;
}
```

This is an authoring shape, not a promise that a public npm package is already available. The current
package can bundle a compatible inline client in its single HTML entry document.

The protocol is intentionally capability-limited: the page cannot call Lua, read process memory,
access credentials, call arbitrary host methods, or use the raw WebView bridge.

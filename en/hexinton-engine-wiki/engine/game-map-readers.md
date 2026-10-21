# Live map reader packages

Status: current client contract, checked against code on 2026-10-10. The shared map viewer and VM selection are implemented. The checked-in Palworld reader and teleport packages are experimental; actual game compatibility is not established by their fixture tests.

A live map reader is an ordinary package. It publishes a capability ViewModel and a location ViewModel through the existing [services and feeds](view-models-and-feeds.md). The Map tab reads those values and submits compiled surface commands. It does not read process memory itself, and packages do not get a separate map transport.

## Declare the feeds and commands

For a package with ID `example.world-map`, this fragment declares the two services and the command surface. Add matching `enable`, `disable`, `capability`, `watch`, `start`, `stop` and `teleport` runnables to the normal package manifest. These names are examples; the values advertised by the capability connect them.

```json
{
  "frontend": {
    "hosted": {
      "schemaVersion": 1,
      "viewModels": [
        { "id": "capability", "service": "capability" },
        { "id": "player", "service": "watch" }
      ],
      "groups": []
    },
    "surfaces": [{
      "id": "map-actions",
      "label": "Map actions",
      "kind": "web-inline",
      "entry": "frontend/map-actions/index.html",
      "presentation": { "mode": "panel", "size": "standard" },
      "bindings": {
        "tracking.start": { "type": "command", "action": "start" },
        "tracking.stop": { "type": "command", "action": "stop" },
        "teleport": { "type": "command", "action": "teleport" }
      }
    }]
  }
}
```

The referenced HTML file must exist, and the declared actions/services must pass the normal compiler checks. A reader without teleport omits that binding and action, and advertises `teleportBinding: null`.

## Capability observation

The capability service publishes a value such as:

```lua
function capability()
  publishEvent("capability", {
    kind = "game-map-capability",
    schemaVersion = 1,
    coordinateSpace = "example-world-v1",
    locationFeed = "example.world-map/player",
    surfaceId = "example.world-map/map-actions",
    startBinding = "tracking.start",
    stopBinding = "tracking.stop",
    teleportBinding = "teleport"
  })
end
```

`locationFeed` and `surfaceId` are package-scoped IDs. Binding aliases are local to the surface. The current Map tab requires both start and stop command bindings. The schema permits null values, but a capability without usable start/stop bindings is not selected for live tracking.

Selection requires the capability's `coordinateSpace` to exactly match the map content's calibration. The capability feed, referenced surface and location feed must belong to the same package. Exactly one compatible capability is required: multiple candidates disable automatic selection instead of arbitrarily starting one. Package selection or a future chooser must resolve that ambiguity.

There is no game-name branch or automatic active-world map selection in this contract. A package can report a `worldId`, but current viewer selection is based on the selected map's calibration and compatible capability. Do not claim that reporting a world ID automatically switches maps.

## Location observation

The location service publishes ordinary detached JSON values:

```lua
publishEvent("watch", {
  kind = "game-map-location",
  schemaVersion = 1,
  coordinateSpace = "example-world-v1",
  position = { x = 10, y = 20, z = 30 },
  observedAt = os.date("!%Y-%m-%dT%H:%M:%SZ"),
  state = "ready"
})
```

| Field | Contract |
| --- | --- |
| `kind`, `schemaVersion` | `game-map-location`, `1`. |
| `coordinateSpace` | Exact space shared by the reader, calibration and destination. |
| `position` | Finite numeric `x`, `y`, `z`, or JSON null (`hex.null` in host Lua). |
| `observedAt` | UTC timestamp parseable by the client. The current viewer rejects invalid timestamps or observations older than five seconds. |
| `state` | `ready`, `starting`, `stopped` or `failed`. Only ready observations render a player position. |
| `headingDegrees` | Facing direction in degrees clockwise from map north. Publish it whenever the game exposes the player's facing (see [Facing direction](#facing-direction)); when omitted, the viewer falls back to movement direction. |
| `worldId` | Optional world identity, useful for destination checks. Its meaning must be verified by the package author. |

Coordinate calibration is supplied by the map content, separately from the runtime package. The current `mercator-affine-v1` transform has two rows of four coefficients applied to `[x, y, z, 1]`, followed by the map projection. Retain verified axes, units, offsets and radius; a raw map pin coordinate is not automatically a game position.

## Facing direction

Publish the real facing direction whenever the game exposes it: camera or controller yaw for third-person and first-person games where WeMod-style maps follow the camera, otherwise the player actor's yaw. Read the yaw ψ (radians) in the world plane spanned by the calibration's first two world axes, measured from +axis0 towards +axis1, and convert it through the published map calibration `matrix`:

```lua
-- m = calibration.matrix of the published map content (rows of x, y, z, translation coefficients)
local e = m[1][1] * math.cos(psi) + m[1][2] * math.sin(psi)
local n = m[2][1] * math.cos(psi) + m[2][2] * math.sin(psi)
headingDegrees = math.deg(math.atan(e, n)) % 360
```

That is `heading = atan2(m00·cos ψ + m01·sin ψ, m10·cos ψ + m11·sin ψ) mod 360`. If the game's horizontal plane is x/z (y up), use the x and z columns instead. Verify the signs against the live published map content: walk forward along a road on the map and confirm the arrow points along it, then turn 90° right and confirm the arrow turns clockwise. Unreal `Rotator.Yaw` is in degrees; convert to radians first.

When a package publishes no heading, the client derives one from movement in map space: it ignores movement under about three pixels at the deepest map zoom, keeps the last heading while the player stands still, re-anchors without turning after teleports or world changes, and shows a non-directional marker until the first movement. A published `headingDegrees` always wins over the fallback, so publish it only from a verified source.

## Demand and lifecycle

Capability and location services may initialize their feeds when the host starts them, but target polling should remain stopped until `start` requests it. Make repeated start/stop calls safe. Stop polling when the tab releases demand, the live preference is disabled, or the package/session tears down; `disable` must destroy package-owned timers and undo owned hooks.

The authoritative session must be `Attached` before the client starts the reader. React preserves the enabled preference while waiting and relies on the existing session state. A disappeared pawn, respawn or world transition must not leave stale object references in the reader: resolve the current player again or publish an unavailable observation.

## Teleport action

The Map tab sends a normal command to the advertised teleport binding:

```json
{
  "coordinateSpace": "example-world-v1",
  "position": { "x": 10, "y": 20, "z": 30 },
  "worldId": "optional-verified-world"
}
```

The destination is the pin's separate 3D game destination, not its map image coordinate. The UI checks matching spaces and an advertised destination world against the current location world. The package must independently validate finite coordinates, required height, world compatibility, player availability and game-side success. A UI check does not replace runtime validation. Await command completion before reporting success.

For Unreal games, a host-Lua adapter can call a declared UE4SS dependency:

```lua
local reader = "example.unreal-reader"
local position = hex.query(reader, "location", {})
hex.action(reader, "teleport", parameters)
```

The UE4SS dependency owns reflected game-thread calls and returns detached values; host Lua owns the capability/player feeds and demand timer. See [dependency calls](dependency-runnable-api.md) and [UE4SS requirements](ue4ss-runtime.md). The current experimental Unreal reader calls `K2_GetActorLocation` and `K2_TeleportTo`, preserving actor rotation. It does not establish compatibility with every Unreal game or add framework delivery automatically.

## Verify the package

Use [unit and integration tests](testing.md) for production transformations, missing/ambiguous matches, repeated start/stop, timer cleanup, failed enable rollback, malformed destinations and failed game calls. Simulated Unreal objects can verify the adapter's contract and cleanup but cannot establish a real game's pawn ownership, movement authority or coordinate alignment. Qualify those in the actual supported game/build before publishing a compatibility claim.

Current implementation references:

- [Shared map types and guards](https://github.com/HexintonDev/HexModClient/tree/main/packages/game-maps/src)
- [Client capability selection and command demand](https://github.com/HexintonDev/HexModClient/blob/main/ProcessEngine/host/HexEngine.WebViewTest/Views/src/features/maps/useMapRuntime.ts)
- [Experimental reader packages and harness](https://github.com/HexintonDev/HexModClient/tree/main/ProcessEngine/experiments/game-maps)

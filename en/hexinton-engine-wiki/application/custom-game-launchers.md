# Custom Game Launchers

Status: current desktop script contract. Launchers can be delivered in synchronized packages.
The saved installation must use `CustomScript`; the client does not yet provide a complete launcher configuration screen.

A game launcher can run an application JavaScript function before starting the game. The function
can choose an executable, set its working directory and environment variables, or open a launcher URI.
It runs before the native engine attaches to the game.

## Connect a launcher to a game

Two pieces must agree:

1. The game's saved installation uses the `CustomScript` launch type.
2. A selected synchronized package supplies a launcher script, or that game's local support store supplies one.

A support package does not replace an installation's existing Steam launch action automatically.
Once an installation uses `CustomScript`, refreshing installed games preserves that choice and its
selected paths and arguments while updating provider metadata.
Refresh packages downloads and validates selected packages. A selected package's application script
with `kind: "launcher"` is used from the session's applied package files. Exactly one selected launcher
is allowed; launcher scripts in dependency-only packages are not selected automatically. Invalid or
ambiguous synchronized launchers fail visibly. When no selected package declares a launcher, the host
can use the older local store described below.

For normal package upload and synchronization, declare the script in `package.json`:

```json
{
  "id": "example.game-support",
  "version": "1.0.0",
  "dependencies": [],
  "application": {
    "scripts": [{
      "id": "offline-launch",
      "kind": "launcher",
      "runtime": "js",
      "entryFile": "launcher/launch.js",
      "entrySymbol": "launch",
      "capabilities": ["launcher.read", "launcher.launch"]
    }]
  }
}
```

Publication requires the backend's game-support creator approval or first-party authorization for
these capabilities. An ordinary creator cannot grant those permissions by adding manifest strings.
The package must be selected through defaults or a subscription and refreshed before launch.
Already installed, validated packages remain usable under the client's offline policy.

The installation's `launch` value has this shape:

```json
{
  "type": "CustomScript",
  "selectedDirectory": "C:/Games/ELDEN RING/Game",
  "selectedFile": "C:/Games/ELDEN RING/Game/eldenring.exe",
  "arguments": []
}
```

The launcher receives those choices alongside `gameId`, `installationId`, `platform`,
`platformGameId`, `installPath`, `executablePath` and `buildId`. These are local installation values.
Use the registered game ID consistently in the support-package directory and manifest.

## Support-package files

The desktop reads support packages from `%LOCALAPPDATA%/Hexinton Mod/support-packages`:

```text
support-packages/
  <registered-game-id>/
    active.json
    versions/
      1.0.0/
        manifest.json
        launcher/
          launch.js
```

`active.json` selects the version:

```json
{ "version": "1.0.0" }
```

The support-package `manifest.json` identifies the game and launcher. Replace
`<registered-game-id>` with the actual game ID before using this example:

```json
{
  "schemaVersion": 1,
  "packageId": "game-support:elden-ring",
  "gameId": "<registered-game-id>",
  "version": "1.0.0",
  "supportedBuilds": [],
  "entrypoints": { "launcher": "launcher/launch.js" },
  "requiredCapabilities": {
    "launcher": ["launcher.read", "launcher.launch"]
  }
}
```

This is the launcher's support-package manifest, separate from a trainer's `package.json`.
The loader checks the schema, matching game/version, script path containment and file existence.
If a known installation build is outside a nonempty `supportedBuilds` list, loading fails.
An empty list adds no build restriction.

Launcher scripts receive only the allowed `launcher.read` and `launcher.launch` capabilities.
Adding other names here does not grant them. See [host capabilities](application-js-host-capabilities.md)
for the executable and URI call options.

## Elden Ring example

This script starts `eldenring.exe` directly with the two Steam environment variables. Select the
executable in the game's `Game` directory using the installation values above.
Keep Steam running before using this example. In the integration check, the direct game process
exited when Steam was closed and stayed running after Steam was opened.

```js
export function launch(args) {
  const executable = (args.selectedFile || args.executablePath || "")
    .replace(/\\/g, "/");

  if (!/(?:^|\/)eldenring\.exe$/i.test(executable) || executable.lastIndexOf("/") < 0) {
    return { status: "failed", message: "Select eldenring.exe in the game's Game folder." };
  }

  const result = hex.launcher.openExecutable(executable, {
    workingDirectory: executable.slice(0, executable.lastIndexOf("/")),
    arguments: args.arguments || [],
    environment: { SteamAppId: "1245620", SteamGameId: "1245620" }
  });

  return result.accepted
    ? { status: "accepted", processId: result.processId }
    : { status: "failed", message: "Windows rejected the launch request." };
}
```

The environment values apply to the launched child process. The working directory is the folder
containing the selected executable. The host starts it directly with separate arguments, without
interpreting a batch command or changing the application's own working directory.

## Return values and failures

Export the function named by `entrySymbol` (`launch` in these examples). It must return `{ "status": "accepted" }` after Windows accepts
the launch; `processId` is optional. Returning the raw `hex.launcher.openExecutable` result is
insufficient because it has `accepted`, rather than the required `status` field.

An accepted launch means the start request succeeded. The client still detects the actual game
process and attaches before enabling in-game controls. See [game detection](game-detection.md).

Missing or incompatible support-package files produce `launch_script_not_found`. A denied
capability produces `launch_script_not_authorized`. A script exception or a return value without
`status: "accepted"` produces `launch_script_failed`. Keep a failed launch visible and correct the
saved installation or launcher files before retrying.

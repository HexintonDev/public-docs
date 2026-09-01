# Application JavaScript Host Capabilities

Status: current local application capability contract. Cloud authorization policy is future work.

Application JavaScript runs in the desktop host's Jint runtime. It can access host behavior only
through the `hex` object. CLR reflection and unrestricted CLR access are not exposed.

## Capability Names

| Capability | Available host surface |
| --- | --- |
| `process.read` | `hex.findProcess`, `hex.findProcesses` |
| `window.read` | Window discovery and foreground-window queries |
| `filesystem.read` | Directory, file metadata, existence, and text reads |
| `filesystem.write` | Text-file writes |
| `winapi` | `hex.winapi.call` |
| `launcher.read` | Normalized launch context input |
| `launcher.launch` | `hex.launcher.openExecutable`, `hex.launcher.openUri` |

Missing capability access fails with `script_capability_denied` and includes the denied capability.
The current local/test policy treats declared capabilities as granted. Future cloud policy must
review `winapi` and `filesystem.write` as dangerous grants.

## Process and Window Queries

```js
const matches = hex.findProcesses("game.exe");
const foreground = hex.getForegroundWindow();
```

These are observation APIs. They do not attach a native engine process session and cannot perform
Lua memory reads or writes.

## Filesystem

```js
const entries = hex.readDirectory("C:/Games/Demo");
const stat = hex.stat("C:/Games/Demo/game.exe");
const text = hex.readTextFile("C:/Games/Demo/config.ini");
hex.writeTextFile("C:/Temp/hex-output.txt", "diagnostic text");
```

`hex.stat(path)` returns `null` when a path does not exist. Returned directory and stat values are
serializable and include path/type information. Use forward-slash paths in package examples.

## Launcher

```js
const executable = hex.launcher.openExecutable("C:/Games/Demo/game.exe", {
  workingDirectory: "C:/Games/Demo",
  arguments: ["--windowed"],
  environment: { DEMO_MODE: "1" }
});

const uri = hex.launcher.openUri("steam://rungameid/1091500");
```

Launcher calls return a serializable acceptance object containing `accepted`, `method`, and
`target`. A launcher may start one terminal executable or URI, but may not recursively dispatch
another custom-script strategy.

## WinAPI

`hex.winapi.call` requires `winapi`. A request names a module, export, return type, and optional
argument descriptors. Supported argument types include `bool`, `int32`, `uint32`, `int64`,
`uint64`, `pointer`, `handle`, `null`, `utf16`, `ansi`, and `utf8`. Pointer and handle returns are
serialized as hexadecimal strings.

```js
const pid = hex.winapi.call({
  module: "kernel32.dll",
  function: "GetCurrentProcessId",
  returnType: "uint32"
});
```

Only call trusted, documented exports with validated arguments. Capability declaration is not a
substitute for input validation or package review.

## Runtime Boundary

Application JavaScript is for host-side UI, launch, discovery, and declared application behavior.
Lua and Auto Assembler are the attached native engine runtimes. A JavaScript surface must use a
host command, query, view model, or feed to interact with native package behavior.

# Application Errors

Status: current application error payload standard.

Application failures use one structured shape across C# services, WebView commands, application
JavaScript, sync, launch, and engine adapters.

```json
{
  "severity": "error",
  "domain": "game_launch",
  "code": "launch_target_missing",
  "message": "The selected game executable could not be found.",
  "details": { "gameId": "demo.game.alpha" }
}
```

`severity`, `domain`, `code`, `message`, and `details` are always present. Application logic branches
on `domain` and `code`, never on message wording. Details are diagnostic string values and must not
contain credentials, tokens, arbitrary file contents, or unnecessary full user paths.

## Correlated Commands

Every accepted WebView request receives exactly one correlated result:

```json
{
  "type": "commandResult",
  "payload": {
    "requestId": "request-1",
    "ok": false,
    "result": null,
    "error": {
      "severity": "error",
      "domain": "sync",
      "code": "package_invalid",
      "message": "The package could not be applied.",
      "details": {}
    }
  }
}
```

A command error answers one operation. A projected state error represents a durable workflow state
and is cleared only by its recovery transition. Toasts are transient feedback and must not replace
projected state.

## Common Domains

Launch uses `game_launch`; installation profile operations use `game_installation`; package and
native failures are mapped at their application boundary. Unknown exceptions become a safe
`*_unknown_error` message while detailed exception data remains in protected logs.

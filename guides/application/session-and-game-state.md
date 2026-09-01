# Session and Game State

Status: current desktop session behavior.

The application owns one `GameSession` per game identity. A game identity (`gameId`) is not the
same as an attached process identity (`pid`); a pid is discovered later and is valid only for the
current attachment generation.

## States

```text
SyncRequired -> NotRunning -> Loading -> Attached
       |             |           |
       v             v           v
   SyncFailed     Loading     NotRunning
```

- `SyncRequired`: package synchronization and hosted-view compilation are pending.
- `SyncFailed`: preparation failed; script and trainer commands are inadmissible.
- `NotRunning`: packages are ready but no target is attached; detection is active.
- `Loading`: launch was accepted and detection is waiting for a verified target.
- `Attached`: a positive pid and session token exist; native commands are allowed.
- `Detached`: an internal recovery state without an active pid; re-detection can attach a new target.

A failed detection tick does not mean the game exited. Only a conclusive target-loss observation can
reconcile an attached session.

## Command Gating

Native package commands require `Attached`, a current pid, a current session token, synchronized
package membership, and a valid runnable. UI commands should use the typed application boundary and
must not call the native engine directly.

## Command Versus Observation

A command result reports one operation. The committed trainer value is owned by the session and
projected through application state. Values that change independently of a command must use a
package service and feed.

When a target is lost, the application stops hosted services, destroys the native context, clears
active runtime values, and publishes the resulting session state. Feature state is not automatically
replayed when a later attachment is created.

## Worker Rule

Session mutations, state transitions, and trainer-value commits run through one serialized session
worker. Detection performs discovery outside that worker, then re-enters it to apply the immutable
observation. This prevents detection, launch, reload, and command operations from interleaving.

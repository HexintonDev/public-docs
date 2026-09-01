# Game Detection

Status: current desktop detection behavior.

Detection starts after package synchronization when a session reaches `NotRunning`. The heartbeat
runs once immediately and then every two seconds by default. It owns cadence, retry, duplicate-failure
suppression, and logging; it never mutates session state directly.

## Strategies

### Finder mode

A synchronized package may declare an application-JavaScript finder. The host invokes it once per
heartbeat with game identity, current pid when attached, and normalized installation metadata. The
finder returns `null` or an object containing a positive `pid`, with optional platform, build, and
diagnostic fields.

Configured finder mode is authoritative for that tick. A null result does not fall back to loose
executable-name matching. Finder scripts are observational and may use only `process.read`,
`window.read`, and `filesystem.read`.

### Traditional mode

Without a configured finder, detection checks the preferred registered installation, re-verifies the
current pid when attached, then enumerates processes. It prefers exact executable-path matching and
uses executable-name matching only when Windows hides the path.

## Failure and Shutdown

Exceptions, malformed finder results, denied capabilities, and transient process access failures are
inconclusive. The current target is preserved and the next heartbeat retries. Repeated identical
failures are suppressed until a successful cycle resets suppression.

Shutdown cancels the heartbeat and in-flight finder before the shared script executor is disposed.
A detector cannot launch a process, attach a native context, or change session state itself.

# Sample: Service Timer Feed

Sample ID: `service/timer-feed`

This sample demonstrates a service that publishes one initial value and then updates it from a timer.

## Evidence Record

Validated against commit `3307591`.

- Fixture: `ProcessEngine/tests/script_fixtures/service_lifecycle/`
- Manifest: `.../trainer.service/package.json`
- Lua entry: `.../trainer.service/lua/main.lua`
- Service lifecycle test: `ProcessEngine/tests/script_execution_controller_test.cpp`

## Behavior

The `watch-hp` service validates an optional positive `interval`, publishes an initial
`health_changed` event, creates a disabled timer, configures `Interval` and `OnTimer`, then enables
the timer. `disable` destroys the timer and clears its Lua reference.

## Reuse Rules

- Publish an initial authoritative value before starting periodic observation.
- Publish later values only when the meaningful value changes.
- Keep one timer per service unless parallel behavior is required and documented.
- Destroy the timer in `disable` and set its variable to `nil`.

## Expected Result

The service publishes immediately, later publishes a changed value, and stops publishing after
package disable or script teardown.
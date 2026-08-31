# Sample: Lua Multi-File Require

Sample ID: `lua/multifile-require`

This sample demonstrates package-local Lua modules and the `require` cache.

## Evidence Record

Validated against commit `3307591`.

- Fixture: `ProcessEngine/tests/script_fixtures/lua_require_multifile/`
- Main Lua entry: `.../trainer.player/lua/main.lua`
- Inventory helper: `.../trainer.player/lua/helpers/inventory.lua`
- Module test: `ProcessEngine/tests/script_execution_controller_test.cpp`

## Behavior

`require("helpers.inventory")` loads `lua/helpers/inventory.lua` from the current package. Calling
`require` with the same module ID later returns the cached module value rather than evaluating the
file again.

## Reuse Rules

- Use dot-separated module IDs only, such as `helpers.inventory`.
- Do not use absolute paths, path separators, `..`, or a `.lua` suffix in a module ID.
- Use `require` only for files within the current package.
- Consume another package through a declared dependency contract, not cross-package `require`.

## Expected Result

The sample confirms that the same module table is returned from the cache and that the helper's load
count remains stable after repeated `require` calls.
# Sample: Lua Multi-File Require

Sample ID: `lua/multifile-require`

This sample demonstrates package-local Lua modules and the `require` cache.

## Source and Test

- [Pinned fixture directory](https://github.com/HexintonDev/HexModClient/tree/3307591b19f2f73e543636e77de0d77b959a3fd2/ProcessEngine/tests/script_fixtures/lua_require_multifile)
- [Main Lua entry](https://github.com/HexintonDev/HexModClient/blob/3307591b19f2f73e543636e77de0d77b959a3fd2/ProcessEngine/tests/script_fixtures/lua_require_multifile/trainer.player/lua/main.lua)
- [Inventory helper](https://github.com/HexintonDev/HexModClient/blob/3307591b19f2f73e543636e77de0d77b959a3fd2/ProcessEngine/tests/script_fixtures/lua_require_multifile/trainer.player/lua/helpers/inventory.lua)
- [Module test](https://github.com/HexintonDev/HexModClient/blob/3307591b19f2f73e543636e77de0d77b959a3fd2/ProcessEngine/tests/script_execution_controller_test.cpp)

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
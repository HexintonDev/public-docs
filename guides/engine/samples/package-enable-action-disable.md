# Sample: Package Enable, Action, and Disable

Sample ID: `package/enable-action-disable`

This sample shows the smallest complete package lifecycle with a dependency, a Lua action, symbols,
an Auto Assembler allocation, and matching cleanup.

## Source and Test

- [Pinned fixture directory](https://github.com/HexintonDev/HexModClient/tree/3307591b19f2f73e543636e77de0d77b959a3fd2/ProcessEngine/tests/script_fixtures/basic_enable_chain)
- [Player manifest](https://github.com/HexintonDev/HexModClient/blob/3307591b19f2f73e543636e77de0d77b959a3fd2/ProcessEngine/tests/script_fixtures/basic_enable_chain/trainer.player/package.json)
- [Player Lua entry](https://github.com/HexintonDev/HexModClient/blob/3307591b19f2f73e543636e77de0d77b959a3fd2/ProcessEngine/tests/script_fixtures/basic_enable_chain/trainer.player/lua/main.lua)
- [Dependency Lua entry](https://github.com/HexintonDev/HexModClient/blob/3307591b19f2f73e543636e77de0d77b959a3fd2/ProcessEngine/tests/script_fixtures/basic_enable_chain/trainer.shared/lua/main.lua)
- [Lifecycle test](https://github.com/HexintonDev/HexModClient/blob/3307591b19f2f73e543636e77de0d77b959a3fd2/ProcessEngine/tests/script_execution_controller_test.cpp)

## Behavior

The host enables `trainer.shared` before `trainer.player`. `enable` publishes a symbol and creates
an allocation. The `give-runes` action receives an integer argument, writes through an explicit
process handle, and returns the supplied amount. `disable` unregisters every published symbol and
releases the allocation.

## Reuse Rules

- Declare every dependency in the manifest; do not rely on load order.
- Validate action arguments with `parameterSchema`.
- Treat every symbol and allocation created by `enable` as an owned resource.
- Reverse enable work during `disable`, even when an action has never run.

## Expected Result

After enable, dependency globals are available through the declared dependency boundary. After the
action, its return value equals `amount`. After disable, package-owned symbols and the allocation no
longer exist.
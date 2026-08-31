# Sample: Auto Assembler Allocation Action Pair

Sample ID: `aa/allocation-action-pair`

This sample demonstrates an Auto Assembler action that allocates a named code cave and a second action
that releases it.

## Source and Test

- [Pinned fixture directory](https://github.com/HexintonDev/HexModClient/tree/3307591b19f2f73e543636e77de0d77b959a3fd2/ProcessEngine/tests/script_fixtures/aa_action_pair)
- [Manifest](https://github.com/HexintonDev/HexModClient/blob/3307591b19f2f73e543636e77de0d77b959a3fd2/ProcessEngine/tests/script_fixtures/aa_action_pair/trainer.player/package.json)
- [Enable action](https://github.com/HexintonDev/HexModClient/blob/3307591b19f2f73e543636e77de0d77b959a3fd2/ProcessEngine/tests/script_fixtures/aa_action_pair/trainer.player/aa/enable-godmode.aa)
- [Disable action](https://github.com/HexintonDev/HexModClient/blob/3307591b19f2f73e543636e77de0d77b959a3fd2/ProcessEngine/tests/script_fixtures/aa_action_pair/trainer.player/aa/disable-godmode.aa)
- [AA action test](https://github.com/HexintonDev/HexModClient/blob/3307591b19f2f73e543636e77de0d77b959a3fd2/ProcessEngine/tests/script_execution_controller_test.cpp)

## Behavior

The `enable-godmode` action runs `alloc(godmodeCave, 32)`. The matching `disable-godmode` action
runs `dealloc(godmodeCave)`. The Lua package lifecycle remains Lua because Auto Assembler is
action-only in the native host.

## Reuse Rules

- Declare Auto Assembler runnables with `runtime: "aa"` and an `entryFile`.
- Do not use Auto Assembler for `enable`, `disable`, or a continuously running service.
- Pair every `alloc` with a deterministic `dealloc` action or a Lua `disable` cleanup path.
- Build real hooks on top of validated scans and restoration logic; allocation alone is not a hook.

## Expected Result

The first action creates `godmodeCave`; the second action removes it. The test verifies the
allocation/action lifecycle through the package command host.
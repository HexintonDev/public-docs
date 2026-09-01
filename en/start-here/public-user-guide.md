# Public User Guide

Status: current user guide and basic scripting introduction.

This guide is for users who have installed and can open the Hexinton Mod client. It explains how to select a game, launch it, connect a trainer, and use mod features. It also introduces how to write a basic script in Studio. It does not cover source downloads, compilation, or client development. For advanced mod development, see [Further Learning](#further-learning).

## Before You Start

Before using Hexinton Mod, confirm that:

* You are using Windows.
* The Hexinton Mod client is installed and opens normally.
* The target game is installed locally.
* The target game appears in the client's `My Games` list.
* You understand the rules and anti-cheat policy for online games.

Mods may modify a game process or save data. Back up important saves before use and only use trainers and mods that you trust. Do not use them in online modes that prohibit game modification.

## Main Interface

After opening the client, the left navigation bar contains these entries:

| Entry | Purpose |
| --- | --- |
| Home | View registered games and recently used games. |
| Creator | Open creator features. |
| Debug | View application and connection status for troubleshooting. |
| Settings | Change the language and trainer feedback sounds. |
| My Games | View games registered with Hexinton Mod on this machine. |

The search box at the top of Home searches games registered with the local client. It does not search for or download games or mods from the network.

## Open a Game Trainer

1. Find the target game in `Home` or `My Games`.
2. Select the game card to open its trainer page.
3. Wait for the client to prepare the game information and trainer content.

The trainer page usually contains:

* `Mods`: Features available for the current game.
* `Saves`: Current content may be demonstration data and is not a production save-management feature.
* `Play`: Launches the game.
* `Studio`: Opens the script and trainer package editor.

If the game does not appear in `My Games`, the current version does not provide a complete `Add New Game` workflow for ordinary users. Confirm that your client version has registration information for the game.

## Choose a Game Installation

If a game has multiple installations, use the dropdown beside `Play` to choose which version to launch.

1. Select the dropdown beside `Play`.
2. Choose the correct installation in the `Launch installation` list.
3. The selected installation becomes the default for later launches.

The list may show these statuses:

* `Missing file`: The saved game executable was moved, deleted, or cannot be accessed.
* `No installation profiles`: No game installation profile is currently available.

Use `Add game executable` to select the game's Windows `.exe` file. Select the game executable itself, not a launcher shortcut, folder, or another file type.

Current limitation: when a game has no installation profile at all, the installation menu may not open. Ordinary users cannot complete the first addition from that page in this case. Use a client version with game information already configured or contact the maintainer.

## Launch and Connect the Game

1. Open the target game's trainer page.
2. Confirm that `Play` uses the correct installation.
3. Select `Play`.
4. Wait for the game to start.
5. Wait for Hexinton Mod to detect and connect to the game process.

During launch, the button may show:

| Status | Meaning |
| --- | --- |
| `Launching...` | The client is sending and processing the launch request. |
| `Cancel launch` | Cancels the current launch request. |
| `Preparing session` | The trainer and game session are being prepared. |
| `Updating session...` | The current session content is being synchronized. |
| `Running` | The game is running. |
| `Session sync failed` | Trainer content synchronization failed. |
| `No installation` | No usable game installation is available. |

`Launch request accepted` only means that the client accepted the launch request. It does not mean that the trainer is connected. Continue waiting until the trainer controls are unlocked.

If the game was started by Steam, another launcher, or the user, the client still needs to detect the correct target process and attach to it. In-game modifications are available only after the connection is complete.

## Use Trainer Features

After connecting, open the `Mods` tab. The available features come from trainer packages synchronized and loaded successfully for the current game.

### Toggles

Toggles usually enable or disable a persistent effect, such as unlimited health or resources.

1. Turn on the relevant toggle.
2. Wait for the operation to complete.
3. Turn it off when you want to restore the game's original behavior.

Do not click repeatedly while a command is running. Before closing the game, turn off persistent features that are still enabled.

### Numeric Controls

Numeric controls set quantities, multipliers, or other parameters.

1. Enter or adjust the value.
2. Confirm that it is within the allowed range.
3. Select the relevant execute button, or wait for the control to submit as designed.

Some controls provide increment/decrement buttons or dropdown options. The actual in-game value may update shortly after a successful submission.

### Actions

An item with an `Execute` button represents a one-time operation, such as adding an item, teleporting, or refreshing data.

1. Enter the required parameters.
2. Select `Execute`.
3. Wait for the success or failure result.

A success message confirms only that the command completed. It does not mean that the feature remains active. Use the relevant toggle for a persistent effect.

### Read-only State

Some items display the current in-game value and cannot be edited. They refresh through trainer state updates. Immediately after a command completes, the displayed value may briefly remain old; wait for the next state update.

### Custom Interfaces

Some trainers show dedicated panels or small applications, such as an item search, waypoint list, or color picker. They can call only the commands and data explicitly authorized by that trainer.

Custom interfaces remain locked while the game is not connected. Use them after the connection succeeds.

## Write Your Own Script

Hexinton Mod organizes scripts into packages. A package contains at least a `package.json` file and one Lua file. You do not need to modify the client itself; edit package files in `Studio`.

### Open Studio

1. Open the target game's trainer page.
2. Select `Studio` at the top of the page.
3. If the client says that Studio can only be opened in the desktop application, use the Hexinton Mod desktop client instead of a regular browser.
4. Select the local package to edit in Studio.
5. If the package is synchronized and read-only, select `Create Local Copy...` and edit the copy.

Synchronized packages are normally not editable directly. Creating a local copy prevents you from overwriting synchronized content and lets you keep your changes.

### Create Package Files

Use `New Package...` in Studio to create a package, or create this structure in a local copy:

```text
example.health/
  package.json
  lua/main.lua
```

The directory name must exactly match the `id` in `package.json`. Use English letters, numbers, hyphens, or dots in file and directory names, and avoid spaces.

### Write package.json

`package.json` tells Hexinton Mod which scripts to load and which features users can execute:

```json
{
  "id": "example.health",
  "version": "1.0.0",
  "displayName": "Health Example",
  "runtime": {
    "files": [
      { "path": "lua/main.lua", "runtime": "lua" }
    ],
    "public": { "lua": "lua/main.lua" },
    "runnables": [
      {
        "id": "enable",
        "kind": "enable",
        "runtime": "lua",
        "entryFile": "lua/main.lua",
        "entrySymbol": "enable"
      },
      {
        "id": "disable",
        "kind": "disable",
        "runtime": "lua",
        "entryFile": "lua/main.lua",
        "entrySymbol": "disable"
      },
      {
        "id": "heal",
        "kind": "action",
        "runtime": "lua",
        "entryFile": "lua/main.lua",
        "entrySymbol": "heal",
        "parameterSchema": {
          "type": "object",
          "properties": {
            "amount": { "type": "integer" }
          }
        }
      }
    ]
  }
}
```

The most important fields are:

* `id`: The package's unique ID, which must match the directory name.
* `version`: The package version.
* `runtime.files`: All runtime files to load.
* `runnables`: The features provided by the script.
* `kind`: The feature type, commonly `enable`, `disable`, or `action`.
* `entrySymbol`: The function name that exists in the Lua file.
* `parameterSchema`: The parameter format accepted by an action.

JSON must use strict syntax: property names and string values require double quotes, and the final property must not have a trailing comma.

### Write the Lua Script

Put the following in `lua/main.lua`:

```lua
local health = 100
local maximumHealth = 100

function enable()
    -- Install features owned by this package here.
end

function disable()
    -- Remove hooks, timers, and other resources created by enable.
end

function heal(arguments)
    local amount = arguments.amount or 25
    health = math.min(maximumHealth, health + amount)
    return {
        current = health,
        maximum = maximumHealth
    }
end
```

Script rules:

* `enable()` runs when the package starts.
* `disable()` runs when the package stops and must clean up its resources.
* An `action` function receives a parameter table, such as `arguments.amount`.
* Functions may return numbers, strings, booleans, or tables containing those values.
* Do not put keys, account information, or private file contents in a package.
* Operate only on addresses, symbols, and processes that you know belong to the target game.

### Common Lua Operations

After the script is connected to the game, it can use common functions such as these:

```lua
local pid = findProcess("game.exe")
local processHandle = openProcess(pid)

local address = processHandle:getAddress("game.exe+0x1234")
local currentValue = processHandle:readInteger(address)
processHandle:writeInteger(address, 999)
```

Common functions include:

| Function | Purpose |
| --- | --- |
| `findProcess(name)` | Find the game process ID. |
| `openProcess(pid)` | Open a session for a process. |
| `getAddress(expression)` | Resolve a module, symbol, or address expression. |
| `readInteger(address)` | Read an integer. |
| `writeInteger(address, value)` | Write an integer. |
| `readQword(address)` / `writeQword(address, value)` | Read or write a 64-bit integer. |
| `readFloat(address)` / `writeFloat(address, value)` | Read or write a floating-point value. |
| `AOBScan(pattern)` | Scan for a byte pattern. |
| `registerSymbol(name, address)` | Register a package-scoped symbol. |
| `autoAssemble(text)` | Execute an Auto Assembler operation. |
| `createTimer(...)` | Create a timer task. |
| `sleep(milliseconds)` | Pause the current script. |

Prefer using the handle returned by `openProcess(...)` for reads and writes. For example, `processHandle:readInteger(...)` makes the target process explicit and helps avoid operating on another process. Address expressions can use a form such as `game.exe+0x1234`.

### Show Script Features in the Trainer

Defining a function in Lua does not automatically add a button to the trainer page. To expose an action, declare it in `package.json` under `runnables` and keep these fields consistent:

```text
runnables[].entryFile    -> actual Lua file
runnables[].entrySymbol  -> actual Lua function name
runnables[].runtime      -> lua
```

For example:

```json
{
  "id": "heal",
  "kind": "action",
  "runtime": "lua",
  "entryFile": "lua/main.lua",
  "entrySymbol": "heal"
}
```

The loaded trainer interface will then show this action. Its exact appearance depends on the trainer interface configuration provided by the package.

### Save, Apply, and Test

1. Edit `package.json` or a Lua file in the Studio editor.
2. Wait for the editor to save automatically, or use its save action.
3. Return to the game trainer page.
4. If `Apply package changes` appears, select it.
5. Wait for `Applying package changes...` to finish.
6. Start the game and wait until the session is connected.
7. Run your new action or enable its toggle in the `Mods` page.
8. Disable the feature after testing, then modify the script.

Studio provides `Widget Preview` for trainer controls, but a preview does not replace testing with the real game. Scripts that read or write memory, scan, patch, or hook a process must be verified in a recoverable test environment.

### Common Script Errors

| Symptom | Check |
| --- | --- |
| Package is not shown | The directory name matches `id`, and files are at their declared paths. |
| Package cannot load | `package.json` uses strict JSON syntax. |
| Button fails after selection | `entrySymbol` exactly matches the Lua function name. |
| Feature cannot be disabled | `disable()` cleans up resources created by `enable()`. |
| Address read fails | The game is connected, and the module name and address expression are correct. |
| Parameter is empty | `parameterSchema` matches the parameter name read by Lua. |
| Change is not visible | The files were saved and `Apply package changes` was selected. |

Fix script errors while the game is not running when possible. Do not hide errors by repeatedly selecting an execute button, and do not run memory-modification scripts found online unless you understand what they do.

## Trainer Cannot Be Used

If trainer controls are locked or unavailable, check the following in order:

1. The game is running.
2. Hexinton Mod is not still preparing the session.
3. The client is connected to the correct game process.
4. The game version is compatible with the trainer.
5. The trainer package synchronized and loaded successfully.
6. The game is not running under a different Windows user or higher privilege level.

If the game runs with administrator privileges, try running Hexinton Mod with the same privileges. Do not disable system security features without understanding the risks.

## Apply Package Updates

If the page shows `Apply package changes`, local trainer files have changed but are not yet applied to the current session.

1. Make sure the current operation has completed.
2. Select `Apply package changes`.
3. Wait for `Applying package changes...` to finish.
4. Confirm that trainer controls become available again.

Ordinary users usually do not see this message. It mainly appears after local trainer content is edited or updated.

## Settings

Open `Settings` from the left navigation to change:

* The client display language.
* Trainer operation feedback sounds.
* Sound volume.
* The interval between consecutive feedback sounds.
* Feedback sound preview.

Sounds only report trainer operation status. They do not mean that an in-game modification will remain active. Use the trainer result and in-game behavior as the final indication.

## Stop Using the Trainer

At the end of a session, use this order:

1. Disable persistent trainer features that are enabled.
2. Wait for the disable operations to complete.
3. Exit the game normally.
4. Wait for the client to show that the game is disconnected.
5. Close Hexinton Mod.

This gives the trainer an opportunity to clean up hooks, timers, memory allocations, and other session resources.

## Frequently Asked Questions

### Hexinton Mod Cannot Find the Game

If you see `Hexinton Mod can't find the game.`:

1. Confirm that the game is installed.
2. Check the game `.exe` from the `Play` menu.
3. If the installation shows `Missing file`, select the correct `.exe` again.
4. Confirm that the game was not moved to another directory.

### Windows Cannot Start the Game

If you see `Windows couldn't start the game.`:

* Try starting the game manually from its folder or official launcher.
* Check that the game files are complete.
* Check whether security software blocked the client from starting the game.
* Select the correct game `.exe` again.

### Launch Link Is Invalid

If you see `The game's launch link isn't working.`, the saved URI or launcher link is no longer valid. Use a valid installation profile or start the game manually from its official launcher.

### Game Started but Trainer Is Still Locked

* Wait for the game to finish loading.
* Confirm that a supported game version is running.
* Confirm that multiple processes with the same name are not running at once.
* Open `Debug` to view the session status.
* Restart the game and Hexinton Mod, then try again.

### Operation Timed Out

If you see `Hexinton Mod took too long to respond.`, do not retry repeatedly. Check whether the game is frozen or changing scenes, wait a few seconds, and try once more.

### Current Operation Is Unavailable

If you see `This action isn't available right now.`, the game may not be connected, the session may be updating, or the current trainer may not provide that operation. Wait for the state to stabilize and try again.

### Client Version Is Out of Date

If you see `This Hexinton Mod version is out of date.`, install the latest client version provided by the maintainer before using it again.

## Current Version Limitations

The following screens or features do not yet provide a complete ordinary-user workflow in the current version:

* Online mod catalog, download, installation, and removal.
* Registering a new game from the client.
* Adding a game's `.exe` for the first time when no installation profile exists.
* Production save management on the `Saves` page.
* Real online responses from AI Assistant.
* Production save and load support for trainer presets.
* Lag Remover.
* Some recommendations and filter buttons on Home.

These items may appear in the interface, but should not be treated as reliable production features yet.

## Further Learning

This guide covers ordinary-user operation and basic scripting. For complete trainer scripts, start with the [Hexinton Engine technical documentation](../hexinton-engine-wiki/engine/). It covers memory scanning, hooks, Auto Assembler, package lifecycle, the Lua API, control configuration, and testing by topic.

* [Hexinton Engine technical documentation](../hexinton-engine-wiki/engine/)
* [Script packages and lifecycle](../hexinton-engine-wiki/engine/script-packages.md)
* [Lua scripting](../hexinton-engine-wiki/engine/lua-scripting.md)
* [Memory and address resolution](../hexinton-engine-wiki/engine/memory-and-addresses.md)
* [AOB scanning](../hexinton-engine-wiki/engine/aob-scanning.md)
* [Hooks and Auto Assembler](../hexinton-engine-wiki/engine/hooks-and-auto-assembler.md)
* [Testing and failure handling](../hexinton-engine-wiki/engine/testing.md)

## Get Help

When reporting a problem, provide:

* The Hexinton Mod client version.
* The Windows version.
* The game name and version.
* The game installation source, such as Steam or an independent installation.
* The steps taken before the problem occurred.
* The complete error message shown on the page.
* Session status from the `Debug` page.

Do not publicly upload account tokens, private file paths, save files, or other sensitive information.

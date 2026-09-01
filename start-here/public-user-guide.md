# Public User Guide

Status: current user guide and basic scripting introduction.

This guide is for users who have installed and can open the Hexinton Mod client. It explains how to select a game, launch it, connect a trainer, and use mod features. It also introduces how to write a basic script in Studio. It does not cover source downloads, compilation, or client development. For more advanced mod development, see [Further learning](#further-learning).

## Before You Start

Before using Hexinton Mod, confirm that:

* You are using Windows.
* The Hexinton Mod client is installed and opens normally.
* The target game is installed locally.
* The target game appears in the client's `My Games` list.
* You understand the rules and anti-cheat policy for online games.

Mods may modify a game process or save data. Back up important saves before use and only use trainers and mods that you trust. Do not use them in online modes that prohibit game modification.

## Main Interface

After opening the client, the left navigation bar contains these main entries:

| Entry | Purpose |
| --- | --- |
| Home | View registered games and recently used games |
| Creator | Open creator features |
| Debug | View application and connection status for troubleshooting |
| Settings | Change the language and trainer feedback sounds |
| My Games | View games registered with Hexinton Mod on this machine |

The search box at the top of Home searches games registered with the local client. It does not search for or download games or mods from the network.

## Open a Game Trainer

1. Find the target game in `Home` or `My Games`.
2. Select the game card to open its trainer page.
3. Wait for the client to prepare the game information and trainer content.

The trainer page usually contains:

* `Mods`: Shows trainer features available for the current game.
* `Saves`: Content in the current version may be demonstration data and should not be treated as a production save-management feature.
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

```
example.health/
	package.json
	lua/main.lua
```

The directory name must exactly match the `id` in `package.json`. Use English letters, numbers, hyphens, or dots in file and directory names, and avoid spaces.

### Write package.json

`package.json` tells Hexinton Mod which scripts to load and which features users can execute. The following complete example is a useful starting point:

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
### Save, Apply, and Test
```
1. Edit `package.json` or a Lua file in the Studio editor.
2. Wait for the editor to save automatically, or use its save action.
3. Return to the game trainer page.
4. If `Apply package changes` appears, select it.
5. Wait for `Applying package changes...` to finish.
6. Start the game and wait until the session is connected.
7. Run your new action or enable its toggle in the `Mods` page.
8. Disable the feature after testing, then modify the script.
| `readInteger(address)` | Read an integer. |
Studio provides `Widget Preview` for trainer controls, but a preview does not replace testing with the real game. Scripts that read or write memory, scan, patch, or hook a process must be verified in a recoverable test environment.
| `readQword(address)` / `writeQword(address, value)` | Read or write a 64-bit integer. |
### Common Script Errors
| `AOBScan(pattern)` | Scan for a byte pattern. |
| Symptom | Check |
| --- | --- |
| Package is not shown | The directory name matches `id`, and files are at their declared paths. |
| Package cannot load | `package.json` uses strict JSON syntax. |
| Button fails after selection | `entrySymbol` exactly matches the Lua function name. |
| Feature cannot be disabled | `disable()` cleans up resources created by `enable()`. |
| Address read fails | The game is connected, and the module name and address expression are correct. |
| Parameter is empty | `parameterSchema` matches the parameter name read by Lua. |
| Change is not visible | The files were saved and `Apply package changes` was selected. |
Defining a function in Lua does not automatically add a button to the trainer page. To expose an action, declare it in `package.json` under `runnables` and keep these three fields consistent:
Fix script errors while the game is not running when possible. Do not hide errors by repeatedly selecting an execute button, and do not run memory-modification scripts found online unless you understand what they do.
```
## Trainer Cannot Be Used
runnables[].entrySymbol -> 实际 Lua 函数名
If trainer controls are locked or unavailable, check the following in order:
```
1. The game is running.
2. Hexinton Mod is not still preparing the session.
3. The client is connected to the correct game process.
4. The game version is compatible with the trainer.
5. The trainer package synchronized and loaded successfully.
6. The game is not running under a different Windows user or higher privilege level.
	"kind": "action",
If the game runs with administrator privileges, try running Hexinton Mod with the same privileges. Do not disable system security features without understanding the risks.
	"entryFile": "lua/main.lua",
## Apply Package Updates
}
If the page shows `Apply package changes`, local trainer files have changed but are not yet applied to the current session.

1. Make sure the current operation has completed.
2. Select `Apply package changes`.
3. Wait for `Applying package changes...` to finish.
4. Confirm that trainer controls become available again.
1. 在 Studio 编辑器中修改 `package.json` 或 Lua 文件。
Ordinary users usually do not see this message. It mainly appears after local trainer content is edited or updated.
3. 返回游戏训练器页面。
## Settings
5. 等待 `Applying package changes...` 完成。
Open `Settings` from the left navigation to change:
7. 在 `模组` 页面执行你新增的动作或打开开关。
* The client display language.
* Trainer operation feedback sounds.
* Sound volume.
* The interval between consecutive feedback sounds.
* Feedback sound preview.

Sounds only report trainer operation status. They do not mean that an in-game modification will remain active. Use the trainer result and in-game behavior as the final indication.
| ------------ | ----------------------------------- |
## Stop Using the Trainer
| package 无法加载 | `package.json` 是否为严格 JSON           |
At the end of a session, use this order:
| 启用后无法关闭      | `disable()` 是否清理了 `enable()` 创建的资源  |
1. Disable persistent trainer features that are enabled.
2. Wait for the disable operations to complete.
3. Exit the game normally.
4. Wait for the client to show that the game is disconnected.
5. Close Hexinton Mod.

This gives the trainer an opportunity to clean up hooks, timers, memory allocations, and other session resources.

## Frequently Asked Questions

### Hexinton Mod Cannot Find the Game
2. Hexinton Mod 是否仍在准备会话；
If you see `Hexinton Mod can't find the game.`:
4. 当前游戏版本是否与训练器兼容；
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
2. 等待关闭操作完成；
### Client Version Is Out of Date
4. 等待客户端显示游戏已经断开；
If you see `This Hexinton Mod version is out of date.`, install the latest client version provided by the maintainer before using it again.

这样可以让训练器有机会清理已经创建的钩子、计时器、内存分配和其他会话资源。

## 常见问题

### Hexinton Mod 找不到游戏

如果看到 `Hexinton Mod can't find the game.`：

1. 确认游戏已经安装；
2. 从 `Play` 菜单检查游戏 `.exe`；
3. 如果安装项显示 `Missing file`，重新选择正确的 `.exe`；
4. 确认游戏没有被移动到新的目录。

### Windows 无法启动游戏

如果看到 `Windows couldn't start the game.`：

* 尝试从游戏文件夹或官方启动器手动启动游戏；
* 检查游戏文件是否完整；
* 检查安全软件是否阻止客户端启动游戏；
* 重新选择正确的游戏 `.exe`。

### 启动链接无效

如果看到 `The game's launch link isn't working.`，说明保存的 URI 或启动器链接已经失效。请使用有效的安装配置，或从官方启动器手动启动游戏。

### 游戏已启动，但训练器仍然锁定

* 等待游戏完成加载；
* 确认启动的是受支持的游戏版本；
* 确认没有同时运行多个同名游戏进程；
* 打开“调试”页面查看会话状态；
* 重启游戏和 Hexinton Mod 后重试。

### 操作超时

如果看到 `Hexinton Mod took too long to respond.`，不要立即连续重试。先检查游戏是否卡死或正在切换场景，等待几秒后再执行一次。

### 当前操作不可用

如果看到 `This action isn't available right now.`，通常表示游戏尚未连接、会话正在更新，或当前训练器没有提供该操作。等待状态稳定后重试。

### 客户端版本过旧

如果看到 `This Hexinton Mod version is out of date.`，请安装维护者提供的最新客户端版本后再使用。

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

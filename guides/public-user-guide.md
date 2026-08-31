# Hexinton Mod 用户手册

状态：当前用户操作指南和基础脚本入门。

本文面向已经安装并能打开 Hexinton Mod 客户端的普通用户，介绍如何选择游戏、启动游戏、连接训练器并使用模组功能，也介绍如何使用 Studio 编写第一个基础脚本。本文不包含源码下载、编译或客户端程序搭建内容。本文包含基础脚本入门；更复杂的模组开发请参考“进一步学习”。

## 使用前须知

使用 Hexinton Mod 前，请确认：

- 你使用的是 Windows 系统；
- Hexinton Mod 客户端已经安装并能正常打开；
- 目标游戏已经安装在本机；
- 目标游戏已经出现在客户端的“我的游戏”列表中；
- 你了解在线游戏的使用规则和反作弊政策。

模组可能修改游戏进程或存档。建议在使用前备份重要存档，并仅使用你信任的训练器和模组。不要在禁止修改游戏的在线模式中使用。

## 认识主界面

打开客户端后，左侧导航栏包含以下主要入口：

| 入口 | 用途 |
| --- | --- |
| 首页 | 查看已注册游戏和最近使用的游戏 |
| 创作者 | 进入创作者相关功能 |
| 调试 | 查看应用和连接状态，主要用于问题排查 |
| 设置 | 修改语言和训练器反馈音效 |
| 我的游戏 | 查看本机已经注册到 Hexinton Mod 的游戏 |

首页顶部的搜索框用于搜索已经注册到本机客户端的游戏。它不会搜索或下载网络上的游戏和模组。

## 打开游戏训练器

1. 在“首页”或“我的游戏”中找到目标游戏。
2. 点击游戏卡片，进入该游戏的训练器页面。
3. 等待客户端完成游戏信息和训练器内容的准备。

训练器页面通常包含：

- `模组`：显示当前游戏可用的训练器功能；
- `存档`：当前版本中的内容可能是演示数据，不建议将其作为正式存档管理功能使用；
- `Play`：启动游戏；
- `Studio`：打开脚本和训练器包编辑器，用于编写自己的脚本。

如果游戏没有出现在“我的游戏”中，当前版本没有面向普通用户的完整“添加新游戏”入口。请确认你使用的客户端版本已经为该游戏配置了注册信息。

## 选择游戏安装位置

如果一个游戏存在多个安装位置，可以通过 `Play` 右侧的下拉按钮选择要启动的版本。

1. 点击 `Play` 右侧的下拉按钮。
2. 在 `Launch installation` 列表中选择正确的游戏安装项。
3. 选中的安装项会成为之后默认使用的安装位置。

列表中可能出现以下状态：

- `Missing file`：之前保存的游戏可执行文件已经被移动、删除或无法访问；
- `No installation profiles`：当前没有可用的游戏安装配置。

菜单中的 `Add game executable` 用于选择游戏的 Windows `.exe` 文件。请选择游戏本身的可执行文件，不要选择启动器快捷方式、文件夹或其他类型的文件。

当前版本存在一个限制：当游戏完全没有安装配置时，安装位置菜单可能无法打开。如果遇到这种情况，普通用户暂时无法从该页面完成首次添加，请使用已经配置好游戏信息的客户端版本或联系维护者。

## 启动并连接游戏

1. 进入目标游戏的训练器页面。
2. 确认 `Play` 使用的是正确安装位置。
3. 点击 `Play`。
4. 等待游戏启动。
5. 等待 Hexinton Mod 检测游戏进程并完成连接。

启动过程中，按钮可能显示：

| 状态 | 含义 |
| --- | --- |
| `Launching...` | 客户端正在发送并处理启动请求 |
| `Cancel launch` | 可以取消当前启动请求 |
| `Preparing session` | 正在准备训练器和游戏会话 |
| `Updating session...` | 正在同步当前会话内容 |
| `Running` | 游戏已经运行 |
| `Session sync failed` | 训练器内容同步失败 |
| `No installation` | 没有可用的游戏安装位置 |

看到 `Launch request accepted` 只表示客户端接受了启动请求，不代表训练器已经连接到游戏。请继续等待，直到训练器控件解除锁定。

游戏已经由 Steam、其他启动器或用户手动启动时，客户端仍需要检测到正确的目标进程并完成附加。只有连接完成后，游戏内修改功能才可使用。

## 使用训练器功能

连接成功后，打开 `模组` 标签。页面显示的功能由当前游戏已经同步并成功加载的训练器包提供。

### 开关功能

开关通常用于启用或禁用某项持续效果，例如无限生命或无限资源。

1. 打开对应开关。
2. 等待操作完成提示。
3. 需要恢复游戏原始行为时，关闭该开关。

不要在命令仍在执行时反复点击。关闭游戏前，建议先关闭已经启用的持续功能。

### 数值功能

数值控件用于设置数量、倍率或其他参数。

1. 输入或调整数值。
2. 确认数值位于控件允许的范围内。
3. 点击对应的执行按钮，或等待该控件按设计提交。

某些控件提供加减按钮或下拉选项。提交成功后，游戏中的实际值可能稍后才会更新。

### 动作功能

带有 `Execute` 按钮的项目表示一次性操作，例如添加物品、传送或刷新数据。

1. 按要求填写参数。
2. 点击 `Execute`。
3. 等待成功或失败结果。

一次成功提示只代表该次命令完成，不表示功能会持续生效。需要持续效果时，应使用对应的开关功能。

### 只读状态

部分项目只显示游戏中的当前值，不能编辑。它们会通过训练器状态更新自动刷新。命令刚完成时，显示值可能短暂保留旧内容，请等待下一次状态更新。

### 自定义界面

部分训练器会显示专用面板或小型应用，例如物品搜索、传送点列表或颜色选择器。它们只能调用该训练器明确授权的命令和数据。

当游戏尚未连接时，自定义界面会保持锁定。连接成功后再进行操作。

## 编写自己的脚本

Hexinton Mod 使用 package 来组织脚本。一个 package 至少包含一个 `package.json` 和一个 Lua 文件。你不需要修改客户端程序本身，只需要在 `Studio` 中编辑 package 文件。

### 打开 Studio

1. 打开目标游戏的训练器页面。
2. 点击页面顶部的 `Studio`。
3. 如果客户端提示只能在桌面应用中打开，请使用 Hexinton Mod 桌面客户端，而不是普通浏览器。
4. 在 Studio 中选择要编辑的本地 package。
5. 如果 package 是同步内容并显示为只读，选择 `Create Local Copy...`，然后编辑副本。

同步 package 通常不能直接修改。创建本地副本可以避免直接覆盖同步内容，也方便你保留自己的修改。

### 创建 package 文件

在 Studio 中使用 `New Package...` 创建新 package，或在本地副本中建立如下文件结构：

```text
example.health/
	package.json
	lua/main.lua
```

目录名必须和 `package.json` 中的 `id` 完全一致。文件名和目录名使用英文、数字、短横线或点号，避免使用空格。

### 编写 package.json

`package.json` 告诉 Hexinton Mod 要加载哪些脚本，以及用户可以执行哪些功能。下面是一个可以作为起点的完整示例：

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

这几个字段最重要：

- `id`：package 的唯一 ID，必须和目录名一致；
- `version`：package 版本号；
- `runtime.files`：所有要加载的运行时文件；
- `runnables`：脚本提供的功能列表；
- `kind`：功能类型，常用的是 `enable`、`disable` 和 `action`；
- `entrySymbol`：Lua 文件中实际存在的函数名；
- `parameterSchema`：动作接受的参数格式。

JSON 必须使用严格格式：属性名和文本值都要使用双引号，最后一个属性后不要加逗号。

### 编写 Lua 脚本

在 `lua/main.lua` 中写入：

```lua
local health = 100
local maximumHealth = 100

function enable()
		-- 在这里安装属于本 package 的功能。
end

function disable()
		-- 在这里移除 enable 创建的钩子、计时器和其他资源。
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

脚本规则：

- `enable()` 在 package 开始工作时执行；
- `disable()` 在停止 package 时执行，必须负责清理资源；
- `action` 函数接收一个参数表，例如 `arguments.amount`；
- 函数可以返回数字、字符串、布尔值或由这些值组成的表；
- 不要把密钥、账号信息或私人文件内容写进 package；
- 只操作你明确知道属于目标游戏的地址、符号和进程。

### 常用 Lua 操作

脚本连接到游戏后，可以使用以下常用函数：

```lua
local pid = findProcess("game.exe")
local processHandle = openProcess(pid)

local address = processHandle:getAddress("game.exe+0x1234")
local currentValue = processHandle:readInteger(address)
processHandle:writeInteger(address, 999)
```

常用函数包括：

| 函数 | 用途 |
| --- | --- |
| `findProcess(name)` | 查找游戏进程 ID |
| `openProcess(pid)` | 打开指定进程会话 |
| `getAddress(expression)` | 解析模块、符号或地址表达式 |
| `readInteger(address)` | 读取整数 |
| `writeInteger(address, value)` | 写入整数 |
| `readQword(address)` / `writeQword(address, value)` | 读取或写入 64 位整数 |
| `readFloat(address)` / `writeFloat(address, value)` | 读取或写入浮点数 |
| `AOBScan(pattern)` | 按字节特征扫描地址 |
| `registerSymbol(name, address)` | 注册当前 package 的符号 |
| `autoAssemble(text)` | 执行 Auto Assembler 操作 |
| `createTimer(...)` | 创建定时任务 |
| `sleep(milliseconds)` | 暂停当前脚本 |

推荐优先使用 `openProcess(...)` 返回的句柄进行读写。例如 `processHandle:readInteger(...)` 明确表示操作哪个进程，避免误操作其他进程。地址表达式可以使用类似 `game.exe+0x1234` 的格式。

### 把脚本功能显示在训练器中

仅仅在 Lua 中写出函数，不一定会自动在训练器页面显示按钮。要让用户看到一个动作，必须在 `package.json` 的 `runnables` 中声明它，并确保以下三项一致：

```text
runnables[].entryFile  -> 实际 Lua 文件
runnables[].entrySymbol -> 实际 Lua 函数名
runnables[].runtime -> lua
```

例如：

```json
{
	"id": "heal",
	"kind": "action",
	"runtime": "lua",
	"entryFile": "lua/main.lua",
	"entrySymbol": "heal"
}
```

然后在训练器页面中，这个动作会由已加载的训练器界面显示。控件的具体外观取决于 package 提供的训练器界面配置。

### 保存、应用和测试

1. 在 Studio 编辑器中修改 `package.json` 或 Lua 文件。
2. 等待编辑器自动保存，或使用编辑器提供的保存操作。
3. 返回游戏训练器页面。
4. 如果出现 `Apply package changes`，点击它。
5. 等待 `Applying package changes...` 完成。
6. 启动游戏并等待状态变为已连接。
7. 在 `模组` 页面执行你新增的动作或打开开关。
8. 测试完成后关闭功能，再修改脚本。

Studio 提供 `Widget Preview` 预览训练器控件，但预览不能代替真实游戏测试。涉及内存读写、扫描、补丁或钩子的脚本，必须在可恢复的测试环境中验证。

### 常见脚本错误

| 现象 | 检查内容 |
| --- | --- |
| package 不显示 | 目录名是否等于 `id`，文件是否位于声明的路径 |
| package 无法加载 | `package.json` 是否为严格 JSON |
| 按钮点击后失败 | `entrySymbol` 是否和 Lua 函数名完全一致 |
| 启用后无法关闭 | `disable()` 是否清理了 `enable()` 创建的资源 |
| 读取地址失败 | 游戏是否已连接，模块名和地址表达式是否正确 |
| 参数为空 | `parameterSchema` 和 Lua 中读取的参数名是否一致 |
| 修改没有出现 | 是否保存文件并点击 `Apply package changes` |

脚本错误应先在游戏未运行状态下修复。不要通过反复点击执行按钮来掩盖错误，也不要在不了解指令含义时直接运行网上找到的内存修改脚本。

## 训练器无法操作

训练器控件处于锁定或不可点击状态时，依次检查：

1. 游戏是否已经运行；
2. Hexinton Mod 是否仍在准备会话；
3. 客户端是否已经连接到正确的游戏进程；
4. 当前游戏版本是否与训练器兼容；
5. 训练器包是否同步或加载失败；
6. 游戏是否以不同的 Windows 用户或更高权限运行。

如果游戏使用管理员权限启动，请尝试让 Hexinton Mod 使用相同权限运行。不要在不了解风险的情况下关闭系统安全功能。

## 应用包更新

如果页面显示 `Apply package changes`，表示本地训练器文件已经发生变化，但尚未应用到当前会话。

1. 确保当前操作已经完成。
2. 点击 `Apply package changes`。
3. 等待 `Applying package changes...` 结束。
4. 确认训练器控件恢复可用。

普通用户通常不会看到此提示。它主要出现在本地训练器内容被编辑或更新后。

## 设置

进入左侧的“设置”页面，可以修改：

- 客户端显示语言；
- 训练器操作反馈音效；
- 音效音量；
- 连续提示音之间的时间间隔；
- 提示音预览。

音效仅用于反馈训练器操作状态，不代表游戏中的修改一定持续生效。最终状态仍以训练器结果和游戏内表现为准。

## 停止使用

结束本次使用时，建议按照以下顺序操作：

1. 关闭已经启用的持续训练器功能；
2. 等待关闭操作完成；
3. 正常退出游戏；
4. 等待客户端显示游戏已经断开；
5. 关闭 Hexinton Mod。

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

- 尝试从游戏文件夹或官方启动器手动启动游戏；
- 检查游戏文件是否完整；
- 检查安全软件是否阻止客户端启动游戏；
- 重新选择正确的游戏 `.exe`。

### 启动链接无效

如果看到 `The game's launch link isn't working.`，说明保存的 URI 或启动器链接已经失效。请使用有效的安装配置，或从官方启动器手动启动游戏。

### 游戏已启动，但训练器仍然锁定

- 等待游戏完成加载；
- 确认启动的是受支持的游戏版本；
- 确认没有同时运行多个同名游戏进程；
- 打开“调试”页面查看会话状态；
- 重启游戏和 Hexinton Mod 后重试。

### 操作超时

如果看到 `Hexinton Mod took too long to respond.`，不要立即连续重试。先检查游戏是否卡死或正在切换场景，等待几秒后再执行一次。

### 当前操作不可用

如果看到 `This action isn't available right now.`，通常表示游戏尚未连接、会话正在更新，或当前训练器没有提供该操作。等待状态稳定后重试。

### 客户端版本过旧

如果看到 `This Hexinton Mod version is out of date.`，请安装维护者提供的最新客户端版本后再使用。

## 当前版本限制

以下界面或功能在当前版本中尚未形成完整的普通用户流程：

- 在线模组目录、下载、安装和卸载；
- 从客户端注册一个全新的游戏；
- 没有任何安装配置时首次添加游戏 `.exe`；
- `存档` 页面的正式存档管理；
- AI Assistant 的真实在线回答；
- 训练器预设的正式保存和加载；
- Lag Remover；
- 首页部分推荐内容和筛选按钮。

这些内容可能出现在界面中，但不应被视为当前可依赖的正式功能。

## 进一步学习

本文包含普通用户操作说明和基础脚本入门。要真正编写完整的训练器脚本，请先阅读英文 [Hexinton Engine technical documentation](engine/README.md)。其中按主题说明内存扫描、Hook、Auto Assembler、Package 生命周期、Lua API、控件配置和测试方法。

- [Hexinton Engine technical documentation](engine/README.md)
- [Script packages and lifecycle](engine/script-packages.md)
- [Lua scripting](engine/lua-scripting.md)
- [Memory and address resolution](engine/memory-and-addresses.md)
- [AOB scanning](engine/aob-scanning.md)
- [Hooks and Auto Assembler](engine/hooks-and-auto-assembler.md)
- [Testing and failure handling](engine/testing.md)

## 获取帮助

报告问题时，请提供：

- Hexinton Mod 客户端版本；
- Windows 版本；
- 游戏名称和游戏版本；
- 游戏安装来源，例如 Steam 或独立安装；
- 问题发生前的操作步骤；
- 页面显示的完整错误信息；
- “调试”页面中与本次会话相关的状态信息。

不要公开上传账号令牌、私人文件路径、存档文件或其他敏感信息。
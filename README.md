# MCDK 联机大厅插件

适用于网易我的世界的 MCDK 联机大厅插件，提供客户端与服务器双端日志查询，支持游戏调试、作品发布与自测管理、商品管理及运营配置。

## 功能概览

- 查询客户端与服务器双端日志，支持增量读取、服务器错误筛选和报错上下文查看。
- 通过官方调试接口执行大厅客户端 Python，直接返回类型、文本表示和 JSON 值。
- 启动联机大厅自测，自动创建房间并打开游戏，检查登录、启动和进服问题。
- 管理联机大厅作品，支持新建草稿、上传包体、更新免机审自测、提交审核和删除草稿。
- 新建商品、更新商品自测、提交商品审核和删除商品草稿。
- 管理自测服与正式服的存储排序 key 和运营配置。
- 提供 MCP 工具，可通过 AI 客户端或 MCDK 命令行调用。

## 安装与配置

需要 Windows、[MCDK](https://github.com/GitHub-Zero123/MCDevTool) 和 MCS。使用时无需打开 MCS。

1. 从 [Releases](https://github.com/mofaner/mcdk_lobby_plugin/releases) 下载并解压插件，保留文件夹内的全部文件。
2. 将 `mcdk_lobby_plugin` 文件夹放入 MCDK 安装目录的 `plugins` 文件夹。
3. 将 [配置示例](mcdev.example.json) 中的插件条目合并到项目 `.mcdev.json` 的 `plugins` 数组，填写插件绝对路径和作品 ID。保留原有配置，不要覆盖整个文件。

| 配置项 | 说明 |
| --- | --- |
| `plugins[].path` | 插件所在目录的绝对路径 |
| `config.item_id` | 联机大厅作品 ID |
| `config.launch_lobby` | `true` 启用大厅测试，`false` 恢复本地测试 |
| `config.studio_directory` | MCS 安装目录；留空时查找各盘根目录下的 `MCStudio`，自定义位置需填写 |
| `config.room_name` | 测试房间名称，默认“联机大厅自测” |
| `config.max_players` | 房间人数，默认 4 |

项目的 `auto_join_game` 设为 `true`。`game_executable_path` 指向网易游戏客户端的 `Minecraft.Windows.exe`，支持原始数字版本目录及 `PCLauncher_x64` 目录。数字目录按网易版本匹配服务器；`PCLauncher_x64` 读取可执行文件中的引擎版本，并确认平台支持对应服务器版本后才创建房间。请保留游戏原有的目录名称。

### 客户端版本选择

**多人联机测试推荐使用 `PCLauncher_x64`，联机表现更加稳定。**

`PCLauncher_x64` 是 MCS 云端测试所使用的客户端版本。

客户端代码执行入口为 [`lobby_execute_code`](#执行大厅客户端代码)。

| 客户端 | 使用建议 | 多人联机表现 |
| --- | --- | --- |
| 原始数字版本目录 | 已有测试环境可继续使用 | 稳定性不如 `PCLauncher_x64`，多人测试时建议切换 |
| `PCLauncher_x64` | 双账号及多人联机测试优先选择 | 多人联机更加稳定 |

更新插件前，先结束测试并关闭登录窗口，再替换插件文件。多个项目可以共用同一份插件。

## 开始测试

作品需要已有可用的自测版本。在项目目录执行：

```powershell
mcdk run --project . --session lobby-plugin --lane full --background
```

首次使用时，在弹出的窗口登录网易账号。之后会自动恢复有效登录，并创建房间、启动游戏。首次进入或资源更新时，需要在游戏中确认下载资源包。

**测试期间保持登录窗口运行，可以最小化。** 换号前先结束测试，再点击窗口中的“切换账号”。

网络代理可能导致游戏进服失败以及资源包下载缓慢。

结束测试：

```powershell
mcdk stop --session lobby-plugin --project .
```

## 双账号联机测试

两个会话使用相同的客户端版本，推荐均使用 `PCLauncher_x64`，具体区别见上文“客户端版本选择”。

先按上节启动房主会话 `lobby-plugin`，房间人数上限至少设为 2。第二个窗口使用有权参与该作品多人测试的另一账号。两个客户端使用不同的 full Lane 和调试端口。

在同一项目目录执行以下 PowerShell 命令启动客号。它会读取房主当前的房间号，核对作品并加入房间。

```powershell
$plugins = (Get-Content -Raw .mcdev.json | ConvertFrom-Json).plugins
$lobby = $plugins | Where-Object id -eq 'com.zy.mcdk-lobby'
$lobby.config | Add-Member -NotePropertyName guest_account -NotePropertyValue $true -Force
$guestConfig = 'plugins=' + (ConvertTo-Json -InputObject $plugins -Compress -Depth 16)
mcdk run --project . --session lobby-guest --lane full --set $guestConfig --set ptvsd_debugger.port=56789 --background
```

通过会话名分别读取客户端输出。每个会话单独保存自己的 `next` 游标，返回的 `account_slot` 和 `game_pid` 可用于核对来源。

```powershell
mcdk call lobby_local_logs --project . --session lobby-plugin --args '{"limit":30}'
mcdk call lobby_local_logs --project . --session lobby-guest --args '{"limit":30}'
mcdk stop --project . --session lobby-guest
mcdk stop --project . --session lobby-plugin
```

先停止客号会话可让客号单独退房，房主可以继续测试。停止房主会话会关闭房间。

## 上传包体与免机审自测

单独上传自测包时，在插件安装目录使用以下命令。登录窗口就绪并完成账号登录后即可上传，无需运行 `mcdk run`、打开游戏或创建房间。

```powershell
.\lobby-login.exe login
.\lobby-login.exe publish-upload 作品ID D:/packs/world.zip
.\lobby-login.exe publish-status
```

上传工具使用**已经打包好的 ZIP**。ZIP 的唯一顶层目录内需包含 `level.dat`、`db/`、`behavior_packs/` 和 `resource_packs/`。

上传会替换该作品的自测包并提交免机审自测。正式审核通过 `lobby_item_review` 单独提交。上传期间保持登录窗口运行，避免同时在开发者平台编辑该作品。

平台任务通过 `publish-status` 查看进度与结果。已有大厅游戏会话也可调用 MCP 工具 `lobby_publish_upload` 和 `lobby_publish_status`。`running` 为执行中，`succeeded` 为完成，`submitted` 为已提交、等待确认，`failed` 为失败。失败后先检查作品当前状态，再决定是否重新操作。

## 作品、商品与运营管理

新建作品和商品会保存为草稿，自测、提审分别使用对应工具。新建大厅时需要确定是否开启内购，创建后无法修改。商品提审完成后，仍需按平台流程上架。

存储排序 key 支持 `self_test`（自测服）和 `obt`（正式服）。运营配置使用 `self_test_config`、`obt_config` 填写 JSON，并通过 `config_note` 填写配置说明；未填写的环境保持原配置。正式服配置的生效以平台审核结果为准。

管理写操作可通过 `dry_run=true` 预先检查参数和操作内容。各工具的完整参数可直接查询：

```powershell
mcdk help lobby_create_item --project .
mcdk help lobby_goods_create --project .
mcdk help lobby_operation_config --project .
```

## 查看状态与日志

```powershell
mcdk call lobby_status --session lobby-plugin --project .
mcdk call lobby_diagnose --session lobby-plugin --project .
mcdk call lobby_local_logs --session lobby-plugin --project . --args '{"limit":30}'
mcdk call lobby_server_logs --session lobby-plugin --project . --args '{"limit":30,"errors_only":true}'
```

日志返回的 `next` 可作为下一次读取的 `since`；客户端与服务器日志分别记录读取位置。`errors_only=true` 用于筛选服务器错误日志。

启动状态不能代替游戏画面确认。若一直等待进服，请检查游戏是否弹出资源下载提示；插件无法读取资源下载进度。

## 执行大厅客户端代码

在 `.mcdev.json` 顶层启用官方调试端口，然后重新启动游戏：

```json
"ptvsd_debugger": {
    "enabled": true,
    "ip": "127.0.0.1",
    "port": 56788
}
```

进服后调用 `lobby_execute_code`。参数沿用原生 `execute_code` 的 `code`、`is_client`、`direct_return`，默认在客户端同步执行并返回结果：

```powershell
mcdk call lobby_execute_code --session lobby-plugin --args '{"code":"6*7"}'
mcdk call lobby_execute_code --session lobby-plugin --args '{"code":"import mod.client.extraClientApi as clientApi; _result = {\"player_id\": clientApi.GetLocalPlayerId()}"}'
```

单个表达式直接返回值，多行代码通过 `_result` 返回；未赋值时返回 `None`。结果包含 `return_type`、`return_repr` 和 `return_value`，元组转换为 JSON 数组，其他对象返回类型与表示。变量保留在独立的调试命名空间内，不使用当前暂停模块的变量；命名空间持续到游戏进程结束。

`direct_return=false` 只确认任务已安排到客户端下一帧，执行结果通过 `get_latest_logs` 中的 `[LobbyCode]` 查看，不生成额外日志文件。工具只支持客户端，`is_client=false` 会报错，不能执行大厅云端服务器代码，也不会替换 MCDK 原生 `execute_code`。

执行时会短暂暂停客户端主线程，并在结束后恢复和断开调试器；不要同时用 VS Code 等调试器连接同一端口。连接与协议等待有超时，但不能强制终止正在执行的 Python，因此不要提交无限循环、长时间阻塞或等待客户端下一帧才会完成的同步代码。遇到执行状态未知、线程恢复失败或返回值过大时，代码可能已经生效，应先检查游戏状态，不要直接重试有副作用的操作。单次返回的 JSON 上限为 200000 字节。

## MCP 工具

### 账号与房间

| 工具 | 功能 |
| --- | --- |
| `lobby_login` | 登录网易账号 |
| `lobby_status` | 查看登录和测试状态 |
| `lobby_diagnose` | 检查启动与进服问题 |
| `lobby_switch_account` | 切换账号 |
| `lobby_retry_login` | 重新登录 |
| `lobby_create_room` | 创建自测房间 |
| `lobby_room` | 查看房间状态 |
| `lobby_stop_room` | 关闭测试房间 |
| `lobby_local_logs` | 查看客户端日志 |
| `lobby_server_logs` | 查看服务器日志 |

### 作品与商品

| 工具 | 功能 |
| --- | --- |
| `lobby_publish_login` | 登录开发者平台 |
| `lobby_publish_status` | 查看平台任务进度与结果 |
| `lobby_publish_check` | 查看作品权限与自测状态 |
| `lobby_publish_upload` | 上传包体并更新免机审自测 |
| `lobby_publish_options` | 查询作品分类、图片要求和商品收费类型等选项 |
| `lobby_create_item` | 新建联机大厅作品 |
| `lobby_item_review` | 提交作品审核 |
| `lobby_item_delete` | 删除作品草稿 |
| `lobby_goods_list` | 查看商品列表与详情 |
| `lobby_goods_create` | 新建商品 |
| `lobby_goods_self_test` | 更新商品自测 |
| `lobby_goods_review` | 提交商品审核 |
| `lobby_goods_delete` | 删除商品草稿 |
| `lobby_storage_sort_keys` | 管理存储排序 key |
| `lobby_operation_config` | 管理运营配置 |

### 其他使用方式

未启动游戏时，也可以在插件目录使用独立命令：

```powershell
./lobby-login.exe login
./lobby-login.exe publish-upload 作品ID D:/packs/world.zip
./lobby-login.exe publish-status
```

Cookie 登录作为备选方式：调用 `lobby_publish_login`，参数为 `{"method":"cookie"}`，在弹出的窗口中导入开发者平台 Cookie。有效登录会保存供下次使用，过期后需要重新导入。Cookie 仅在登录窗口填写，不要放入项目配置或聊天。

## 相关项目与文档

- [MCDK](https://github.com/GitHub-Zero123/MCDevTool)：网易我的世界开发与调试工具。
- [联机大厅调试与多人测试文档](https://mc.163.com/dev/mcmanual/mc-dev/mcguide/26-联机大厅/20-联机大厅调试与多人测试文档.html?catalog=1)：官方大厅测试说明。

## 普通机审自测与反馈

已有包体可用 `lobby_machine_self_test` 提交普通机审自测。作品已经在自测时，传 `replace_self_test: true` 会先撤销旧自测，再送审同一版本。切换期间平台可能影响现有玩家连接。任务状态 `submitted` 表示已受理。

`lobby_review_feedback` 返回作品当前状态、版本、带时间的审核反馈和最近一页中标题匹配该作品的系统消息。传 `read_messages: true` 可读取这些消息的正文，平台会将它们标记为已读。应按提交时间区分本轮反馈和历史消息。

房间正在由旧版插件运行时，可使用新版本的独立平台命令，由网易 SDK 登录开发者平台：

```powershell
.\lobby-login.exe --platform-once D:/MCStudio publish_machine_test 作品ID --replace-self-test
.\lobby-login.exe --platform-once D:/MCStudio publish_feedback 作品ID --read-messages
```

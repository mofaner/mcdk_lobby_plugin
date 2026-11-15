# MCDK 联机大厅插件

适用于网易我的世界的 MCDK 联机大厅插件，提供客户端与服务器双端日志查询，支持游戏调试、作品发布与自测管理、商品管理及运营配置。

## 功能概览

- 查询客户端与服务器双端日志，支持增量读取、服务器错误筛选和报错上下文查看。
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

项目的 `auto_join_game` 设为 `true`。`game_executable_path` 使用网易游戏客户端路径，可通过 `mcdk games` 查询；请保留游戏原有的版本目录名称。

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

## 上传包体与免机审自测

使用有作品管理权限的账号登录后，指定作品 ID 和地图 ZIP 的绝对路径：

```powershell
mcdk call lobby_publish_upload --session lobby-plugin --project . --args '{"item_id":"你的作品ID","zip_path":"D:/packs/world.zip"}'
mcdk call lobby_publish_status --session lobby-plugin --project .
```

上传工具使用**已经打包好的 ZIP**。ZIP 的唯一顶层目录内需包含 `level.dat`、`db/`、`behavior_packs/` 和 `resource_packs/`。

上传会替换该作品的自测包并提交免机审自测。正式审核通过 `lobby_item_review` 单独提交。上传期间保持登录窗口运行，避免同时在开发者平台编辑该作品。

平台任务通过 `lobby_publish_status` 查看进度与结果：`running` 为执行中，`succeeded` 为完成，`submitted` 为已提交、等待确认，`failed` 为失败。失败后先检查作品当前状态，再决定是否重新操作。

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

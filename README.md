# mcdk_lobby_plugin

创建网易联机大厅自测房间，通过 mcdk 启动客户端，分别读取本地和云端日志。运行时不需要打开 MCS，但需要其安装目录中的网易 SDK。

## 安装

需要 Windows、MCS、mcdk，以及已有可用自测版本的联机大厅作品。

从 [Releases](https://github.com/mofaner/mcdk_lobby_plugin/releases) 下载压缩包，将 `mcdk_lobby_plugin` 文件夹放到 **MCDK 安装目录的 `plugins` 目录**，多个项目共用一份插件。例如 MCDK 位于 `D:/myProject/mcdk/mcdk.exe` 时，插件目录为 `D:/myProject/mcdk/plugins/mcdk_lobby_plugin/`，其中保留 `plugin.json`、插件 DLL 和 `lobby-login.exe` 等完整发布文件。

各项目复用现有的 `.mcdev.json`：将 `mcdev.example.json` 中的大厅插件条目合并到 `plugins` 数组，保留已有插件和其他配置；已有 `com.zy.mcdk-lobby` 条目时直接修改，不重复添加。示例仅包含插件扩展配置，不是完整的项目配置，不要用它覆盖 `.mcdev.json`。

填写插件配置中的作品 ID。`studio_directory` 可省略或留空：按盘符顺序查找各盘根目录下的 `MCStudio`，使用首个包含 `MPayCliCppWrapper.dll` 的目录；填写路径时只使用指定目录。自动搜索不扫描整盘，自定义安装位置需显式填写；找不到时会提示配置路径。`plugins[].path` 填写上述共享插件目录的**绝对路径**，按实际 MCDK 安装位置修改。游戏路径继续使用项目已有的 `game_executable_path`，可用 `mcdk games` 查询。大厅自动启动从游戏所在的四段数字版本目录（例如 `3.10.0.420447/Minecraft.Windows.exe`）获取网易客户端版本，无需配置 `client_version`，旧配置中的该字段不再参与自动启动。请保留原始版本目录；无法识别时会在登录和创建房间前报错。直接调用 `lobby_create_room` 同样自动识别版本。

大厅自测时将插件的 `config.launch_lobby` 设为 `true`，并确保项目顶层的 `auto_join_game` 为 `true`；恢复本地测试时，将 `config.launch_lobby` 设为 `false`。切换前先结束当前测试。

更新插件时，先结束正在使用它的测试并关闭登录窗口，再替换共享目录中的发布文件，所有引用它的项目共用新版。原来安装在项目内的插件也可以迁移到共享目录，并同步修改各项目的 `plugins[].path`。

## 开始与结束

```powershell
mcdk run --project . --session lobby-plugin --lane full --background
```

插件自动登录、创建房间并启动游戏。首次在弹出的网易窗口登录，之后自动恢复有效登录。首次下载新资源时，在游戏中选择下载并加入。

**不推荐开启代理的 TUN 模式进入游戏。** 网络不稳定可能导致进服失败或资源下载缓慢；遇到这些问题时，优先关闭 TUN 模式后重试。

**使用时保持登录窗口运行，可最小化；关闭会中断游戏连接。**

结束测试，在另一个终端执行：

```powershell
mcdk stop --session lobby-plugin --project .
```

这会结束客户端并关闭本工具的房间，保留登录供下次使用。

## 账号操作

需要换号时，先结束测试，再点击窗口中的“切换账号”。

运行中的 MCDK 会话使用 `mcdk call lobby_login`、`lobby_status`、`lobby_retry_login` 和 `lobby_switch_account`。`lobby_login` 默认使用项目的 MCS 目录，传入参数时覆盖项目值。

游戏未启动时，可在插件目录用以下独立命令排查登录：

```powershell
./lobby-login.exe login
./lobby-login.exe status
./lobby-login.exe retry
./lobby-login.exe switch
```

命令返回 JSON。换号会打开网易登录窗口；用 `status` 查询，`login_state=authenticated` 表示登录完成。测试运行期间禁止换号。登录命令未指定目录或传入空字符串时，按盘符顺序查找各盘根目录下的 `MCStudio`；自定义安装位置用 `--studio-directory '实际目录'` 指定。

## 日志与状态

在已配置并启用插件的项目目录执行 `mcdk help lobby_create_room` 可查看工具参数。也可用 `mcdk help lobby_create_room --project '项目路径'` 指定项目；仅将插件放入 MCDK 的 `plugins` 目录，不会让未配置插件的项目自动识别这些工具。

```powershell
mcdk call lobby_status --session lobby-plugin --project .
mcdk call lobby_diagnose --session lobby-plugin --project .
mcdk call lobby_local_logs --session lobby-plugin --project . --args '{"limit":30}'
mcdk call lobby_server_logs --session lobby-plugin --project . --args '{"limit":30}'
```

手动开房可执行 `mcdk call lobby_create_room --session lobby-plugin --args '{}'`，或通过 `--args '{"room_name":"临时测试","max_players":2}'` 覆盖本次设置。需先登录，且当前账号没有房间；该工具仅开房，不让已运行的客户端进服。房间名称默认“联机大厅自测”，人数默认 4。原调用中的 `name` 改为 `room_name`，不再接受 `client_version`。

日志返回 `next`，下一次作为 `since` 传入；两端游标独立。本地游标只属于本次 mcdk 进程，重启后重新读取。插件不自动导出日志；本地缓存有容量上限，`dropped` 表示旧记录已移出缓存。

| 工具 | 作用 |
| --- | --- |
| lobby_login | 打开登录程序，可选参数 studio_directory |
| lobby_status | 登录、房间、启动阶段与首次下载提示 |
| lobby_diagnose | 只读检查本地依赖、版本映射、认证及当前房间，指出优先处理的步骤 |
| lobby_switch_account | 与界面共用的账号切换流程 |
| lobby_retry_login | 与界面共用的登录重试流程 |
| lobby_create_room | 按项目配置创建并启动自测房间，可覆盖 item_id、room_name、max_players |
| lobby_room | 查询本工具房间及服务器状态 |
| lobby_local_logs | 读取 mcdk 采集的客户端输出 |
| lobby_server_logs | 按平台偏移量读取云端日志 |
| lobby_stop_room | 关闭并退出本工具房间 |

`game_pid` 和 `control_connected` 均不代表已进入世界。`world_state` 区分 `not_started`（未准备客户端）、`waiting_client`（等待控制连接）、`waiting_world`（控制通道已连接，等待入服通知）、`in_world`（收到入服通知）、`failed`（控制通道失败）和 `stopped`（已停止）。控制通道失败只表示失去该通道，不能证明游戏本身崩溃。部分客户端不报告控制通道，等待状态也不能证明未进服。

首次进入或资源更新可能弹出“下载资源包”，需要在游戏中确认并等待完成。插件不设置资源下载超时，不会因等待较久自动判断进服失败。当前已核对的协议没有下载弹窗和进度信号，`download_state=unobservable`；无法自动区分主菜单、等待下载确认和下载中，应查看游戏画面。远程房间没有 DebugMod 时，不能依赖 `mcdk wait` 判定进服。

本地日志的 `collection_state` 区分 `not_connected`（两个事件均未接入）、`partial`（只接入一个事件）、`subscribed_no_output`（已订阅但尚未收到输出）和 `receiving`（已收到输出）。`stdout_subscribed`、`stderr_subscribed` 表示具体订阅结果。`output_state=no_output` 只表示此游标之后暂无输出，不能断言游戏没有日志；云端日志与本地采集是两条独立通道。

云端日志新增 `text`，清理常见 HTML 展示标签、解码实体并保留换行、缩进和 `<module>`。原始 `entity` 保留不变。使用 `--args '{"limit":100,"errors_only":true}'` 筛选含错误关键词的批次；为了保留 traceback，匹配时保留整批上下文，不做逐行删除。`output_state` 区分无输出、无匹配错误和可用文本。分页可能截断堆栈，可增大 `limit` 或继续读取 `next`；过滤未命中也按原始批次推进游标。

`lobby_diagnose` 不启动登录程序、不创建或关闭房间。返回 `checks` 和 `blocked_at`，检查当前插件已通过宿主加载、SDK 必需文件、游戏路径与版本、平台版本映射、认证和当前房间。诊断可执行不代表全部检查通过，需读取各项 `state`。当前没有已核对的作品自测版本查询接口，该项标为 `unverified`，需在开发者平台确认；SDK 文件存在也不等于原生组件运行兼容性已验证。未登录时平台查询标为未验证。

参考：[MCDevTool SDK](https://github.com/GitHub-Zero123/MCDevTool/tree/main/sdk/plugin-sdk)、[官方大厅自测说明](https://mc.163.com/dev/mcmanual/mc-dev/mcguide/26-联机大厅/20-联机大厅调试与多人测试文档.html?catalog=1)。

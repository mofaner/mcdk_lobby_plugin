# mcdk_lobby_plugin

创建网易联机大厅自测房间，通过 mcdk 启动客户端，分别读取本地和云端日志。运行时不需要打开 MCS，但需要其安装目录中的网易 SDK。

## 安装

需要 Windows、MCS、mcdk，以及已有可用自测版本的联机大厅作品。

从 [Releases](https://github.com/mofaner/mcdk_lobby_plugin/releases) 下载压缩包，解压到项目目录，保留 `mcdk_lobby_plugin` 文件夹。将其中的 `mcdev.example.json` 复制为项目的 `.mcdev.json`，已有配置则合并示例中的插件配置。

填写游戏程序路径、插件目录、MCS 安装目录、作品 ID 和客户端版本号。游戏路径与版本可用 `mcdk games` 查询。

## 开始与结束

```powershell
mcdk run --project . --session lobby-plugin --lane full --background
```

插件自动登录、创建房间并启动游戏。首次在弹出的网易窗口登录，之后自动恢复有效登录。首次下载新资源时，在游戏中选择下载并加入。

**使用时保持登录窗口运行，可最小化；关闭会中断游戏连接。**

结束测试，在另一个终端执行：

```powershell
mcdk stop --session lobby-plugin --project .
```

这会结束客户端并关闭本工具的房间，保留登录供下次使用。

## 账号操作

需要换号时，先结束测试，再点击窗口中的“切换账号”。

正常启动不需要单独执行登录命令。游戏未启动时，可在插件目录执行：

```powershell
./lobby-login.exe login
./lobby-login.exe status
./lobby-login.exe retry
./lobby-login.exe switch
```

命令返回 JSON。换号会打开网易登录窗口；用 `status` 查询，`login_state=authenticated` 表示登录完成。测试运行期间禁止换号。MCS 不在默认目录 `D:/MCStudio` 时，登录命令加 `--studio-directory '实际目录'`。

## 日志与状态

```powershell
mcdk call lobby_status --session lobby-plugin --project .
mcdk call lobby_local_logs --session lobby-plugin --project . --args '{"limit":30}'
mcdk call lobby_server_logs --session lobby-plugin --project . --args '{"limit":30}'
```

日志返回 `next`，下一次作为 `since` 传入；两端游标独立。本地游标只属于本次 mcdk 进程，重启后重新读取。插件不自动导出日志；本地缓存有容量上限，`dropped` 表示旧记录已移出缓存。

| 工具 | 作用 |
| --- | --- |
| lobby_login | 打开登录程序，参数 studio_directory |
| lobby_status | 登录、房间与客户端进程信息 |
| lobby_switch_account | 与界面共用的账号切换流程 |
| lobby_retry_login | 与界面共用的登录重试流程 |
| lobby_create_room | 创建并启动自测房间 |
| lobby_room | 查询本工具房间及服务器状态 |
| lobby_local_logs | 读取 mcdk 采集的客户端输出 |
| lobby_server_logs | 按平台偏移量读取云端日志 |
| lobby_stop_room | 关闭并退出本工具房间 |

`game_pid` 不代表已进入世界。`control_connected` 仅指启动器控制通道；部分客户端不报告该通道，`world_state=unknown` 不能解释为未进服。远程房间没有 DebugMod 时，不能依赖 `mcdk wait` 判定进服，应核对实际画面或明确的游戏日志。

参考：[MCDevTool SDK](https://github.com/GitHub-Zero123/MCDevTool/tree/main/sdk/plugin-sdk)、[官方大厅自测说明](https://mc.163.com/dev/mcmanual/mc-dev/mcguide/26-联机大厅/20-联机大厅调试与多人测试文档.html?catalog=1)。

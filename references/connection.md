# 连接、安装与故障恢复

新版Windows应用提供固定工具入口 `PhysLabAgent.exe`，使用应用自带运行时，不需要用户另外安装Python或Node.js。首次打开1.13.0及更新版，在“Agent连接”启用本地连接；面板有MCP和Skill两种操作指引。

## 正常接入

优先用已配置的 `physlab_*` MCP工具。用户在应用点“复制MCP配置”，粘贴到Agent的设置/工具/MCP页面，保存后刷新工具；不是把JSON发进聊天。command是本机固定工具入口，args为 `["mcp"]`。每台电脑自行生成配置，不共享另一台电脑的绝对路径。安装目录变化后打开新版一次或点“检查并修复连接”更新登记。

本地命令用PowerShell：

```powershell
& (Join-Path ([Environment]::GetFolderPath('LocalApplicationData')) 'PhysLab/AgentBridge/PhysLabAgent.exe') connect --launch
```

将最后的connect替换为协议命令即可，如 `scene.apply --input <参数JSON>`。无需猜安装路径、工作目录、端口或令牌。固定入口不存在时让用户打开新版应用检查修复。1.12.x旧版兼容脚本 `scripts/physlab.py` 才需要Python3.10+，无需pip。

## 用户提供连接文件

`.physlab-connection.json` 是当前窗口会话的连接凭据，不是实验配置。让用户在目标窗口“Agent连接 → 导出当前窗口连接文件”，将本机完整路径交给Agent。推荐交给脚本读取，不把文件内容或token打印进聊天、日志或结果报告。

- MCP：`physlab_connect {"connectionFile":"本机完整路径"}`；连接成功后本次MCP进程后续工具复用该目标。
- 本地工具：`connect --connection <文件路径>`；后续命令仍传 `--connection`。
- 兼容Python：`python <技能目录>/scripts/physlab.py connect --connection <文件路径>`。

文件绕过默认目录和环境覆盖，先校验会话，再返回实际版本及实验名称。它只连接127.0.0.1，不执行文件中附带的程序路径或命令。不能让纯云端聊天访问用户电脑。关闭/重新启用连接、重启应用后旧文件失效；显式文件连接失败不能悄悄回退到其他窗口。

## 错误与下一步

- `APP_NOT_RUNNING`：查看错误返回的searchedDirectory，以及应用面板的实际连接目录；检查修复或提供连接文件。不能仅凭此错误声称程序没有运行。
- `NO_LAUNCHER`：当前目录缺少启动登记，不能推断程序未安装、开关未启用或数据位于Temp。查看环境覆盖和实际登记。
- `RUNTIME_UNAVAILABLE`：固定入口对应的应用已移动或删除；打开新位置的新版应用，检查并修复连接。
- `CONNECTION_FILE_EXPIRED`：窗口关闭、凭据更新或文件过期；重新导出，不反复重试旧文件。
- `INVALID_CONNECTION_FILE`：应使用连接文件而非.physlab.json实验文件；检查本机路径。
- `MULTIPLE_SESSIONS`：选择返回的sessionId，或使用目标窗口连接文件，不随机操作。
- `RECOVERY_PENDING`：应用有待处理草稿，用户需先选择恢复/丢弃，Agent不能覆盖。
- `CANCELLED`：连接被关闭，不自行重新启用。

应用每5秒检查发现登记，缺失时从当前服务内存重建；面板“检查并修复连接”还校验实际本机服务和工具入口。实际目录、登记状态和修复时间可在“不含凭据”的信息里查看。`--inspect`等调试参数不能证明程序正在使用测试目录。

Agent执行端必须能访问这台Windows电脑。支持看图但不能执行本地命令或连接本地MCP的产品不能操作软件；读图能力、模型服务和费用由用户自己的Agent提供。

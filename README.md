# PhysLab Experiments Skill

让支持图片理解和本地工具的 AI Agent，将文字题或带图物理题搭成 PhysLab 中可编辑、可运行、可保存的实验。

本仓库仅提供 **Skill、接口协议说明与旧版兼容脚本**，不包含 PhysLab 桌面应用、安装包或模型服务。使用前需通过作者提供的安装包安装 **Windows 桌面版 PhysLab 1.14.0 或更新版本**。仅下载此仓库不能启动实验台。

## 能做什么

- 从题目文字和图片提取给定条件、连接关系、待求量与必要的展示假设。
- 查询应用实际支持的器材，创建或修改同一实验台中的模型。
- 运行实验、读取真实测量值、对照解析结果并查看截图。
- 每道题保存为独立的 `.physlab.json`，以后直接在应用中打开、修改和重跑。

图片识别和模型调用由用户自己的 Agent 完成；PhysLab 不接收模型 API Key。应用及本地接口离线运行，Agent 使用何种模型及其费用由用户决定。纯网页聊天、无法访问本机 MCP 或命令工具的 Agent 不能直接控制软件。

## 快速开始

### 1. 准备应用

打开 PhysLab → 右上角 **Agent 连接** → **启用连接**。该面板有面向新手的分步操作说明。

### 2. 添加 MCP 工具（推荐）

1. 在 PhysLab 中选择“方式一：MCP”，点击“复制 MCP 配置”。
2. 打开 Agent 的设置 → 工具 / MCP → 添加本地 `stdio` 服务，粘贴配置并保存。**配置放在设置页面，不是聊天框。**
3. 刷新或重启 Agent 的工具服务，应看到 `physlab_connect`、`physlab_scene_apply` 等工具。

配置由每台电脑的应用生成，使用固定的 `PhysLabAgent.exe` 入口和应用自带运行时，不需要另外安装 Python 或 Node.js。不要套用别人的绝对路径；升级或移动应用后，打开新版并“检查并修复连接”，无需手改版本目录。

### 3. 安装 Skill

从 [Releases](https://github.com/ppk0115/physlab-experiments/releases/latest) 下载 Skill ZIP，解压得到 `physlab-experiments` 文件夹，导入所用 Agent 的技能库，再刷新技能或新开会话。也可以在应用中导出同一套 Skill。

Codex 默认可放在用户目录的 `.codex/skills/physlab-experiments`；自定义了 `CODEX_HOME` 时，放在其 `skills` 子目录。熟悉 Git 的用户可执行：

```powershell
git clone https://github.com/ppk0115/physlab-experiments.git "$env:USERPROFILE/.codex/skills/physlab-experiments"
```

目录已经存在时不要覆盖，先备份或按已有安装方式更新。其他 Agent 以自己的技能管理入口为准；支持 MCP 但没有技能库的产品，也可让 Agent 阅读本仓库 [SKILL.md](SKILL.md)。

### 4. 发题目

将下面的话发到 Agent 聊天框，并附上题目文字或图片：

> 使用 physlab-experiments，把这道物理题搭成可编辑实验。先连接并核对实验名称，保留题目给定条件，单独注明展示假设，运行核对关键量，按题目名称保存独立配置，返回文件路径、截图和检查结果。

Codex 中也可以用 `$physlab-experiments` 调用。Agent 应先确认实际连接成功，再搭建；题图看不清的关键条件应先核对，不能假称识别成功。

完成后得到实际应用中的模型、截图、测量结果和 `.physlab.json` 配置。以后在 PhysLab 点“打开”选择该配置即可使用，不需要再调用 AI。

## 没有 MCP 时

Agent 需要本机命令执行能力，可按 Skill 调用应用安装的固定入口：

```powershell
& (Join-Path ([Environment]::GetFolderPath('LocalApplicationData')) 'PhysLab/AgentBridge/PhysLabAgent.exe') connect --launch
```

固定入口不存在时，先打开新版 PhysLab，在“Agent 连接”检查修复，不要猜应用安装目录。仓库中的 `scripts/physlab.py` 仅作为旧版兼容客户端，需要 Python 3.10+，无需 pip；它不是新版 MCP 的必需依赖。

## 找不到窗口时

1. 在目标 PhysLab 窗口点击“检查并修复连接”。
2. 仍失败，点击“导出当前窗口连接文件”，将 `.physlab-connection.json` 的**本机完整路径**交给 Agent。
3. MCP 调用 `physlab_connect`，将路径作为 `connectionFile` 参数；CLI 使用 `--connection <文件路径>`。

连接文件用来指定窗口，实验配置用来保存题目，两者不同。连接文件包含会话凭据，不要上传到公开仓库；关闭/重开连接或重启应用后应重新导出。文件连接失败不会自动回退到另一个窗口。完整说明见 [连接与恢复](references/connection.md)。

## 模型范围与版本

基于 PhysLab 1.14.0、操作协议 1、场景格式 13。当前为 3D 展示、XY 平面物理；球和普通滑块无自转，电路为直流稳态，薄透镜为近轴模型，液面固定，弦波为一维小振幅，热学为集中参数。Agent 应查询实际器材能力，不把超出模型范围的题目宣称为正确模拟。

对应应用版本已验证本地 CLI/MCP、文字题与带图题、真实测量、逐题保存和连接文件恢复；不表示已逐一验证所有第三方 Agent 产品。

## 仓库内容

| 文件 | 用途 |
|---|---|
| [SKILL.md](SKILL.md) | Agent 的读题、建模、检查与保存流程 |
| [references/connection.md](references/connection.md) | 安装、连接及故障恢复 |
| [references/modeling.md](references/modeling.md) | 简洁实验格式与关系建模 |
| [references/operations.md](references/operations.md) | 调用、测量、重试与错误处理 |
| assets/protocol.json | 版本化操作协议与参数结构 |
| scripts/physlab.py | Python 3.10+ 旧版兼容 CLI/MCP 客户端 |
| agents/openai.yaml | Codex 技能展示元数据 |


## 1.14.0 新增能力

支持固定墙壁、保留倾角的教材理想轨道接头，以及 `analysis.query` / `analysis.clear` 独立精确定位。反查高度时可选择重复时刻，未来状态明确标记为预测。读数留在分析面板，场景显示独立的虚拟位置，不改变主实验。

操作协议仍为1；以应用实际返回的能力声明为准。新实验格式为13，旧文件仅在主动保存后升级。旧题目依据因连接语义变化而过期时，应重新核对计算结果。

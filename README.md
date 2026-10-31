# PhysLab Experiments Skill

让支持图片理解和本地工具的 AI Agent，将文字题或带图物理题搭成 PhysLab 中可编辑、可运行、可保存的实验。

本仓库仅提供 **Skill、接口协议说明与旧版兼容脚本**，不包含 PhysLab 桌面应用、安装包或模型服务。使用前需获取并运行与本仓库匹配的 **PhysLab 2.0.0-dev.6 Windows 教材模型开发包**，保留同目录全部资源。仅下载此仓库不能启动实验台。

> 当前主分支为教材2.0开发版Skill。完整2.0尚未正式发布，必须通过应用实时能力与 `model.analyze` 检查，不能把所有器材组合视为已验收。使用旧版PhysLab 1.14.x时可选[旧版Skill标签v1.14.0](https://github.com/ppk0115/physlab-experiments/tree/v1.14.0)。

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

推荐在对应开发版应用中导出当前 Skill ZIP，解压后导入 Agent 的技能库。也可点击仓库 **Code → Download ZIP**，解压后将包含 `SKILL.md` 的文件夹重命名为 `physlab-experiments` 再导入；刷新技能或新开会话后生效。历史 [Releases](https://github.com/ppk0115/physlab-experiments/releases) 中的1.14.0包不包含教材2.0新增能力。

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

对应 **PhysLab 2.0.0-dev.6、操作协议1、场景格式14、简洁建模规格v2（兼容读取v1）**。首步读取应用能力声明，教材模型要求 `textbookPointModels`、`formalTrigonometry` 和 `modelAnalysis`；读取加速度还需 `accelerationReadings`。旧应用缺少能力时应明确提示使用对应新版，不发送不存在的命令，也不把旧有限尺寸模型作为质点题完成。

- 普通小球、滑块、砝码默认按真正质点计算；外观大小不决定位置、距离或碰撞时刻，中心标记为物理位置。
- 题给37°与sin=0.6、cos=0.8同时保存，正式参数进入求解；不反算覆盖角度，不移动墙面或缩小外观补偿结果。
- 已验证的范围包括自由运动、直轨摩擦/无损接头/墙面、同轨碰撞、单圆轨、单细线/轻杆、同轴弹簧/阻尼和自由二维弹簧的支持组合。具体条件以 `model.analyze` 的实际报告为准。
- 多约束、圆弧接头、滑轮杠杆、板块传送带及其他领域仍有未完成的教材2.0门禁；不能因为器材在目录中就宣称任意组合可用。
- 加速度大小与X/Y分量来自同一求解状态，单位m/s²；事件时刻区分前后单侧连续加速度，瞬时冲量不换算成有限加速度。

应用开发版已通过相关定向物理、历史/精确查询、CLI、保存重开和Windows开发包断网验证；尚未完成全部组合和所有第三方Agent产品的验收。

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


## 教材2.0新增工作流

题文/题图 → 正式物理参数和理想化约定 → `model.analyze` → `scene.validate` → `scene.apply` → 实际运行/测量/独立查询 → 单独保存配置。

`analysis.query` 与 `analysis.clear` 支持独立定位；右侧高度可编辑，历史状态在场景中显示分身。`acceleration`、`ax`、`ay` 可用于实际测量、历史读取与精确时间查询。所有能力以实时manifest为准，本仓库的协议JSON仅描述对应开发版本。

当前主分支和应用随包Skill的七个载荷文件同步；只发布Skill相关文件，不包含应用代码、用户题图、实验输出、会话文件或token。

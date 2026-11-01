# 调用与结果

CLI优先用固定入口 `PhysLabAgent.exe <协议命令> --input <参数文件>`；兼容脚本为 `python <技能目录>/scripts/physlab.py`。完整定位方法见connection.md。显式窗口连接用 `--connection <导出文件>`；MCP先调用physlab_connect并传connectionFile，后续工具复用目标。CLI输出JSON，退出码0表示ok:true；MCP工具名为physlab_加协议名中的点改下划线。

常用顺序：connect → catalog.list / catalog.describe → model.analyze → scene.validate → scene.apply → view.set → artifact.screenshot 并查看 → simulation.step / measurements.get → reset 或暂停 → artifact.save。

- model.analyze 使用 {"spec":规格,"mode":"replace"} 分析拟建模型，省略spec分析当前场景；scene.validate返回同一 modelReport。solvable:false 时按 issues 的器材和原因报告，不能创建其他模型后声称完成。
- validate / apply 参数是 {"spec":简洁规格,"mode":"replace"}；patch 用于已有场景。确认 applied:true 和 recoveryId，不能只看到请求发送就宣称完成。
- view.set 支持 camera:fit/front/perspective、zoom、labels、vectors、trajectory、chart。zoom 为绝对相机缩放值，范围 0.05–50，值越小视野越广；可从返回的 camera.zoom 调低以给速度箭头留边距。selection.set 用 objects:[ID] 选择观察对象。
- simulation.step 的 seconds 量化到最近 1/240 s，最少一步，每次最多 30 s；以实际 advancedSeconds 与 state.time 为准。run 连续播放，pause 暂停，reset 恢复编辑。
- measurements.get {} 读取全部实际状态；object 指定对象或 system。history:true 默认返回最后 limit 条原始记录（最多3000），不插值。同一事件的前后记录不会被分页拆开，因此特别小的 limit 可能返回整个同时间组。要读取较早或完整高频记录，使用 order:"asc",since:0，随后将返回的 nextAfter 作为 after 继续分页，直到 hasMore:false；until 限定时间上界。windowStart/windowEnd 表示当前保留窗口，分页期间需留意旧记录过期。source:initial 是初始条件，不能当作运行后结果。静态电路、光学也能直接测量。
- scene.get 的 studyStale 表示当前物理条件已与题目依据不一致。修改题目参数后重新核对推导和参考点，并通过 patch 同时更新 notes；应用不会自动解题或移动参考点。场景格式12新增依据签名，旧文件内存迁移后仍可编辑。
- artifact.save 保存初始场景和题目依据；截图返回 PNG。文件均在应用专属目录，返回真实路径、大小和 SHA-256。name 只允许中文、字母、数字、空格、下划线、连字符，不能传任意系统路径。MCP 截图会附图像内容。

## 重试和恢复

重要 CLI 修改可传 --request-id UUID；MCP 工具可传 requestId 和 expectedRevision 元数据。重试使用同一 ID 与完全相同的参数，避免重复搭建或推进；改变内容重用 ID 会得到 REQUEST_ID_CONFLICT。缓存不是跨应用重启的永久保证。

status / scene.get 返回 revision；CLI 可传 --expected-revision，防止覆盖期间发生的用户修改。REVISION_CONFLICT 时重新读取，不能强行覆盖。超时不证明未执行，先查状态或重试原 ID。

apply 校验成功后才保存恢复快照并提交。recovery.list 列出快照，recovery.restore {id:返回的UUID} 恢复旧初始场景；恢复前也备份当前场景，不直接改快照文件。

## 错误

INVALID_SPEC / INVALID_ARGUMENTS 看 details 字段路径；UNKNOWN_PARAMETER / UNSUPPORTED_EQUIPMENT 重新查目录；UNIT_MISMATCH 修正单位；MISSING_OBJECT / AMBIGUOUS_OBJECT 修正引用；INVALID_PLACEMENT 看重叠、嵌入、绳长和挂点；INVALID_CIRCUIT 看拓扑；SIMULATION_REJECTED 看物理参数范围；EDIT_LOCKED 先 reset；APP_NOT_READY / RENDER_TIMEOUT 检查界面和 3D；CANCELLED 表示用户关闭连接，不自行重新启用。

同一错误连续两次修正仍失败，应检查模型理解和器材说明，而非无界重试或降低题意要求。应用没有任意代码执行、shell 或无限制文件操作接口。


## 独立精确定位（analysis.query / analysis.clear）

协议仍为1；先检查应用协议清单的 `capabilities.independentAnalysis`，旧版不支持时请升级到1.14.0以上。教材模型版实验格式为14，旧文件只在保存时写入新版本；轨道语义改变后，旧题目依据会标为过期，需要重新核对。

- `analysis.query {"object":"ball","channel":"y","mode":"time","time":1.001234,"phase":"after"}`：独立回放初始场景和已记录开关操作，返回真实状态、时间、来源及误差；不改变主实验时钟、物体位置或曲线。
- `mode:"value", value:-5, start:0, end:10`：按物理量反查，返回最多100个候选，含上升/下降、极值或持续满足的时间区间。默认范围0到max(10s,当前时间)，超限需缩小范围。
- `mode:"both", time:1, value:-5`：校验同一状态；条件不符返回 `consistent:false`、实际值和差值，不生成匹配分身。
- 返回 `sample` 中所有量来自同一状态；`source:"prediction"` 为尚未发生的未来预测，后续操作可能改变结果。`interpolated:false` 表示来自同一内核的独立计算，不是对绘图点插值。时间求值目标1μs、高度残差目标10μm，只表示数值定位精度。
- `analysis.clear {}` 取消计算并清除定位读数/分身，不重置实验。新查询、切换对象或模型变化会取消旧计算；单次超过30秒返回超时提示。
- 教材事件记录保留精确时间及 before/after 状态；在事件时刻用 phase:"before" / "after" 区分撞墙前后，不能对速度跳变插值。system.frictionHeat 是摩擦热量，collisionLoss 是非弹性碰撞损失；frictionSegments 按过程分段，不能混用。
- 主实验仍采用固定1/240s步长，独立求值最后可用短步抵达目标时间；碰撞跳变处找不到的中间值不能报告成解。

教材开发版新增连续加速度通道：运动物体的 `acceleration` 为加速度大小，`ax` / `ay` 为分量，单位均为 `m/s²`。初始状态、运行、历史和 `analysis.query` 共用求解方程；先检查 `capabilities.accelerationReadings`。事件时刻用 phase 区分前后单侧加速度，碰撞和绳子收紧的瞬时冲量不能换算为有限瞬时加速度。不要用相邻绘图采样速度之差计算它，也不要把速率不变误认为加速度为零。

## 教材电路布局与几何检查

先检查 `capabilities.textbookCircuitLayout` / `circuitGeometryInspection`，旧应用没有这些能力时不能调用以下命令。

- `circuit.layout {}`：编辑状态下，按明确端子关系整理单电源串联、并联和可归约混联；复杂网络返回通用布局提示。可撤销，保留电气参数和开合状态。
- `circuit.inspect {}`：只读，返回 `ok`、`issues`、`metrics`、`screenshotReviewRequired:true`，不改变实验或主时钟。检查器材重叠、导线穿器材、不同节点导线重合，以及满足端子引出要求的可直接减少折弯路径。
- 结构/物理校验、几何检查和截图复核分别进行；几何通过不能替代电路求解，不能替代标签和题图含义的人工视觉检查。

## 启动诊断与编辑约束

`validationHighlights` 表示启动失败会保存具体 `issues[].objects` 并在三维场景显示红框；界面「定位问题」聚焦这些器材。模型分析和校验只读返回拟建规格的问题，不会在当前无关实验上画红框。`simulation.run` 失败的 `details.issues` 可用于Agent定位，按实际协议版本/能力读取。

`constraintEditingLimits` 表示界面拖动和坐标输入限制细线端距，允许松弛，不修改保存的线长；过短的新线长会拒绝且显示实际保留值。正式Agent输入仍需按题设通过校验，不会被界面自动修正。`boxSelection` 为界面整体移动/删除能力；无新增框选命令，不要编造工具。

物体不动时先检查场景重力、初速度、受力和时间。纯电路无需设置g=0；当前实验已明确g=0时不能因预期下落而擅改题设，需说明原因。自由落体不要求固定点；只有所选约束模型的缺失/冲突才报错。

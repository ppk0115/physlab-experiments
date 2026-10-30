# 调用与结果

CLI优先用固定入口 `PhysLabAgent.exe <协议命令> --input <参数文件>`；兼容脚本为 `python <技能目录>/scripts/physlab.py`。完整定位方法见connection.md。显式窗口连接用 `--connection <导出文件>`；MCP先调用physlab_connect并传connectionFile，后续工具复用目标。CLI输出JSON，退出码0表示ok:true；MCP工具名为physlab_加协议名中的点改下划线。

常用顺序：connect → catalog.list / catalog.describe → scene.validate → scene.apply → view.set → artifact.screenshot 并查看 → simulation.step / measurements.get → reset 或暂停 → artifact.save。

- validate / apply 参数是 {"spec":简洁规格,"mode":"replace"}；patch 用于已有场景。确认 applied:true 和 recoveryId，不能只看到请求发送就宣称完成。
- view.set 支持 camera:fit/front/perspective、zoom、labels、vectors、trajectory、chart。zoom 为绝对相机缩放值，范围 0.05–50，值越小视野越广；可从返回的 camera.zoom 调低以给速度箭头留边距。selection.set 用 objects:[ID] 选择观察对象。
- simulation.step 的 seconds 量化到最近 1/240 s，最少一步，每次最多 30 s；以实际 advancedSeconds 与 state.time 为准。run 连续播放，pause 暂停，reset 恢复编辑。
- measurements.get {} 读取全部实际状态；object 指定对象或 system。history:true 默认返回最后 limit 条原始记录（最多3000），不插值。要读取较早或完整高频记录，使用 order:"asc",since:0，随后将返回的 nextAfter 作为 after 继续分页，直到 hasMore:false；until 限定时间上界。windowStart/windowEnd 表示当前保留窗口，分页期间需留意旧记录过期。source:initial 是初始条件，不能当作运行后结果。静态电路、光学也能直接测量。
- scene.get 的 studyStale 表示当前物理条件已与题目依据不一致。修改题目参数后重新核对推导和参考点，并通过 patch 同时更新 notes；应用不会自动解题或移动参考点。场景格式12新增依据签名，旧文件内存迁移后仍可编辑。
- artifact.save 保存初始场景和题目依据；截图返回 PNG。文件均在应用专属目录，返回真实路径、大小和 SHA-256。name 只允许中文、字母、数字、空格、下划线、连字符，不能传任意系统路径。MCP 截图会附图像内容。

## 重试和恢复

重要 CLI 修改可传 --request-id UUID；MCP 工具可传 requestId 和 expectedRevision 元数据。重试使用同一 ID 与完全相同的参数，避免重复搭建或推进；改变内容重用 ID 会得到 REQUEST_ID_CONFLICT。缓存不是跨应用重启的永久保证。

status / scene.get 返回 revision；CLI 可传 --expected-revision，防止覆盖期间发生的用户修改。REVISION_CONFLICT 时重新读取，不能强行覆盖。超时不证明未执行，先查状态或重试原 ID。

apply 校验成功后才保存恢复快照并提交。recovery.list 列出快照，recovery.restore {id:返回的UUID} 恢复旧初始场景；恢复前也备份当前场景，不直接改快照文件。

## 错误

INVALID_SPEC / INVALID_ARGUMENTS 看 details 字段路径；UNKNOWN_PARAMETER / UNSUPPORTED_EQUIPMENT 重新查目录；UNIT_MISMATCH 修正单位；MISSING_OBJECT / AMBIGUOUS_OBJECT 修正引用；INVALID_PLACEMENT 看重叠、嵌入、绳长和挂点；INVALID_CIRCUIT 看拓扑；SIMULATION_REJECTED 看物理参数范围；EDIT_LOCKED 先 reset；APP_NOT_READY / RENDER_TIMEOUT 检查界面和 3D；CANCELLED 表示用户关闭连接，不自行重新启用。

同一错误连续两次修正仍失败，应检查模型理解和器材说明，而非无界重试或降低题意要求。应用没有任意代码执行、shell 或无限制文件操作接口。

# 简洁实验格式 v2（协议1，场景14）

先检查能力清单。应用保留读取v1，但教材质点语义以 `model.analyze` 报告为准；应用尚未支持的组合不能靠旧引擎替代。题目说明 notes 不参与求解。

先查 `catalog.list`，再按需查 `catalog.describe {"types":["incline","block"]}`。参数以返回的名称、范围和单位为准。数字默认 SI，角度默认度；支持 `"200 g"`、`"30 cm"`、`"36 km/h"`、`"20 μC"`、`"0.5 T"` 等兼容单位字符串。

```json
{
 "version":2,"title":"斜面题目实验","gravity":10,
 "objects":[
  {"id":"斜面","type":"incline","at":[0,3],"angle":-30,"params":{"length":4,"friction":0}},
  {"id":"A","type":"block","params":{"mass":"500 g","width":0.4,"height":0.3},"placeOn":{"track":"斜面","along":{"fraction":0.2}}}
 ],
 "notes":{"goal":"检查指定时间后的速度","known":["斜面 30°，光滑，质量 0.5 kg，g=10"],"derived":[],"assumptions":["滑块宽高仅用于展示，质点位于轨道上；轨道长度是正式物理条件"],"unknowns":["速度"]}
}
```

不要照抄示例参数代替题目条件。默认重力 9.8，题目给了 g=10 时必须设置 10。notes 可包含 sourceText、source 和 goal，以及 known、derived、assumptions、unknowns 四类文字列表，随实验保存。

## 位置与运动

- id 用于稳定引用，name 可选，默认等于 id。引用优先用 ID，唯一名称也能解析；重名报错。
- type 来自器材目录；params 仅放目录中允许的普通参数。
- at:[x,y] 单位 m。angle 逆时针为正；直线轨道位置是起点，圆弧是圆心，液体位置是液面中心。
- velocity:[vx,vy] 或 {speed,direction}；向右 0°、向上 90°。负速度大小不合法，通过方向表达。
- placeOn:{track,along,side,align} 将质点参考点放到轨道上，不添加半径、半高或间隙。along 是正式物理距离或 {fraction:0..1}，默认中点。side 默认 left，滑块可对齐展示姿态；外观尺寸不决定轨道位置。质点不能设置非零 gap；自由释放时直接设置 at。
- 运动对象的 mechanics.model 默认为 point；明确涉及尺寸或转动时使用 finite-translation / rigid，并在 mechanics.physicalSize 保存必要物理尺寸。当前支持范围须由 model.analyze 确认。
- 对象 angle 保留原始角度；trigonometry:{sin,cos,source,raw?} 指定该方向的题给三角值，例如 angle:-37, trigonometry:{sin:-0.6,cos:0.8,source:"题给 sin37°=0.6、cos37°=0.8，轨道向右下降"}。速度方向也可在 velocity:{speed,direction,trigonometry} 中单独指定。不能反算并覆盖题设角度。
- textbook.constants 可设置 pi、coulomb、source。没有题给值时使用应用默认值；模型分析会指出尚未支持的使用方式。

## 机械连接与轨道

```json
{"id":"弹簧","type":"spring","a":{"world":[0,1]},"b":{"object":"A","local":[0,0]},"params":{"stiffness":"40 N/m","restLength":0.8}}
```

端点 a/b 可简写为对象 ID（中心），也可给局部挂点或世界坐标。新 rope/rod 未给 length 时取实际端距；弹簧原长不会自动等于端距。铰链 b 简写为对象 ID 时，应用计算与 a 重合的局部挂点，不移动杆件。

滑轮 mount 采用相同挂点格式。新增动滑轮只创建滑轮，不自动添加砝码、载荷或质量；未安装草稿可摆放和保存，完成运行规格时必须明确安装到题目中的已有物体，轮本身无质量。scene 中 mount=null 表示未安装，位置采用 transform；已安装后的世界位置由 mount 计算，不能把局部偏移或陈旧的 transform 当成世界坐标。界面分别显示滑轮中心场景坐标与安装偏移。routed-rope 使用 a/b 与有序 via:[{pulley:"P",side:-1}]；新绳未给 length 时采用真实切线和包角总长。交叉不建立连接。

轨道使用 joins:[{from:["坡道",1],to:["水平段",0],align:false}]。0 是起点，1 是终点；应用移动 to 轨道使端点重合。align:true 还会对齐切线；题目有折角时用 false。先拼接再计算 placeOn。

## 电路

```json
{
 "version":1,"title":"串联测量",
 "objects":[
  {"id":"E","type":"battery","params":{"voltage":9}},
  {"id":"S","type":"switch","params":{"closed":true}},
  {"id":"A","type":"ammeter"},
  {"id":"R","type":"resistor","params":{"resistance":"30 Ω"}},
  {"id":"V","type":"voltmeter"}
 ],
 "circuits":[{"series":["E","S","A","R"],"close":true}],
 "wires":[["V.a","R.a"],["V.b","R.b"]]
}
```

series 自动生成导线：电源 a（正端）向外供电，普通元件按 a→b 接入。反接电源或特殊分支用明确的 wires 端子对。每个器件有 a/b 两端；电压为 Va−Vb，电流正值为 a→b，电源供电时自身电流可为负，不能把负数误判为失败。

纯电路新实验默认按接线结构整理布局（详细规则见[circuit-layout.md](circuit-layout.md)）；layout:"preserve" 保留位置，layout:"circuit" 显式整理。交叉不接通，必须共用端子引用。通过 series 加入电流表，通过 wires 将电压表并联到待测两端。

## 其他领域

- 电场、磁场与带电物体使用目录参数；charge 单位 C，正磁场出屏，重力仍按实验 gravity 作用。
- 光学设置位置、角度、focalLength、aperture 等；是否成像或命中光屏以追迹读数为准。
- 浮力需正确的密度和实际体积。液体 at.y 为液面；两个重叠液体会拒绝。
- wave-source / wave-probe 用 string:"绳 ID"，params.position 为沿绳距离。频率幅度遵守当前小振幅模型。
- heater / thermometer 用 target:"材料 ID"；thermal-link 用 a:"材料 A"、b:"材料 B"。几何接触不等于传热。

## 修改

scene.apply 的 mode:"patch" 以当前初始场景为基础；已有 ID 可省略 type，仅改变给出的字段。remove:["A"] 级联移除依赖连接器。失败完全不应用。运行中先 reset 再修改结构。移动器材不改变连接器原长，题目修改初始条件时应一并明确。


## 教材连接与墙壁（教材模型版 / 场景14）

- 轨道 `joins` 默认 `align:false, model:"textbook"`。端点重合，保留题目给出的倾角；只有确需切线对齐才传 `align:true`。高级 `model:"geometric"` 保留真实硬折角冲击。交叉不是连接。
- 教材过渡只在实际连接局部、已有轨道支撑时接管运动方向，不吸附飞来的物体，不阻止凸形开放轨道离轨。斜面接平面可直接建模，不必偷偷加圆弧或改倾角。
- 普通物体按质点计算，球心/物块中心为物理参考点。显示半径、宽高不改变轨道半径、运动距离、碰撞时刻或能量；圆轨等组合需先通过 model.analyze。有限尺寸与转动必须正式选择对应模型，不使用小尺寸近似质点。
- `type:"wall"` 为独立固定墙，`at` 是物理墙面中点，质点到达该面时碰撞；厚度仅展示，`angle:90` 为竖直。默认 `length:3, thickness:0.15, depth:0.4, friction:0, restitution:1`。墙面两侧均碰撞，恢复系数独立于 `world.trackRestitution`；墙不是轨道接头。
- 理想连接不额外增加质量、阻尼或储能；仍保留题设摩擦、弹簧阻尼以及松绳突然拉紧的非弹性约束冲量。冲突约束按错误处理，不用补回能量掩盖问题。

```json
{"version":1,"gravity":0,"objects":[{"id":"ball","type":"ball","at":[-2,0],"velocity":[4,0]},{"id":"wall","type":"wall","at":[0,0],"angle":90,"params":{"length":4,"restitution":1,"friction":0}}]}
```

## 细线编辑与错误定位

界面会限制拖动和数值定位产生的细线端距，但不更改保存的线长；同组两端一同移动时保留内部距离，外接细线仍限制整个组的移动。弹簧不受此限制。通过Agent给定的非法初态仍需按原始条件拒绝，不自动移动题目物体。

`model.analyze` / `scene.validate` 的问题包含具体器材ID；缺失质点连接、超线长及不支持组合可分别定位。启动后物体静止时先检查实际g和受力；g=0不是引擎停止。没有固定端本身不构成自由落体错误。

细线器材参数 `length` 是不变的设定线长；运行/历史测量中的 `length` 是两端距离，`slack` 是余长。界面分别标明，不应把端距增加误报为线被拉伸。

## 小球碰撞与能量

先检查 `pointHeadOnCollisions` / `finiteSphereCollisions` / `collisionEnergyReadings`。普通运动物体默认教材质点，`textbook.pointCollisions` 可设 `head-on`（默认）或 `disabled`；相遇时对心碰撞的轴为碰前相对运动方向，同轨时沿轨道。显示半径不决定相遇时间，球面视觉相交不表示质点已碰撞。不能为了球面接触擅自改变题目采用的质点模型。

题设需要半径或斜碰接触几何时，球体显式设置 `mechanics:{model:"finite-translation",physicalSize:{radius:0.25}}`。`params.radius` 仍为展示半径，二者独立。当前有限球体支持自由运动、光滑球间及光滑墙面碰撞，不计自转；轨道、弹簧、绳杆等耦合及同时多体碰撞仍需 `model.analyze`，不能宣称全部支持。

`world.ballRestitution` 是物体间恢复系数，墙使用独立 restitution。检查实际事件的 before/after、collision.normal、impulse、energyLoss 和系统 collisionLoss；e=1 时验证总动能，e<1 时验证“动能减少＝碰撞耗散”，同时核对动量。单个物体能获得或失去动能，不能把它的动能变化误报为整个系统耗散。碰撞耗散与 frictionHeat 分开，瞬时冲量不能除以显示帧时长冒充瞬时力。

## 连续组合力学

`coupledMechanics` 表示应用具备组合事件求解路径，仍需分析整个场景。可将多个质点、直轨/圆弧网络、光滑墙、中心胡克弹簧/阻尼器、测力计，以及互不共享运动端的细线/轻杆单元放在同一实验；细线/轻杆单元可以是固定端＋质点，或两个自由运动质点。允许与其他物体碰撞，绳子受压必须松弛，轻杆可以传递压缩冲量。相同质点同时受到多根定长约束，或同时需要轨道和绳杆的独立约束，当前需明确报告，不按单约束强行计算。

- 开放直轨与圆弧可以自由飞出、再接触；相交且未连接的轨道不构成接头。`joins` 表达连续轨道连接，保留题设角度；默认教材模型只在实际接头交接，不额外耗能，凸形单边轨道可自然离轨。`model:"geometric"` 采用目标轨道法向碰撞与 `world.trackRestitution`，碰撞耗散单列。
- 自由飞行相切进入圆弧，需同时满足“位置在实际入口、速度沿向内行进的切线、支撑力可实现”；不能只把球丢到圆轨最低点后改方向。非相切接触可能损失能量。改变速度/半径后要重新核对衔接条件，不能沿用原来的无损结论。
- 弹簧对真实两端施力、储能只统计一次，允许跨教材接头；碰墙/碰球后继续沿同一时钟演化。固定端/双运动端细线的收紧损失统计为 `constraintLoss`，与 `collisionLoss`、`frictionHeat`、`dampingLoss` 分开。
- 完全非弹性碰撞后，共线物体可在外力作用下继续压紧，以连续接触力共同运动；需要拉力才可维持时会分离，不能把恢复系数0误认为永久粘合。涉及第三个持续接触物体或非共线多重约束时仍需联立求解，按实际报告说明。
- 有轨道或定长约束参与的碰撞，物体动量可能因固定支撑冲量而变化；按完整模型的冲量/能量关系核对，不对所有组合强套“两个物体总动量守恒”。
- 示例可采用“两球各1 kg，A以5 m/s撞静止B，平台边缘(0,4)，g=10；B飞行0.5 s到(2.5,2.75)，速度(5,-5)；半径1 m圆弧圆心(2.5+√2/2,2.75+√2/2)，起始极角225°、圆心角225°”。整个过程必须保留A和B，同一场景核对碰撞0.2 s、飞出0.3 s、入轨0.8 s及系统机械能92.5 J。这里的数值仅为演示例子，不可覆盖用户题设。

有限尺寸球体、滑轮绳路、杠杆铰链及多重共享约束仍以本机分析报告为准；组合能力不代表任意器材组合均已交付。

# 简洁实验格式 v1

先查 `catalog.list`，再按需查 `catalog.describe {"types":["incline","block"]}`。参数以返回的名称、范围和单位为准。数字默认 SI，角度默认度；支持 `"200 g"`、`"30 cm"`、`"36 km/h"`、`"20 μC"`、`"0.5 T"` 等兼容单位字符串。

```json
{
 "version":1,"title":"斜面题目实验","gravity":10,
 "objects":[
  {"id":"斜面","type":"incline","at":[0,3],"angle":-30,"params":{"length":4,"friction":0}},
  {"id":"A","type":"block","params":{"mass":"500 g","width":0.4,"height":0.3},"placeOn":{"track":"斜面","along":{"fraction":0.2}}}
 ],
 "notes":{"goal":"检查指定时间后的速度","known":["斜面 30°，光滑，质量 0.5 kg，g=10"],"derived":[],"assumptions":["轨道长度与滑块尺寸用于展示，只检查到达末端前的运动"],"unknowns":["速度"]}
}
```

不要照抄示例参数代替题目条件。默认重力 9.8，题目给了 g=10 时必须设置 10。notes 可包含 sourceText、source 和 goal，以及 known、derived、assumptions、unknowns 四类文字列表，随实验保存。

## 位置与运动

- id 用于稳定引用，name 可选，默认等于 id。引用优先用 ID，唯一名称也能解析；重名报错。
- type 来自器材目录；params 仅放目录中允许的普通参数。
- at:[x,y] 单位 m。angle 逆时针为正；直线轨道位置是起点，圆弧是圆心，液体位置是液面中心。
- velocity:[vx,vy] 或 {speed,direction}；向右 0°、向上 90°。负速度大小不合法，通过方向表达。
- placeOn:{track,along,side,align,gap} 自动计算球或滑块的表面接触位置。along 为沿轨道表面的距离或 {fraction:0..1}，默认中点。side 默认 left（沿轨道正向的左侧），right 为另一侧；滑块默认 align:true 对齐切线。gap 默认 0.001 m。碰撞使用真实外形，不能忽略半径和尺寸。

## 机械连接与轨道

```json
{"id":"弹簧","type":"spring","a":{"world":[0,1]},"b":{"object":"A","local":[0,0]},"params":{"stiffness":"40 N/m","restLength":0.8}}
```

端点 a/b 可简写为对象 ID（中心），也可给局部挂点或世界坐标。新 rope/rod 未给 length 时取实际端距；弹簧原长不会自动等于端距。铰链 b 简写为对象 ID 时，应用计算与 a 重合的局部挂点，不移动杆件。

滑轮 mount 采用相同挂点格式。动滑轮必须明确安装到有质量物体，轮本身无质量。routed-rope 使用 a/b 与有序 via:[{pulley:"P",side:-1}]；新绳未给 length 时采用真实切线和包角总长。交叉不建立连接。

轨道使用 joins:[{from:["坡道",1],to:["水平段",0],align:false}]。0 是起点，1 是终点；应用移动 to 轨道使端点重合。align:true 还会对齐切线；题目有折角时用 false。先拼接再计算 placeOn。

## 电路

```json
{
 "version":1,"title":"串联测量","gravity":0,
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

纯电路新实验默认整理布局；layout:"preserve" 保留位置，layout:"circuit" 显式整理。交叉不接通，必须共用端子引用。通过 series 加入电流表，通过 wires 将电压表并联到待测两端。

## 其他领域

- 电场、磁场与带电物体使用目录参数；charge 单位 C，正磁场出屏，重力仍按实验 gravity 作用。
- 光学设置位置、角度、focalLength、aperture 等；是否成像或命中光屏以追迹读数为准。
- 浮力需正确的密度和实际体积。液体 at.y 为液面；两个重叠液体会拒绝。
- wave-source / wave-probe 用 string:"绳 ID"，params.position 为沿绳距离。频率幅度遵守当前小振幅模型。
- heater / thermometer 用 target:"材料 ID"；thermal-link 用 a:"材料 A"、b:"材料 B"。几何接触不等于传热。

## 修改

scene.apply 的 mode:"patch" 以当前初始场景为基础；已有 ID 可省略 type，仅改变给出的字段。remove:["A"] 级联移除依赖连接器。失败完全不应用。运行中先 reset 再修改结构。移动器材不改变连接器原长，题目修改初始条件时应一并明确。

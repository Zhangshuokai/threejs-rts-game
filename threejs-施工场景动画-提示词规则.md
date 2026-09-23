# Three.js 施工场景动画 —— 提示词（Prompt）规则手册

> 目标：把“能完整描述、并让 AI 生成施工场景动画”的提示词拆成可复用、可校验的规则，
> 使生成结果具备**正确的施工工序、合理的设备运动、可运行的 three.js 工程**。
>
> 本手册的规则来自以下**真实可核验的来源**（详见第 12 节）：
> 1. three.js 官方示例 `webgl_animation_keyframes`（动画系统、渲染管线写法）
> 2. `Mishi17o/ArvixAI-3D-Gen`：自然语言 → three.js 场景的**系统提示词**与固定脚手架
> 3. `img2threejs/img2threejs`：代码化生成 three.js 模型的**分阶段/门禁规则**（几何、材质、动画、自纠）
> 4. `caimingye78/construction-sim`（Tower Construct）：three.js + Rapier 的**施工模拟**实现（塔吊、工序、验收）

---

## 1. 提示词的整体结构（8 段式）

一条高质量的“施工场景动画”提示词应显式包含以下 8 段，缺一段就要在生成结果里“猜”，必然失真：

| 段 | 内容 | 缺失后果 |
|---|---|---|
| A 目标 | 要什么（可交互施工模拟/宣传动画/进度演示）、时长、循环与否 | 做成静态模型或风格跑偏 |
| B 场地 | 坐标范围、地面、道路、围挡、大门、堆场、临时设施 | 构件“悬空”，无场地感 |
| C 对象与构件 | 建筑类型、层数、构件清单与尺寸/材质 | 构件乱造、比例失衡 |
| D 工艺流程 | 施工顺序（基础→主体→围护→封顶）、工序依赖 | 顺序错误、逻辑不成立 |
| E 动画与时间轴 | 设备运动、状态机、释放条件、时长与缓动 | 有模型无动画，或动画穿模 |
| F 相机与交互 | 预设机位、轨道控制、聚焦/剖切/点击 | 只有单一视角，无法查看 |
| G 视觉与光照 | 天空/雾、太阳+阴影、色调映射、材质配色 | 灰暗、无体积感 |
| H 工程与验收 | three 版本、渲染器、性能预算、自检项、交付物 | 不可运行/帧率崩/无法验收 |

**写法建议**：用“结构化清单 + 明确数值 + 明确禁止项”，不要用“好看/流畅/逼真”这类词
（img2threejs 明确要求使用 3D 术语，禁止 `nice/smooth/shiny`）。

---

## 2. 通用 three.js 场景规则（跨项目复用）

> 来自 ArvixAI-3D-Gen 的系统提示词 `src/prompts.py`，其原文规则经实测可稳定出图：

- **固定脚手架**：预先定义好 `scene`、`camera`、`renderer`、`controls`，AI 只负责“往场景里加东西”。
- **必须有地面**：用一个大 `PlaneGeometry` 做地板/场地。
- **材质**：使用 `MeshStandardMaterial` 或 `MeshPhysicalMaterial`，颜色要有区分度。
- **尺度**：单个物体缩放控制在 `0.5 ~ 5.0` 单位；物体都放在原点附近。
- **阴影**：每个 mesh 都要 `castShadow = true`（地面 `receiveShadow = true`）。
- **输出格式**：**只输出裸 JavaScript**，不要 Markdown 代码块、不要注释、不要解释。
  （Arvix 在调用侧还会 `replace("```javascript","")` 兜底，但提示词里应直接禁止。）

补充的现代写法（来自 three.js 官方示例 `webgl_animation_keyframes`，避免旧版 API 坑）：

```js
renderer.setPixelRatio(window.devicePixelRatio);      // 建议再 Math.min(..., 2)
renderer.toneMapping = THREE.ACESFilmicToneMapping;
renderer.setAnimationLoop(animate);                   // 而不是裸 requestAnimationFrame

const timer = new THREE.Timer(); timer.connect(document);
function animate() {
  timer.update();
  const delta = timer.getDelta();                     // 动画一律用 delta 驱动
  mixer.update(delta);
  controls.update();
  renderer.render(scene, camera);
}

window.onresize = () => {
  camera.aspect = innerWidth / innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(innerWidth, innerHeight);
};
```

**渲染器回退规则**（来自 construction-sim `main.js`）：优先 `WebGPURenderer`，`try/catch`
回退到 `WebGLRenderer`；用 `'shadowMap' in renderer` 判断能力，不要假设某个渲染器一定有全部字段。

**降级规则**（来自 Arvix `logic.py`）：模型调用失败时返回一个最小 cube 场景，而不是抛错空白页。

---

## 3. 施工场景内容规则（场地 + 构件）

### 3.1 场地布置（site layout）
提示词中应点名以下要素及大致坐标，避免“只有一栋楼立在虚空”：

- 地面 / 网格参考 $\text{GridHelper}$ / 场地边界
- 施工道路（长条 `PlaneGeometry` + 道路中线）
- 围挡与边界立柱（可 `InstancedMesh` 复用）
- 大门 / 材料堆场 / 预拼装场（staging pads，用于“搭吊前暂存构件”）
- 塔吊基础位置、建筑红线（footprint marker）
- 临时设施（办公区、加工棚，可选）

construction-sim 的 `main.js` 可用作最小场地骨架：

```js
scene.background = new THREE.Color(0x8fb2c8);
scene.fog = new THREE.Fog(0x8fb2c8, 60, 160);
scene.add(new THREE.GridHelper(100, 40, 0x4e5a4a, 0x4e5a4a));   // 场地
// road / roadLine / boundary posts / footprint marker …
```

### 3.2 构件清单与尺寸（building.js 真实用值，可直接作为提示词参数）
| 构件 | 尺寸(m) | 颜色 | 说明 |
|---|---|---|---|
| 基础底板 footing | 8.4 × 0.5 × 8.4 | `0x7f7f79` | 预拼装，最先吊装 |
| 柱 column | 0.4 × 3.2 × 0.4 | `0xa8b2bd` | 每层 4 根 |
| 主梁 beam(主) | 8 × 0.6 × 0.3 | `0x8d99a6` | 沿 X，落在柱顶 |
| 次梁 beam(次) | 0.3 × 0.6 × 8 | `0x8d99a6` | 沿 Z，落在主梁端部 |
| 楼板 slab | 8 × 0.2 × 8 | `0xc9c2b4` | 落在次梁上 |
| 幕墙/墙板 facade | 0.08 × 3.2 × 7.0 | `0x9ec9de` | 透明 `opacity 0.55` |

关键几何规则（**必须写进提示词，否则必穿模**）：
- 各层竖向步距 `4.6 = 层高3.2 + 主梁0.6 + 次梁0.6 + 板0.2`；**主/次梁错开 0.6 高度**，
  使垂直方向的两根梁在角部不占据同一体积。
- 墙板长度 `2×(柱网−柱半宽) − 0.2`：**两端各留 0.1m 间隙**，否则被固定柱夹死、无法就位。
- 构件必须**物理接触/搭接**（相邻构件在接缝处重叠 0.02~0.05），禁止只“靠近不接触”。

### 3.3 施工工序（提示词的 D 段照抄）
```
1. 基础：场地准备 → 基础底板
2. 主体（逐层循环）：柱 → 主梁 → 次梁 → 楼板 → 墙板/幕墙 → 结构校验
3. 封顶：屋面层
4. 完工：全部楼层校验通过
```
- 构件交付分两类：`staged`（预拼装场直接取件）与 `truck`（车辆运抵后从车上吊取）。
- **先暂存、后吊装**：不要把“生产构件”写成凭空出现，要么场地上有堆场，要么有运输车。

---

## 4. 动画与时间轴规则（核心）

### 4.1 设备层级与枢轴（塔吊，来自 `crane.js`）
```
base(塔身, 固定) → slewing(回转组, 绕 Y 旋转 θ) → jib(起重臂)
                                        └ counter-jib(平衡臂+配重)
     jib → trolley(小车, 沿臂 X 平移 r) → hookGroup(吊钩组)
                                        ├ cable(钢丝绳, 竖直拉伸)
                                        └ hook block(滑轮+钩)
```
- 用**极坐标 `(θ, r, h)`** 描述吊钩位姿；世界点转极坐标：
  `θ = atan2(dz, dx)`，`r = hypot(dx, dz)`。
- 钢丝绳用 `cable.scale.y` 拉伸、`cable.position.y` 居中，**不要为每帧重建几何**。

### 4.2 运动规则（拟真，必须写进提示词）
- **分轴限速 + 指数逼近**：回转 `vT≈0.85`、变幅 `vR≈5.0`、起升 `vH≈8.0`，各自独立。
- **真实塔吊行为**：只有回转/变幅基本到位后，才允许起升（`crane.js` 中先判断
  `|dT|<0.004 && |r−this.r|<0.02`，否则垂直速度降到 `0.4×`）。
- **就位判据**：`|dθ|<0.004`、`|Δr|<0.02`、`|Δh|<0.03` 视为 settled。
- **释放条件（关键）**：只有塔吊**完全静止**、且构件水平误差 `<0.07`、垂直 `>目标` 在 `0.2` 内才脱钩；
  然后刚体由 `kinematic` 切成 `dynamic`，让重力落最后几厘米。
  > 否则“边回转边释放”会给构件水平速度，梁会砸落/滑移。
- **缓动**：吊钩/机器人用 `lerp` 或“限速步进”，不要用线性瞬移，也不要帧率相关的 `+= 常数`。

### 4.3 工序状态机（提示词里给出状态名，AI 才不会漏步骤）
```
DELIVERED → TO_PICK → ATTACH → LIFT → TRANSIT → LOWER → SETTLE
          → VALIDATE（校验/偏差异色）→ FASTEN（机器人拧紧）→ 下一构件
```
- 每个状态只做一件事，用 `switch(step)` 实现，便于生成与验收。
- `SETTLE` 用**物理步数**计时（60Hz × 0.7s ≈ 42 步），**不要用 `Date.now()`**，否则变速播放会失效。
- 校验失败 `RESEAT`：报废并重制构件、把同一工序 `unshift` 回队列，最多重试 3 次。

### 4.4 其他动画元素
- **运输车**：`kinematicPositionBased`，沿道路推进，车上构件与车同步；卸完货后驶离并消失。
- **装配机器人**：从基地 `lerp` 飞向连接点，旋翼 `rotation.y += dt*40`，到位后延迟闪烁。
- **反馈动画**：构件就位闪绿/黄；偏差异色；HUD 进度条随完成数更新。
- **关键帧动画**（可选，官方 AnimationMixer 写法）：
  ```js
  mixer = new THREE.AnimationMixer(model);
  mixer.clipAction(clip).play();          // clip 来自 gltf.animations 或自定义 KeyframeTrack
  // 每帧 mixer.update(delta)
  ```
  用于“装配/拆除/镜头动画”等非物理驱动的序列。
- **4D 进度**：构件按 `floor`、工序序号排序，时间轴推进即等于楼层推进。

### 4.5 动画铁律（img2threejs `animation_contract.md`）
> **“存在 clip” ≠ “clip 在播放”。** 必须在采样时刻测量实际值并与轨道值比对。
> 反例：11 个 clip、mixer 有状态、按钮有派发，但**零位移**，且代码审查看不出来。
- 一切时长/幅度用**相对单位**（如层高 H 的分数），不得假设 `H == 1.0`。
- 循环由 `poseReturn`（回到初始姿态）判定，不是由位移判定。
- 门禁：绑定误差 `≤ 2⁻²³`、形变有限、权重归一化、左右不镜像错、接触点不滑移。

---

## 5. 相机与交互规则

- **预设机位**（construction-sim `VIEWS`）：`overview / crane / truck / robot`，一键切换。
- `OrbitControls`：`enableDamping = true`、`dampingFactor ≈ 0.08`、
  `minDistance / maxDistance`、`maxPolarAngle = Math.PI/2.05`（防止钻到地下）。
- 初始 `camera.position` 与 `controls.target` 要给**具体数值**（如 `[36,26,36]` → `[0,9,0]`）。
- 进阶交互（写进提示词即生效）：点击选中构件、剖切面、爆炸图、聚焦某层。
  > img2threejs 要求模型“**可爆炸且可点击**”，且命名规范：每个 mesh 都要命名，
  > 表面细节用 `userData.explodeWithParent` 跟随父构件。

---

## 6. 光照、材质与视觉规则

- **太阳光**：`DirectionalLight` + `castShadow`，并**显式设置阴影相机范围**（否则远景阴影丢失）：
  ```js
  sun.position.set(38, 46, 22);
  sun.shadow.mapSize.set(2048, 2048);
  sun.shadow.camera.left = -32; sun.shadow.camera.right = 32;
  sun.shadow.camera.top = 32;  sun.shadow.camera.bottom = -32;
  sun.shadow.camera.near = 1;  sun.shadow.camera.far = 120;
  ```
- 环境光：`HemisphereLight(天空色, 地面色, 1.1)` + 少量 `AmbientLight`。
- 色调映射：`ACESFilmicToneMapping`；`shadowMap.type = PCFSoftShadowMap`。
- 天空/环境反射：可用 `Sky` + `PMREMGenerator.fromScene(sky)` 赋给 `scene.environment`（官方示例做法）。
- **材质分层描述**（img2threejs 推荐，写提示词时按字段给值）：
  `baseColor / colorVariation / roughness / metalness / normal / bump / wear / dirt / localOverrides`。
- 施工材质建议：钢构 `roughness 0.55 / metalness 0.35`；玻璃幕墙 `roughness 0.15 /
  metalness 0.1 / transparent / opacity 0.55`；混凝土 `roughness 0.9~1.0 / metalness 0`。
- 视觉失败信号（生成后自检）：每个边都过锐或过圆、单色无粗糙度变化、光照掩盖形体、
  重复细节间距过于均匀、近景细节只增加三角面不增加辨识度。

---

## 7. 几何建模规则

- **优先“代码化几何”**，不依赖外部模型：图元、`Shape` 挤出、曲线 `tube`、`InstancedMesh`、
  位移、canvas 程序化贴图。
- 图元选择表：
  - `BoxGeometry`：柱、梁、板、设备机身、块面
  - `CylinderGeometry`：柱、管、车轮、旋翼
  - `TubeGeometry`（沿曲线）：缆索、管线、钢筋
  - `InstancedMesh`：桁架竖杆、螺栓、围栏立柱、重复节点（**不要 for 循环加几百个 mesh**）
- **统一单位与尺度**：构件尺寸用米；根节点 `transform.scale` 必须是 `[1,1,1]`
  （否则子节点被整体缩放成“微粒”）；隐藏容器用透明材质，不要用 scale。
- **反“2.5D 板状”**：对每个 mesh 统计不同 Z 值（`round(z,3)` 去重）；
  若是“平板拉伸”，平面数会卡在 6~10，而真正的体块 > 11。旋转后越来越像纸板就是该问题。
- **接触规则**：相邻构件在接缝处必须重叠 `0.02~0.05`，用断言校验（如 `partA.end >= partB.start`）。
- **禁止**用 alpha 卡片（plane card）表示需要实体的东西；本类管线不产生贴图，卡片只会渲染成不透明矩形。

---

## 8. 性能与工程约束

- **三角面预算**：按构件重要性分档（低 ≤6k / 标准 ≤60k / 主角），超限用减面（decimate ratio，
  例如 `0.4`）且在**蒙皮之前**执行。
- **实例化**：重复构件/细节一律 `InstancedMesh`。
- **LOD**：远距离用低模。
- **静态物体不刷新**：`syncMeshes()` 只在刚体位移超过阈值时才更新对应 mesh（construction-sim 做法）。
- **阴影**：控制 `mapSize` 与阴影相机范围，避免全场景高分辨率阴影。
- **循环**：`Clock.getDelta()` 并 `Math.min(dt, 1/10)` 防止切标签页后大跳；物理固定 `timestep = 1/60`，
  并限制单帧最多步进 `8` 次。
- **依赖与版本**：锁定 three 版本；CDN 用 importmap 或固定 URL；`renderer.setPixelRatio(Math.min(dpr, 2))`。
- **交付物**：单文件 HTML（含脚手架）或 Vite 工程（`index.html + src/*.js + package.json`），
  两者都要求“打开即运行”。

---

## 9. 生成后验收 / 自检规则

1. **多角度查看**：前 / 右 / 后 / 左（转台）各出一张；单一视角不算证据。
2. **动画实测**：不能只看代码里有 `mixer`；要在采样时刻测量实际位移/旋转是否等于轨道值。
3. **施工逻辑校核**：柱有支撑、梁两端落在柱上、板落在梁上、墙板不夹死、构件不重叠穿模。
4. **容差校核**：位置偏差 `<0.08m` 为 OK、`0.08~0.25m` 为 WARN、`>0.25m` 或旋转 `>5°` 为 RESEAT。
5. **自纠决策**（img2threejs）：每次只做一个决定——
   `continue | refine-spec | refine-code | request-input | stop`；
   规格错就改规格，代码错才改代码；改不动就 `request-input` 或 `stop`，不要硬凑。
6. **诚实标注**：近似/风格化/低模要说明，不能假称“完全还原”。

---

## 10. 可直接复制的提示词模板（施工场景动画）

> 把方括号 `[...]` 替换成项目参数即可。

```
你是一名资深 Three.js 图形架构师。请用纯 JavaScript 生成一个可运行的施工场景动画。

【工程约束】
- 使用 three r[版本]；只输出裸 JS，禁止 Markdown/注释/解释。
- 已存在 scene / camera / renderer / controls，请勿重复创建。
- 地面用大 PlaneGeometry；所有物体放在原点附近；单构件尺度 [0.5, 5.0]。
- 材质用 MeshStandardMaterial / MeshPhysicalMaterial；每个 mesh castShadow=true，地面 receiveShadow=true。
- 动画用 delta 驱动；渲染循环用 setAnimationLoop；窗口 resize 要处理。
- 渲染器优先 WebGPU，失败回退 WebGL；pixelRatio 上限 2。

【场地】（总尺寸 [120×120]）
- 地面 + 网格；施工道路 [长120×宽7]，位于 [z=-10.5]；场地围挡立柱按四角布置；
- 预拼装场（staging pads）位于 [x=6.5.., z=5.5..]；建筑红线 [8.4×8.4] 半透明标出。

【建筑】（[5] 层，柱网 [3.8]，层高 [3.2]，竖向步距 [4.6]）
- 构件：基础底板、柱、主梁(X向)、次梁(Z向)、楼板、玻璃幕墙。
- 主/次梁错开 [0.6] 高度；墙板两端各留 [0.1] 间隙；相邻构件接缝重叠 [0.02~0.05]。
- 交付方式：基础与首层柱用 staged；其余用 truck 运抵。

【工序】
基础底板 → 逐层(柱 → 主梁 → 次梁 → 楼板 → 墙板 → 校验) → 封顶 → 完工报告。

【设备与动画】
- 塔吊层级：base → slewing(θ) → jib → trolley(r) → hookGroup(cable 用 scale.y)。
- 吊钩极坐标 (θ, r, h)；分轴限速 + 指数逼近；回转/变幅到位后才起升。
- 状态机：DELIVERED→TO_PICK→ATTACH→LIFT→TRANSIT→LOWER→SETTLE→VALIDATE→FASTEN。
- 仅当塔吊静止且水平误差<0.07、垂直在 0.2 内才脱钩，刚体由 kinematic 切 dynamic 落位；
  SETTLE 用物理步数(≈42步)计时。
- 运输车 kinematic 行驶；装配机器人 lerp 到连接点并旋转旋翼；就位后闪烁反馈。

【相机与交互】
- 预设机位 overview/crane/truck/robot，可一键切换；OrbitControls 带阻尼，限制俯仰与远近。

【光照与视觉】
- 太阳光 DirectionalLight + 阴影相机范围 + 2048 阴影贴图；HemisphereLight + AmbientLight；
- ACESFilmic 色调映射；天空/雾；现代施工配色（钢concrete 灰、幕墙蓝）。

【性能】
- 重复构件与桁架竖杆用 InstancedMesh；只更新移动物体的 mesh；物理固定 1/60，单帧最多 8 步。

请输出完整可运行代码，并附一段中文说明“施工顺序与关键判据”。
```

**英文缩略版**（适合只吃英文的模型）：
```
Senior Three.js architect. Generate runnable construction-site animation JS only (no markdown/comments).
Scaffold: scene/camera/renderer/controls already exist. Ground = large PlaneGeometry; objects near origin; scale 0.5–5; MeshStandard/PhysicalMaterial; castShadow on every mesh; delta-driven animation; setAnimationLoop; handle resize; WebGPU with WebGL fallback.
Site: ground grid, road, boundary posts, staging pads, footprint marker.
Building: 5 floors, grid 3.8, storey 3.2, vertical step 4.6; footing/column/primary beam/secondary beam/slab/glass facade; beams staggered 0.6; panels 0.1 clearance per end; parts must touch/overlap.
Sequence: footing → per-floor (columns→beams→slab→facade→validate) → roof → report.
Crane hierarchy: base→slewing(θ)→jib→trolley(r)→hook(cable.scale.y). Polar (θ,r,h); per-axis speed-limited easing; hoist only after slew/radius near.
State machine: DELIVERED→TO_PICK→ATTACH→LIFT→TRANSIT→LOWER→SETTLE→VALIDATE→FASTEN.
Release only when crane settled and horiz<0.07 & vert within 0.2; kinematic→dynamic drop; settle by physics steps (~42).
Trucks kinematic; robots lerp with spinning rotors; flash feedback on seat.
Camera: preset views + OrbitControls damping. Lighting: sun with shadow-camera bounds + 2048 map, hemisphere+ambient, ACESFilmic, sky/fog.
Perf: InstancedMesh for repeats; update only moving meshes; fixed 1/60 physics, ≤8 substeps.
```

---

## 11. 常见失败与规避（提示词里最好写成“禁止项”）

| 失败 | 规避规则（写进提示词） |
|---|---|
| 只有静态场景、没有动画 | 强制要求状态机 + 每帧 `update(dt)` |
| 边回转边脱钩导致构件砸落/滑移 | 只在 `settled` 且水平/垂直容差内释放，再切 dynamic |
| 主梁与次梁在角部重叠穿模 | 竖向错开 0.6；显式给出标高计算 |
| 墙板被固定柱夹死、悬空 | 两端各留 0.1m 间隙 |
| 构件“悬空”无支撑 | 校验支撑关系（柱→梁→板），接缝重叠 |
| 只有一个正面视角看不出问题 | 要求转台多角度 + 交叉截面检查 |
| 模型是平板拉伸（2.5D） | 用“不同 Z 值计数”自检，改用真实截面/loft |
| 钢丝绳不随高度变化 | 用 `cable.scale.y` 与位置联动 |
| 动画速度与帧率相关 | 一律 `delta` 驱动；物理步进用步数计时 |
| 用 scale 隐藏节点导致整体缩小 | 根节点 scale 保持 `[1,1,1]`，用透明材质隐藏 |
| 重复细节用几百个 mesh | 一律 `InstancedMesh` |
| 有 `mixer` 但画面不动 | 采样测量实际值；门禁校验绑定误差 |

---

## 12. 来源清单（本次联网检索到并核验）

| 来源 | 用途 | 链接 |
|---|---|---|
| three.js 官方示例 `webgl_animation_keyframes` | AnimationMixer / 渲染循环 / 光照 / resize 规范 | https://threejs.org/examples/webgl_animation_keyframes.html |
| `Mishi17o/ArvixAI-3D-Gen` | 自然语言→three.js 的**系统提示词**与固定脚手架（`src/prompts.py`、`src/renderer.py`、`src/logic.py`） | https://github.com/Mishi17o/ArvixAI-3D-Gen |
| `img2threejs/img2threejs` | 分阶段/门禁的代码化 three.js 生成规则（`SKILL.md`、`grimoire/build/geometry_patterns.md`、`grimoire/readiness/animation_contract.md`） | https://github.com/img2threejs/img2threejs |
| `caimingye78/construction-sim`（Tower Construct） | 施工模拟实现：塔吊 `src/crane.js`、工序状态机 `src/sim.js`、构件设计 `src/building.js`、车辆/机器人 `src/actors.js`、场景 `src/main.js` | https://github.com/caimingye78/construction-sim |
| three.js 官方文档/仓库 | API 行为（Skeleton/AnimationMixer/GLTFLoader 等） | https://threejs.org/docs/ · https://github.com/mrdoob/three.js |

> 注：本次检索环境对通用搜索引擎（Bing/DuckDuckGo/Brave/Ecosia 等）返回受限或被过滤，
> 因此规则主要来自**官方示例与 GitHub 上可核验的真实项目源码**，而非二次转载文章。

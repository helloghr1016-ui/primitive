# 语义手势协议 semantic_30 与原语重构技术方案

本文档定义手势原语库的**权威姿态语义协议（定长 30）**，以及从现网 **O20 语义 20 维**迁移的重构方案。
目标：分类指令保持稳定；连续姿态按人手解剖通道扩容（含**独立中节 pip**），经机型 profile 映射到 O30 / O20 / L25 / G20 / O6 / L6 等不同关节信号。

相关文档：

- 原语行为与分类：[`primitives.md`](primitives.md)
- ROS 接口：[`interfaces.md`](interfaces.md)
- 示教面板：[`teach_panel_architecture.md`](teach_panel_architecture.md)
- 控制模式与自定义姿态：[`control_modes_and_custom_pose.md`](control_modes_and_custom_pose.md)

**状态**：设计定稿，分阶段落地（见 §6）。**运行时内部已为 semantic_30（30 维）**；
**P5（O30）已落地**：`o30.yaml` 满映射 `from_semantic` + channels/coupling，executor 优先走通道名路径；
allowlist 与 o20 对齐（含 force/flick/twist/vision）；`vision_fk: true`，FK 使用
`evo_work/o30` mesh（`o30.yaml`: `urdf_model: o30`，`semantic_to_urdf_joint` 直映独立 PIP）。
弧度命令/反馈经 ``hardware_uint8_to_radians`` / ``to_semantic`` 与 `from_semantic` 对称。
**P2 收敛（§5.6）已落地**：`_feedback_command_converged` 在 semantic_30 上按 `channels`/`convergence` 比较；
**P3（大部分）**：`semantic_pose_config` / B 桶 vision 构姿 S30；电流 `hw_current_delta`；
`HandConfig` 解剖别名；FK 双读通道名键 + 接受 S30 入参（indexed 仍 O20 兼容 MOCK）；
**P4（已落地）**：twist* 经 `load_twist_poses` / `s30_pose_from_channels`（YAML
`gestures.twist*` 优先，缺省内置默认）；`tripod_force` / vision+ring close/freeze
已含 pip；C 桶 vision/夹持常量表（`index_ring`/`large_wrap`/`disk`/`parallel_extension`/
`index_middle_adduction`/`index_pinch`/`middle_pinch`）已改 S30 通道构姿，legacy 路径
不再 `project_s30_to_o20`。
经 `digit_flex_close_step` / `bump_digit_flex_chain` 含四指 pip；`HandConfig` 增
`index_pip` 等。
**P6（入口已落地）**：全机型 YAML **显式** `from_semantic`（`from_o20` 仅兼容别名）；
executor 主路径走通道名；示教/Pose Library **新建默认 `semantic_v3`**；
执行核内部量纲为 **semantic_rad**（见 `units_radians_migration.md`）；legacy255 仅边界。
`control_modes_and_custom_pose.md` 等若仍写「回放 → O20 semantic」，按投影理解。

---

## 1. 背景与问题

### 1.1 现状

| 层 | 当前实现 |
|----|----------|
| 指令 | 离散原语名（`init` / `pinch` / `tripod` / …） |
| 姿态语义 | 固定 **O20 20-DOF**（`primitive_base.JOINT_NAMES`），值域 0~255 |
| 机型适配 | 各手 `config/{model}.yaml` 的 `from_o20` + `capabilities` |
| 执行 | `executor`：语义 → 硬件 →（可选）URDF 弧度 |

原语内部统一跑在 O20 空间，方向正确；但 O20 将 `[11–14]` 标为 **预留且业务不写**，无法表达独立中节。

### 1.2 痛点（以 O30 为例）

O30 HOP 主动轴含 `root1 / root2 / tip`（及 roll/yaw）。
现网把 `root2` 临时挂在 O20 预留槽，原语默认不写 → **中节失控**，不是「自由度不够」，而是**语义欠表达**。

G20/L25 属 L20 族，与 O20 语义较贴；O30 是解剖 HOP 结构，需要显式 **pip（中节）** 通道。

### 1.3 设计原则

1. **分类 + 解剖角**：指令用分类名；连续姿态用解剖通道，不用 MANO PCA / MediaPipe 关键点当控制契约。
2. **协议按最高常用扩展定长**：**30**（手指过完备含独立中节 + 腕/掌/预留）。
3. **机型只用子集**：通道 mask + `from_semantic`；缺轴耦合在 profile，不在原语。
4. **感知另接**：MediaPipe / MANO → 投影进 semantic_30，再映射硬件。
5. **渐进兼容**：旧 `semantic_o20` / 20 维落盘经投影升级，避免一次改光所有原语类。

---

## 2. 目标架构

```text
① 分类语义（离散）
   open | pinch | tripod | power | …（与现有原语名 / grasp_type 对齐）
        ↓ 展开为姿态或过程
② semantic_30（连续，权威定长 30）
   解剖通道名；边界 unit=uint8|normalized；未启用轴 mask
        ↓ HandProfile（P2–P4: project + from_o20；P5+: from_semantic）
③ 机型信号
   O30 HOP / O20 SDK / L25 native / G20×16 / O6×6 / ros2_control 弧度
        ↑
可选感知
   MediaPipe 21 landmarks / MANO θ  →  ②（禁止直达 ③）
```

| 层 | 职责 | 非职责 |
|----|------|--------|
| ① | 任务/抓取类别、allowlist | 不绑定 DOF 数 |
| ② | 跨机型可插值、可 YAML 的姿态 | 不等于某款 SDK 关节序 |
| ③ | 总线/控制器真实命令 | 不含原语业务逻辑 |

命名约定（全文统一）：

| 用语 | 含义 |
|------|------|
| 协议名 | `semantic_hand` |
| 版本号 | `semantic_version: 3` |
| 口语 / 文件 | `semantic_30` |
| YAML `space` | `semantic_v3`（权威）；兼容读入 `semantic_o20` |

---

## 3. 语义协议 semantic_30

### 3.1 版本与定长

| 字段 | 值 |
|------|-----|
| 协议名 | `semantic_hand` |
| `semantic_version` | `3` |
| 向量长度 | **30**（固定，禁止运行时变长） |
| 旧版 | `semantic_o20`（version 隐含 1）：长度 20，经官方投影表升到 30 |

说明：人手手指常用完整驱动语义约 **20 DOF**；**30 = 5 指×5 槽（含独立 pip）+ 腕/掌 5**，属机器人扩展用过完备契约，不声称「生物肌腱恰好 30」。

### 3.2 通道布局（冻结）

#### 块 A — 手指 25（idx 0–24）

五指顺序：`thumb`, `index`, `middle`, `ring`, `pinky`。
每指固定 5 槽（指内偏移 0–4）：

| 指内偏移 | 后缀 | 拇指含义 | 四指含义 |
|----------|------|----------|----------|
| 0 | `mcp_flex` | CMC 屈（root1）| MCP 屈（root1 / base） |
| 1 | `mcp_abd` | CMC 侧摆（yaw）| 侧摆（yaw；中立见 §3.3） |
| 2 | `pip` | MCP 屈（常 mask）| **独立 PIP**（root2） |
| 3 | `dip` | IP 屈（tip）| DIP 屈（tip） |
| 4 | `rot` / `extra` | 前旋（roll）| 预留（额外侧摆等；默认可 mask） |

全局下标：`finger_index * 5 + offset`，其中 `finger_index`: thumb=0 … pinky=4。

拇指 offset 4 为 `rot`（前旋/roll），对应 O30 `thumb_roll`、O20 语义 `thumb_rot`（旧 idx 10）。
四指 offset 4 为 `extra`（预留），当前所有机型 mask。五槽定长结构不变，仅拇指最后一槽语义不同。

协议通道名统一用 `pinky_*`；O30 等硬件若为 `little_*`，**仅在 profile `from_semantic` 改名**，不进协议层。

#### 块 B — 腕与掌 5（idx 25–29）

| idx | 通道名 | 含义 |
|-----|--------|------|
| 25 | `wrist_flex` | 腕屈伸 |
| 26 | `wrist_deviation` | 腕尺桡偏 |
| 27 | `wrist_pronation` | 前臂旋前旋后（可 mask） |
| 28 | `palm_arch` | 掌弓 |
| 29 | `reserved` | 协议预留 |

手指原语默认**不修改** 25–29；带腕产品再开通道 mask。

**实施约束（Block B）**：

- P0–P5：25–29 全部通道 mask=false，profile 强制不透传硬件。
- 「禁止写入」定义：原语 / Engine 不得把 mask=false 的 Block B 通道设为**非保持值**（保持 = 上一帧或 0）。全 0 初始向量允许。违反在单测 assert。
- 未来带腕产品开放时新增 **P7**，不拉长向量。

### 3.3 量纲与中立

**对外契约（边界）**：YAML / 感知入口可用 `unit: normalized`，通道值 ∈ `[0.0, 1.0]`。

| 约定 | 含义 |
|------|------|
| flex/pip/dip：`0` | 伸直 / 打开侧 |
| flex/pip/dip：`1` | 最大弯曲（语义满程，非角度制） |
| `mcp_abd`：`0.5` | 侧摆中立（对应内部 uint8 **128**） |
| 腕/掌 | 同上；未定义方向时在 profile 注释写明 |

**过渡策略（冻结）**：

- **P0–P5**：运行时内部表示保持 **uint8 刻度（0~255）**；`ABD_NEUTRAL=128`。
- YAML 默认 `unit: uint8`；`unit: normalized` 仅在加载器边界换算后进入内部。
- 原语与 Executor **不得**各维护一套默认量纲。
- P6 完成后评估是否全量切归一化；切换只动加载器，原语不感知。

### 3.4 能力：功能 token 与通道 mask

两层不要混用：

| 层 | 用途 | 示例 |
|----|------|------|
| **功能 token**（现网 `capabilities`） | catalog / allowlist（原语要不要这只手） | `finger_abd`, `thumb_rot`, `finger_root2`, `vision_fk` |
| **通道 mask**（对齐附录 A 名） | 执行忽略、收敛、Block B | `index_pip: true`, `thumb_pip: false`, `wrist_flex: false` |

P2 起 profile 增加 `channels:`（或等价段）；加载器可将旧功能 token **展开**为通道 mask（见下表）。P0–P5 期间 YAML **双读**：只写旧 token 的机型仍可跑。

**旧功能 token → 通道 mask 展开（默认）**：

| 旧 token | 展开为 true 的通道（其余同档默认 false） |
|----------|------------------------------------------|
| `thumb_abd` | `thumb_mcp_abd` |
| `finger_abd` | `index/middle/ring/pinky_mcp_abd` |
| `thumb_rot` | `thumb_rot` |
| `thumb_tip` | `thumb_dip` |
| `finger_tips` | `index/middle/ring/pinky_dip` |
| `finger_root2` | `index/middle/ring/pinky_pip`（**不含** `thumb_pip`） |
| `vision_fk` / `touch` / `current` / `motor_torque` | 不展开为位置通道（仍为功能 token） |

显式 `channels:` 覆盖展开结果。

```yaml
semantic_version: 3
capabilities:                    # 功能 token（catalog）
  finger_abd: true
  thumb_rot: true
  finger_root2: true
  vision_fk: false
channels:                        # 通道 mask（可省略，由上表展开）
  index_pip: true
  thumb_pip: false
  index_extra: false
  wrist_flex: false
from_semantic:
  index_mcp_flex: { joint: index_root1, invert: false }
  index_pip:      { joint: index_root2, invert: false }
  index_dip:      { joint: index_tip,   invert: false }
  pinky_mcp_flex: { joint: little_root1, invert: false }  # pinky→little 仅此处
coupling:
  thumb_pip: { mode: ignore }    # O30：拇指无独立 MCP 屈轴
  index_pip:                     # 无 pip 机型示例
    mode: copy_from
    source: index_dip
```

**耦合模式（coupling mode）**：

| mode | 含义 | 适用场景 |
|------|------|----------|
| `ignore` | 丢弃原语写入，硬件不发此关节 | 无对应硬件轴（如 O30 `thumb_pip`） |
| `copy_from` | 复制 `source` 通道的值 | O20 缺 pip → 跟随 dip |
| `fixed` | 锁定为 `value` 常量 | abd 中立（`value: 128`） |
| `scale` | `source` × `factor` | pip = dip × 0.7 |
| `also` | 同一语义通道额外驱动其它硬件关节（等价现网 `from_o20.also`） | L25 tip→mid |

示例（L25）：

```yaml
coupling:
  index_pip: { mode: ignore }
  index_mcp_abd: { mode: fixed, value: 128 }
  index_dip: { mode: also, also_joints: [index_mid] }
```

规则：

1. 原语可写未启用通道；profile **ignore / coupling**，不得抛到硬件未知关节。
2. 左右手镜像：在 profile 规定 abd 等符号，不进原语分支。
3. 力/电流监测：通道 → 机型硬件下标的表与位置映射**分文件段**维护。

### 3.5 与旧 O20 的官方投影

| 旧 O20 名 / idx | → semantic_30 |
|-----------------|---------------|
| `thumb_base` (0) | `thumb_mcp_flex` (0) |
| `index_base` … `pinky_base` (1–4) | 对应 `*_mcp_flex` |
| `thumb_abd` (5) | `thumb_mcp_abd` (1) |
| `index_abd` … `pinky_abd` (6–9) | 对应 `*_mcp_abd` |
| `thumb_rot` (10) | **`thumb_rot` (4)**；O30 → `thumb_roll` |
| `rsv_11` … `rsv_14` (11–14) | `index_pip` … `pinky_pip` |
| `thumb_tip` … `pinky_tip` (15–19) | 对应 `*_dip` |
| （无） | 腕/掌、四指 `extra`、`thumb_pip` 默认 → 0 + mask |

投影函数：

- `project_o20_to_s30(vec20) -> vec30`
- `project_s30_to_o20(vec30) -> vec20`（lossy）

**往返单测契约**：

| 字段 | o20→s30→o20 |
|------|-------------|
| 业务 16 轴（base/abd/rot/tip，不含 rsv） | 恒等 |
| `rsv_11..14` ↔ 四指 `pip` | 恒等 |
| `thumb_rot` ↔ s30 idx 4 | 恒等 |
| Block B、四指 `extra`、`thumb_pip` | s30→o20 **丢弃**；o20→s30 **填 0** |
| abd 中立 | 内部 128；normalized 边界 0.5（加载器换算，不在投影层） |

YAML `space: semantic_o20` 在加载器内先投影再进原语/执行器。

### 3.6 映射路径过渡（冻结）

| 阶段 | Executor 映射 |
|------|----------------|
| **P2–P4** | `vec30` →（需要时）经兼容层 → 现有数字下标 **`from_o20`** → 硬件 / 弧度 |
| **P5+** | `vec30` → **`from_semantic`（通道名）** → 硬件 / 弧度；`from_o20` 仅作别名至 P6 删除 |
| **禁止** | 跳过投影、在 primitive 内直接写机型 native 序 |

### 3.7 非协议内容（明确排除）

| 表示 | 用途 | 可否当 ② |
|------|------|----------|
| MediaPipe 21 landmarks | 视觉/遥操入口 | 否 → 投影进 ② |
| MANO θ / PCA / mesh | 重建与数据集 | 否 → retarget 进 ② |
| Feix / Cutkosky 类别 | ① 分类层 | 是（指令名） |
| 机型 native 0~255 序 | ③ | 否 |

**本轮不做的对外消息约定**（避免各端猜测）：

- `/{side}_hand/stream_target`、status/result 内连续角：落地前仍按现网；P2 起若暴露 semantic，须带 `space=semantic_v3` 且长度 30（或显式 `semantic_o20` 由节点投影）。
- 不改 ros2_control 关节名；腕通道不并入臂规划。

---

## 4. 机型映射关系

| 机型 | 典型启用子集 | 备注 |
|------|--------------|------|
| O6 / L6 | 五指 mcp_flex + thumb abd 等 | 大量 mask；无 tip/pip |
| G20 / L25 | mcp_flex/abd + dip 等 | pip 常 `ignore`；FK 可复用 L25 |
| O20 | 同旧语义 | pip 常 mask 或 `copy_from` dip |
| **O30** | mcp_flex/abd + **四指 pip** + dip + thumb rot | 与 HOP 对齐；`thumb_pip` mask |
| 未来 ≤30 主动 | 填 `from_semantic` | 不改向量长度 |
| 超 30 主动（罕见） | 折进已有槽或另开扩展消息 | 不拉长本协议 |

O30 完整映射（关节名对齐 `o30_hand_only.urdf.xacro` / HOP `NAMES_EN`）：

| semantic_30 通道 | O30 HOP 关节 | channels | 备注 |
|-----------------|-------------|----------|------|
| `thumb_mcp_flex` | `thumb_root1` | true | CMC 屈 |
| `thumb_mcp_abd` | `thumb_yaw` | true | CMC 侧摆 |
| `thumb_pip` | — | false | `coupling: ignore`；无独立 MCP 屈轴 |
| `thumb_dip` | `thumb_tip` | true | IP 屈 |
| `thumb_rot` | `thumb_roll` | true | 前旋（旧 O20 `thumb_rot`） |
| `index_mcp_flex` | `index_root1` | true | |
| `index_mcp_abd` | `index_yaw` | true | |
| `index_pip` | `index_root2` | true | 独立 PIP |
| `index_dip` | `index_tip` | true | |
| `index_extra` | — | false | |
| `middle_*` / `ring_*` | 同 index（root1/yaw/root2/tip） | | |
| `pinky_mcp_flex` 等 | **`little_root1` / `little_yaw` / `little_root2` / `little_tip`** | | 协议名 pinky，硬件名 little |
| Block B (25–29) | — | false | §3.2 |

---

## 5. 原语重构方案

### 5.1 运行时目标

```text
Primitive.update(ctx)
  → 读写通道名或 SemanticHand30 视图
  → PrimitiveResult.target_angles: length 30, space=semantic_v3（内部 uint8 刻度）
  → Executor: channel mask →（P2–P4: from_o20 / P5+: from_semantic）→ hw / radians
```

薄 API（模块 `semantic_30.py`）：

```text
sem.finger("index").mcp_flex / .pip / .dip / .abd
sem.thumb.mcp_flex / .abd / .pip / .dip / .rot
sem.as_vector() -> List[30]
```

**HandConfig 双类演进**：

| 类 | 位置 | 职责 | 迁移 |
|----|------|------|------|
| `primitive_base.HandConfig`（别名 `SemanticPoseConfig`） | dataclass | 语义索引查表 | 经 `semantic_pose_config()` 取用；与硬件类分离 |
| `hand_config.HandConfig` | YAML 驱动 | `to_hardware()` / `from_semantic()` | P5 主路径 |

P3 后 primitive 经通道名访问；vision/ring legacy 全程 S30；四指预成型经
`apply_four_finger_pip`；C 桶目标经 `sync_pip_from_dip` 填 root2。

### 5.2 现有原语分桶

| 桶 | 代表 | 改法 |
|----|------|------|
| **A. YAML 静态** | init/open/fist/pinch/point/ok/v_sign | 加载器出 30 维；稀疏名迁解剖名；**.py 少改** |
| **B. HandConfig 索引** | 多数 by_vision / grip | **已落地**：`semantic_pose_config` + `apply_four_finger_pip`；legacy 不再 `project_s30_to_o20` |
| **C. 硬编码 20 维字面量** | twist*、vision 常量表 | **已落地**：`s30_pose_from_channels`；禁止新增 len=20 裸表 |
| **D. 分阶段/力控 + YAML** | tripod、large_wrap、YAML gestures | close/freeze 含 pip；**o20/l25 gestures 已迁 `semantic_v3`**；**内置 `_DEFAULT_*` 亦为 semantic_v3/S30** |
| **E. named_pose / 示教** | teach_panel | 见 §5.7（以 schema 规则为准，勿与「凡旧包都投影」混读） |
| **F. 门控/目录** | grasp_gate、catalog | 基本不动（属 ①）；功能 token 双读 |

### 5.3 改造模板

**旧：**

```python
angles[cfg.index_base] = 80
angles[cfg.index_tip] = 60
```

**新：**

```python
s = SemanticHand30.from_vector(angles)
s.index.mcp_flex = 80
s.index.pip = 40        # O30 进 root2；无能力由 profile 消化
s.index.dip = 60
return s.as_vector()    # len == 30
```

**YAML：**

```yaml
angles:
  space: semantic_v3      # 或过渡期 semantic_o20（加载器投影）
  unit: uint8
  target:
    index_mcp_flex: 80
    index_pip: 40
    index_dip: 60
```

### 5.4 O30 原语策略（协议生效后）

| 原语族 | 要求 |
|--------|------|
| open / init | **四指**显式设 `*_pip`≈伸直；拇指写 mcp_flex/dip/rot，**不依赖** `thumb_pip` 进硬件 |
| fist / pinch / point | 四指屈曲链 mcp → **pip** → dip；拇指 mcp → dip（+ rot 按需） |
| tripod close | 食/中（及参与指）flex **含 pip**；拇指不含 pip 硬件轴 |
| 力控 | 监测映射含 `*_root2`（通道 mask 开时） |
| allowlist | 先扩 A 类，再 D，再 vision |

### 5.5 Engine 层迁移（**必须在 P2 与扩维同批**）

> **硬约束**：禁止「`target_angles` 已是 s30 布局，Engine 仍写 O20 下标 `[5]`/`[10]`」。
> 在 s30 中 idx 5 = `index_mcp_flex`，idx 10 = `middle_mcp_flex`，会动错指。

`gesture_engine.py` 依赖 O20 位置索引：

| 引擎 | 硬编码 | 含义 |
|------|--------|------|
| `PhasedPoseEngine` | `[5]` / `[10]` | thumb_abd / thumb_rot |
| `PhasedLerpEngine` | `[5]` | thumb_abd |
| `RotProgressiveEngine` | `rot_joint=10` | 已参数化，改默认即可 |

**P2 内必须完成**（与长度 30 同批合并）：

- `[5]`/`[10]` → canonical **`thumb_mcp_abd=1` / `thumb_rot=4`**
- `phase1_hold` 默认 `[0, 5, 10, 15]` → **`[0, 1, 4, 3]`**（mcp_flex, abd, rot, dip）
- `RotProgressiveEngine` 默认 `rot_joint`：10 → **4**
- `adaptive_thumb_adduction_angles()` 中 `angles[5]` → abd canonical index
- `SequentialForceCloseEngine` / `StaticPoseEngine`：无需改（已参数化 / 无索引）

P3 **不再**承担 Engine 硬编码迁移（仅 HandConfig 通道化与 A/B 改名）。

### 5.6 收敛检测掩码

`executor._feedback_command_converged()` 现状：

| 手型 | 当前 active_indices |
|------|---------------------|
| L25 | `(0, 1, 2, 3, 4, 5, 10)`（语义主动轴） |
| O30 / O20 | `range(20)` |
| O6 / L6 | `range(num_joints)`（偏硬件长） |

**P2 调整（写死比较空间）**：

1. 先将命令与反馈统一到 **同一空间再比**：
   - 优先在 **semantic_30** 上比较：state 经 `to_semantic` / `from_hardware`→投影得到 vec30；
   - 默认 `active_indices = convergence_channel_indices()` ← `channels` 启用集；
   - 可选 YAML `convergence:` 通道名列表覆盖（如 L25 排除耦合 tip/mid）。
2. Block B、各指 `extra`、mask=false 的 `thumb_pip` 等不参与。
3. O6 等低 DOF：只比较 mask=true 的通道（展开后通常远少于 30），**不要**再用裸 `range(num_joints)` 与 semantic 下标混比。
4. 阈值 20.0（uint8 ≈8%）不变；若 P6 切 normalized → 0.08。

### 5.7 Named Pose 序列化

`pose_library.py` 现状：`POSE_SCHEMA_VERSION = 1`，多为 `space: hardware`。

| 版本 | space | 长度 | 含义 |
|------|-------|------|------|
| 1 | `hardware` | 机型相关 | 现有；**不**自动投影 |
| 2 | `semantic_v3` | 30 | 跨机型可复用 |
| 2 | `hardware` | 机型相关 | 示教标定 |
| 2 | `semantic_o20` | 20 | 读入时 **`project_o20_to_s30`** 升级 |

规则（E 桶与此对齐）：

- P2 起 `POSE_SCHEMA_VERSION = 2`
- version=1 → 按 hardware 处理，不投影
- 仅当 `space: semantic_o20`（或显式 20 维 semantic）→ 自动投影
- 新建默认 `semantic_v3`；示教可选 `hardware`
- 扩展 teach_panel `_ALLOWED_SPACES`

### 5.8 下游 O20 下标消费者（checklist）

与 `NUM_JOINTS=20` grep 同级，列入 P2/P3：

| 模块 | 阶段 | 动作 |
|------|------|------|
| `gesture_engine.py` | **P2** | §5.5 canonical 索引 |
| `executor.py` 收敛 / 过流 `SKIP_OVERLOAD` | **P2** | 通道 mask；过流表按通道或 hw 序文档化 |
| `joint_state_mapping.py` | **P2** | 语义长度 30 或明确仍输出机型长 |
| `pose_library.py` | **P2** | schema v2 |
| `gesture_params.py` | **P1–P2** | 加载统一 30 |
| `primitive_base.HandConfig` / `tip_or_base` / `present_joints` | **P3** | 通道化 |
| `fk_solver.py` | **P3** 或 vision 启用前 | joint_map 离 O20 idx |
| `contact_resolver.py` | **P3** | `_PINCH_MONITOR_*` 改通道 |
| `tripod_force.py` | **P4** | freeze/monitor 含 mcp + 可选 pip（`channel_enabled`）+ dip |
| `joint_currents` 维数 | **P2** 起 | 与 **hw 序**对齐，或按通道 map；禁止 silently 假定 20=semantic |

---

## 6. 落地阶段

| 阶段 | 内容 | 原语 .py | 风险 |
|------|------|----------|------|
| **P0** | `SEMANTIC_30_NAMES`、o20↔s30 投影与往返单测、本文档 | 不动 | 低 |
| **P1** | `gesture_params` 加载统一产出 30 维；仍接受 `semantic_o20` | 不动 | 低 |
| **P2** | 长度 30 全路径；**Engine 索引与扩维同批**（§5.5）；收敛 mask（§5.6）；Pose schema v2；下游 checklist 中 P2 项；映射仍走 **from_o20** | `[:20]`→`[:30]` + Engine 默认索引 | 中 |
| **P3** | `HandConfig` dataclass 废弃 + 通道 API；A/B 改名；fk/contact 等 P3 checklist | 分批 | 中 |
| **P4** | C/D 去字面量；close/freeze 含四指 pip；twist YAML 化 | **已落地**（`load_twist_poses` + vision/force pip） | 中 |
| **P5** | `o30.yaml` **`from_semantic` 满映射** + channels + 扩 allowlist + 真机标定 | 配置为主 | 中 |
| **P6** | 废弃 rsv 业务用法与 `from_o20` 主路径；评估 uint8→normalized；更新 primitives.md / interfaces.md | **入口已落地**（合成 from_semantic；normalized 延后） | 低 |

**推荐顺序（冻结）**：

```text
P0 → P1 → P2（含 Engine 索引）→ P5（真机 pip）→ P3 → P4 → P6
```

理由：P5 验证 pip 不依赖 HandConfig 通道化；但 **依赖 P2 已正确的 Engine 下标**。
**禁止**：P2 扩维后、Engine 未改索引就进 P5。

**P3 scope**：仅 HandConfig / A/B / §5.8 的 P3 项。**不含** Engine 硬编码（已在 P2）、不含 C/D（P4）。

### 6.1 明确本轮不做

- 按机型 fork 整套 primitive 类
- MANO/MediaPipe 直接当 `target_angles`
- 一次改完所有 by_vision 魔法数
- 将协议拉长到 45+
- 开放 Block B 腕控（→ P7）
- 改 ros2_control / URDF 关节命名契约

---

## 7. 测试与验收

| 项 | 验收 |
|----|------|
| 投影 | §3.5 往返表；lossy 字段文档化 |
| 长度 | 执行路径 assert `len(target)==30` |
| Engine | PhasedPoseEngine 在 s30 下 thumb 相位动 **abd/rot**，不动 index/middle mcp_flex |
| 回归 | MOCK 下 L25/O6/O20 allowlist 仍可完成 |
| O30 | `COMM_TYPE=libcanbus`：init/open/pinch；**四指 root2** 相对旧版有可见运动 |
| 示教 | v1 hardware 不误投影；`semantic_o20` 可升级；v2 `semantic_v3` round-trip |
| 通道 | mask=false 写 pip/thumb_pip 不炸、不写未知关节 |
| 收敛 | 在 semantic 统一后按 channels 比较；O6 只比启用通道 |
| 耦合 | ignore / copy_from / fixed / scale / also 单测 |
| Block B | 非保持写入触发 assert；不透传硬件 |
| 映射过渡 | P2 路径仍能吃现网 `from_o20`；P5 后 `from_semantic` 黄金用例 |

---

## 8. 风险与缓解

| 风险 | 缓解 |
|------|------|
| 30 维空洞 | UI 只展示启用通道；默认 0 |
| `thumb_rot` 歧义 | 写死 s30 idx 4 → O30 `thumb_roll`；不是四指 extra |
| **30 维 + O20 Engine 下标** | §5.5 / §6：Engine **并入 P2** |
| 0~1 与 0~255 混用 | 边界 `unit` + 内部 uint8 至 P5 |
| 功能 token vs 通道 mask | §3.4 分层 + 展开表 + 双读 |
| `joint_currents` 维数 | 与 hw 序对齐或通道 map（§5.8） |
| 漏改 `NUM_JOINTS=20` | CI grep + §5.8 checklist |
| 回归面大 | P2 后先 O30（P5），再清 L25 默认路径（P3/P4） |

---

## 9. 目录与模块建议（实施时）

```text
hand_gesture_primitives/
  docs/
    semantic_30_refactor.md     # 本文件（契约）
  hand_gesture_primitives/
    semantic_30.py              # 名称表、投影、SemanticHand30（P0）
    gesture_params.py           # 加载 → 30（P1）
    executor.py                 # 长度、from_o20/from_semantic、收敛（P2/P5）
    gesture_engine.py           # canonical 索引（P2，见 §5.5）
    pose_library.py             # schema v2（P2）
    primitive_base.py           # HandConfig 废弃（P3）
    hand_config.py              # from_semantic（P3 起，P5 主路径）
    fk_solver.py / contact_resolver.py / tripod_force.py  # §5.8
    config/
      o30.yaml                  # from_semantic + channels（P5）
      o20.yaml / l25.yaml ...   # token 双读 → channels 展开
```

---

## 10. 决议摘要

1. **权威姿态协议 = semantic_30**（version 3；5×5 含四指独立 `pip` + 拇指 `rot` + 腕/掌 5）。
2. **分类层不变**；感知不进入控制契约。
3. **原语重构** = 向量 20→30 + 通道解剖化 + close/freeze 纳入**四指** pip；拇指不依赖 `thumb_pip` 硬件。
4. **兼容**靠 o20↔s30 投影；**价值验证**优先 O30 真机（P5）。
5. **P2 必须同时完成：长度 30 + Engine canonical 索引**；禁止未改索引就进 P5。
6. **推荐顺序**：P0→P1→P2→**P5**→P3→P4→P6。
7. **映射**：P2–P4 用投影 + `from_o20`；P5 起主路径 `from_semantic`。
8. **内部量纲**：P0–P5 保持 uint8；归一化仅边界。
9. **capabilities**：功能 token（catalog）与通道 mask（执行）分层；旧 token 可展开；`pinky_*`↔`little_*` 仅 profile。
10. 禁止跳过投影层直接改光全部 primitive 文件。

---

## 附录 A — 通道全名列表（idx → name）

```text
 0 thumb_mcp_flex    1 thumb_mcp_abd    2 thumb_pip    3 thumb_dip    4 thumb_rot
 5 index_mcp_flex    6 index_mcp_abd    7 index_pip    8 index_dip    9 index_extra
10 middle_mcp_flex  11 middle_mcp_abd  12 middle_pip  13 middle_dip  14 middle_extra
15 ring_mcp_flex    16 ring_mcp_abd    17 ring_pip    18 ring_dip    19 ring_extra
20 pinky_mcp_flex   21 pinky_mcp_abd   22 pinky_pip   23 pinky_dip   24 pinky_extra
25 wrist_flex       26 wrist_deviation 27 wrist_pronation 28 palm_arch 29 reserved
```

## 附录 B — 与人手表示调研的对应关系

| 表示 | 本方案落点 |
|------|------------|
| Feix / Cutkosky 等分类 | ① 指令 / grasp_type |
| 解剖关节角 | ② semantic_30 |
| MediaPipe landmarks | 入口 → ② |
| MANO | 入口 → ② |
| 协同/PCA | 欠驱动 profile 内部，不替代 ② |
| 接触/力 | 功能 token + 力控原语，不并入位置 30 维 |

## 附录 C — 修订记录（相对审查）

| 问题 | 处理 |
|------|------|
| P2 扩维与 Engine O20 下标冲突 | Engine 迁入 **P2**；P3 不再改 Engine；禁未改索引进 P5 |
| `from_o20` / `from_semantic` 过渡不清 | §3.6 冻结阶段路径 |
| 功能 token vs 通道 mask | §3.4 分层 + 展开表 |
| §2「归一化」与内部 uint8 矛盾 | 改为边界 unit + 内部 uint8 |
| 拇指策略含 pip | §5.4 改为四指 pip；拇指 ignore |
| NamedPose E 桶与 schema 不一致 | 以 §5.7 为准 |
| 下游消费者遗漏 | §5.8 checklist |
| pinky / little | §3.2 / §4 写明仅 profile 改名 |
| 收敛比较空间 | §5.6 统一 semantic 再按 channels 比 |
| 跨文档权威 | 文首声明 |

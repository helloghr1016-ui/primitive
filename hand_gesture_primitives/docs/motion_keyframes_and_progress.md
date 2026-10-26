# 自定义多关键帧手势与进度细控 — 技术方案（开发指南）

> **状态**：P0–P2 已落地（KeyframeRunner + Pose `keyframes[]` + `pose` 指令 +
> `motion_progress` topic + `gesture_result` 段状态）；内置 prep progress（`tripod`/`large_wrap`）已落地。
> **范围**：Pose Library 多关键帧、段内 progress 细控、统一位置插值驱动层；以及由此引出的架构优化。
> **非范围**：自定义力控可视化编辑、跨手型裸关节 remap、STREAM 与 progress 合并、全局一条 0～1 覆盖全部关键帧（二期可选）。

**PROGRESS 语义**：外部 scrub 扫到 `p=1` **不**卸载原语（可回拉）；仅 AUTO 跑完末段置 `done`。
`progress_driven` 期间 `gesture_result` 不以反馈收敛冒充 `DONE`。

相关文档：

| 文档 | 关系 |
|------|------|
| [`control_modes_and_custom_pose.md`](control_modes_and_custom_pose.md) | 三模式 + 单终态 Pose Library（本文扩展其「示教回放」） |
| [`units_radians_migration.md`](units_radians_migration.md) | 落盘 / 运行时量纲（新关键帧默认 `semantic_v3` + `rad`） |
| [`semantic_30_refactor.md`](semantic_30_refactor.md) | S30 协议；关键帧向量长度与投影 |
| [`interfaces.md`](interfaces.md) | Topic / 指令面（落地后同步更新） |
| [`teach_panel_architecture.md`](teach_panel_architecture.md) | 示教存帧 UX |
| [`primitives.md`](primitives.md) | 内置 prep/close 与力控边界 |

**实现包路径**：`hand_gesture_primitives/hand_gesture_primitives/`
（下文模块名均相对该目录。）

---

## 1. 需求定义（正交拆解）

产品表述常混为一谈，实现上必须拆成三个正交问题：

| ID | 需求 | 本质 | 本期 |
|----|------|------|------|
| **A** | 自定义多中间态（pre / pre1 / pre2 / close…） | **数据**：有序关键帧序列 | ✅ |
| **B** | 对某一阶段 `progress ∈ [0,1]` 细控 | **驱动**：墙钟 auto vs 外部 progress | ✅ |
| **C** | 内置 `prep` / `close` / `auto` | **行为**：命令相位；部分含力控 | ⚠️ 仅**纯定位段**可接 B；力控段拒绝 |

### 1.1 典型用例

1. **示教**：TEACH 下调到姿态 A → 存为 `pre`；再调到 B → 存为 `pre2`；再调到 C → 存为 `grasp`。
2. **顺序回放**：`pose my_grip` → 按 keyframe 顺序 auto 插值跑完。
3. **精细对位**：`pose my_grip pre2 0.35` → 停在 pre→pre2 段的 35% 处，可反复改 progress 不重锚。
4. **内置（有限）**：`tripod prep 0.5` → 仅当该相位为位置插值时允许；`tripod_by_force close 0.5` → **拒绝**。

### 1.2 明确不做（本期）

- 把 STREAM（绝对关节流）与 progress（关键帧间插值）做成同一种模式。
- 自定义编辑 PhasedLerp / 力控 / by_vision 状态机。
- 一条引擎侧 `global_progress` 线性映射整条多关键帧轨迹（易与段内 p 混淆）。
  Teach 面板 UI 已在前端按时长加权映射到段内 `keyframe+local`，再走 `scrub_pose`。
- 为每个自定义姿态新增 Python 原语类。

---

## 2. 当前架构审查（是否合理）

### 2.1 结论摘要

| 维度 | 评价 | 说明 |
|------|------|------|
| **执行边界** | ✅ 合理 | 单一 `GestureExecutor` + `ControlMode` 门控发布 |
| **示教回灌** | ✅ 合理 | TEACH 落盘 → PRIMITIVE 下 `NamedPose`，不另建执行器 |
| **语义协议** | ✅ 合理（迁移中） | S30 + 默认 `semantic_v3`/`rad`；legacy 边界兼容 |
| **引擎层** | ⚠️ 需收敛 | 插值/`t` 计算分散；多套「阶段」语义过载 |
| **元数据** | ⚠️ 需对齐 | catalog / gate / planable / YAML allowlist 多源 |
| **指令与观测** | ⚠️ 偏弱 | String cmd 够离散；高频 progress / 段状态暴露不足 |
| **Pose 模型** | ❌ 不足 | 仅单终态；无法表达需求 A |

**总评**：控制权与发布边界设计正确，**应保留**；为支撑 A+B，需要一次**有节制的中间层抽取**，而不是在 `NamedPose` / 各 Engine 上继续打补丁。不必推倒原语注册表或力控状态机。

### 2.2 现网执行链（保留）

```text
/{side}_hand/gesture_cmd
        → GestureNode（解析 mode / pose / 原语 / prep|close）
        → GestureExecutor.set_primitive(...)
        → tick @ control_rate
             PrimitiveContext + elapsed
             → HandGesturePrimitive.compute()
             → _publish_cmd（受 ControlMode 门控）
```

权威模块：

| 主题 | 文件 |
|------|------|
| 指令解析 | `gesture_node.py` |
| 控制权 / 发布 | `executor.py`（`ControlMode`） |
| 原语基类 | `primitive_base.py` |
| 插值引擎 | `gesture_engine.py` |
| 示教库 | `pose_library.py`（`PoseEntry`，schema version=2） |
| 自定义回放 | `primitives/named_pose.py` |
| 可规划白名单 | `planable_primitives.py` |

### 2.3 今日能力边界（与需求对照）

| 能力 | 今日 | 缺口 |
|------|------|------|
| 自定义姿态 | `PoseEntry(positions, duration)` 单点 | 无关键帧序列 |
| 回放 | `NamedPose` → `StaticPoseEngine(elapsed/duration)` | 无外部 progress |
| `pose` 指令 | `pose <name>`；多余参数 **警告后忽略** | 无 stage/progress 语法 |
| 内置 prep/close | `PHASED_GRASP_PRIMITIVES`（6 个） | 与示教关键帧无关；YAML `stages` 是**时长旋钮** |
| PlanGesture | 13 个静态原语名义采样 | 无多段 / 无 progress 预览 |
| STREAM | 已实现 | ≠ 关键帧 progress |

### 2.4 术语过载（开发前必须读）

同一词在现网有多种含义，**新 API 禁止再复用 `stages` 表示关键帧列表**：

| 现网用语 | 真实含义 | 新方案用语 |
|----------|----------|------------|
| YAML `gestures.*.stages.prep/close` | prep/close **时长（秒）** | 保持；称「命令相位时长」 |
| `PhasedPoseEngine` / `motion.type: phased_pose` | **单终态**、关节子集分时到位 | 保持；称「关节分时」 |
| `PHASED_GRASP_PRIMITIVES` + `prep\|close\|auto` | **命令相位**（可含力控） | 保持；称「抓取相位」 |
| twist* 内局部 `stages=[...]` | 硬编码循环段 | 长期可迁 Runner；本期不动 |
| catalog `phased=True` | 元数据标签（与上者不完全一致） | 中期对齐 |
| **本文 `keyframes[]`** | 示教/配置的有序位置关键帧 | **唯一**表示「多中间态数据」 |

### 2.5 现网架构债（与本需求相关 / 并行）

| ID | 问题 | 影响 | 建议时机 |
|----|------|------|----------|
| D1 | `t=elapsed/duration` 分散在 Static / Phased / Lerp / pinch / twist | 加 progress 易复制粘贴 | **P0 抽 Runner** |
| D2 | `stages` 一词多义 | API/文档灾难 | **RFC 即改名 keyframes** |
| D3 | Pose schema v2 已占用（space/unit），但无多帧 | 「Pose v2 stages」说法易误解 | 称 **keyframe 扩展**；文件 version 可升 3 或条目内可选字段 |
| D4 | 落盘默认已是 `semantic_v3`+`rad`，文档/Teach 仍偏 hardware 255 | 多帧若写错空间会静默错姿 | 新关键帧**强制**空间一致 |
| D5 | `PrimitiveResult` 过瘦；status 缺段/进度 | scrub 难验收、编排难读状态 | P2 |
| D6 | catalog ≠ GraspGate(11) ≠ planable(13) ≠ YAML | Framework/模型对接踩坑 | 并行债 §9 |
| D7 | 两套 `HandConfig` 同名 | 新人困惑 | 中期改名，非本需求阻塞 |
| D8 | `control_modes_*.md` 仍写 STREAM「远期」、非目标含时间序列 | 与代码/本需求冲突 | 文档同步 |

---

## 3. 目标架构

### 3.1 原则

1. **一个执行核不变**：仍经 `HandGesturePrimitive` → `GestureExecutor`。
2. **模式 = 控制权不变**：PRIMITIVE / TEACH / STREAM；progress 是 PRIMITIVE 下的**驱动子模式**，不是第四 Authority。
3. **位置插值唯一实现**：新增 `KeyframeRunner`（模块名可调整，职责不可散）。
4. **力控 / progressive / 接触冻结不进 Runner**：继续走现有引擎与状态机。
5. **数据与代码解耦**：自定义多中间态只扩展 Pose JSON + 通用 `NamedPose`；不新增原语类。
6. **量纲**：新建关键帧默认与 Pose Library 一致 — `space=semantic_v3`，`unit=rad`。

### 3.2 分层

```text
HandGesturePrimitive          ← 语义：是否力控、grasp 相位、门控
        │
        ▼
MotionProgram                 ← 纯几何：有序 KeyframeSegment[]
        │
        ▼
KeyframeRunner                ← 锚定、resolve_t、lerp、space 校验
        │
        ▼
GestureExecutor               ← ControlMode、限幅、过流、发布
```

谁提供 `MotionProgram`：

| 来源 | 提供方 | 备注 |
|------|--------|------|
| 自定义示教 | `pose_library` 的 `keyframes[]` | 主路径 |
| 单点旧姿态 | 无 keyframes → 单段 `default` | 兼容 |
| 内置静态 prep（可选） | 原语导出 1 个定位 Segment | P3；close 力控不导出 |
| STREAM | **不**经 MotionProgram | 绝对关节流 |

### 3.3 驱动模式矩阵

| 模式 | Authority | 目标来源 | 适用 |
|------|-----------|----------|------|
| PRIMITIVE + **auto** | 节点 | 关键帧序列 + `elapsed` | 顺序回放、现网 |
| PRIMITIVE + **progress** | 节点 | 当前关键帧段 + 外部 `p∈[0,1]` | 示教 scrub、精细对位 |
| TEACH | GUI | 不写 gesture 轨迹 | 捕获关键帧 |
| STREAM | 外部 | `stream_target` | 模型连续角 |
| 力控 close / progressive | 原语内部 | 电流/触觉等 | **禁止** progress 覆盖 |

**progress ≠ STREAM**：前者是关键帧间归一化插值；后者绕过关键帧直接给关节目标。

---

## 4. 数据模型

### 4.1 Pose 条目（扩展，向后兼容）

文件桶格式保持现有 `{canonical}_{side}.json`；`POSE_SCHEMA_VERSION` 建议升为 **3**（或保持 2 但条目支持可选 `keyframes` — 实现时二选一，**推荐升 3** 便于校验）。

```json
{
  "name": "my_grip",
  "space": "semantic_v3",
  "unit": "rad",
  "duration": 0.5,
  "positions": [ /* 与末关键帧相同，兼容旧工具 */ ],
  "num_joints": 30,
  "description": "",
  "source": "teach",
  "updated_at": "2026-08-28T12:00:00",
  "keyframes": [
    {
      "name": "pre",
      "positions": [ /* len 与 space 匹配 */ ],
      "duration": 0.4
    },
    {
      "name": "pre2",
      "positions": [ /* ... */ ],
      "duration": 0.3
    },
    {
      "name": "grasp",
      "positions": [ /* ... */ ],
      "duration": 0.5
    }
  ]
}
```

#### 字段规则

| 字段 | 规则 |
|------|------|
| `keyframes` | 可选。缺省 / 空 → 行为与今日单点 `NamedPose` 完全一致 |
| `keyframes[].name` | `^[a-z0-9_]+$`；同一 pose 内唯一；禁止与命令保留字冲突（`stop`/`mode`/…） |
| `keyframes[].positions` | 长度与条目 `space`/`num_joints` 一致；**禁止**与条目顶层 `space`/`unit` 不一致 |
| `keyframes[].duration` | ≥ `_MIN_DURATION`（现网 0.2）；auto 模式该段墙钟时长 |
| 顶层 `positions`/`duration` | **必须等于末关键帧**（写入时自动镜像）；供 export / 旧读路径 |
| 混空间 | 同一 pose 内所有关键帧共享顶层 `space`+`unit`；拒绝「一段 rad 一段 hardware」 |

#### 读取兼容

| 落盘 | 行为 |
|------|------|
| 无 `keyframes` | 单段，名默认 `default`（或 `main`），目标=顶层 `positions` |
| `hardware` / `legacy255` 旧条目 | 可读；**新建多关键帧**只允许 `semantic_v3`+`rad`（实现可对旧 hardware 单点仍回放） |
| `semantic_o20` | 读时 `project_o20_to_s30`（与现网一致）；多关键帧新建不推荐 o20 |

### 4.2 运行时结构（建议放 `motion_program.py`）

```python
@dataclass(frozen=True)
class KeyframeSegment:
    name: str
    target: List[float]       # 与 space 一致的向量
    duration: float
    # drive 默认由指令决定，不必落盘


@dataclass
class MotionProgram:
    ref: str                  # 如 "pose/my_grip"
    space: str                # semantic_v3 | hardware | ...
    unit: str                 # rad | legacy255
    segments: List[KeyframeSegment]


class DriveMode(Enum):
    AUTO = "auto"
    PROGRESS = "progress"


def resolve_t(
    *,
    elapsed: float,
    duration: float,
    progress_override: Optional[float],
) -> float:
    """progress_override 优先；否则 elapsed/duration；结果 clamp 到 [0, 1]。"""
    ...
```

### 4.3 KeyframeRunner 契约（核心 API）

建议新文件：`keyframe_runner.py`（或 `motion_program.py` 同文件）。

| 方法 | 行为 |
|------|------|
| `reset(program, start_q, segment_index=0)` | 进入程序；锚定 `q_enter = start_q` |
| `select_segment(name_or_index, current_q)` | **换段重锚** `q_enter = current_q` |
| `set_progress(p: float)` | 仅 `DriveMode.PROGRESS`；**不**重锚；`p` clamp `[0,1]` |
| `set_drive(AUTO\|PROGRESS)` | 切换驱动；切到 AUTO 时可从当前几何进度推 elapsed（可选，须单测） |
| `compute(elapsed) -> List[float]` | `t = resolve_t(...)`；`q = lerp(q_enter, target, t)` |
| `advance_auto(elapsed) -> ...` | AUTO 下段完成则自动下一段并重锚（顺序回放） |

**锚定硬规则**（必须写单测）：

1. 进入新 segment / 新 pose / 新原语 → 用**当前角**重锚。
2. 同一 segment 内只更新 `progress` → **不**重锚（避免 scrub 抖动/漂移）。
3. `pose name` 无 kf 名 → AUTO 从第一段跑到最后；末段 `t≥1` 后 hold。

### 4.4 与 YAML `stages` 的关系

内置 `tripod` 等配置中的：

```yaml
stages:
  prep: 0.5
  close: 0.5
  pause: 0.0
```

表示 **prep/close 相位时长**，**不是** Pose `keyframes`。
文档与代码注释必须写清；解析器命名空间分离：

- `pose <name> <kf> [p]` → Pose keyframes
- `tripod prep [p]` → 抓取相位（P3 才接 Runner）

---

## 5. 指令与 ROS 接口

### 5.1 Phase 1 — 扩展 `gesture_cmd`（兼容）

```text
pose <name>
pose <name> <keyframe>
pose <name> <keyframe> <progress>
pose <name> <keyframe> auto           # 仅 AUTO 该关键帧段，段末 done（不进入 PROGRESS）
pose <name> progress=<float>          # 可选别名：当前/默认段
```

| 形式 | 行为 |
|------|------|
| `pose my_grip` | AUTO，顺序全 keyframe |
| `pose my_grip pre2` | 进入 `pre2` 段，**PROGRESS hold 在 p=0**（不自动跑完该段） |
| `pose my_grip pre2 0.35` | PROGRESS，锚定后停在 0.35 |
| 非法 kf 名 | reject + status；列出可用 kf |

**推荐语义（开发默认）**：

- 带 `<progress>` → `DriveMode.PROGRESS`，该段 hold 在 p。
- 仅 `<keyframe>` 无数字 → 进入该段且 `progress=0` hold（便于示教从段起点微调）；若产品更想「跑完该段」，用 `pose name kf auto` 显式扩展（可选）。
- 无 kf → 全程序 AUTO。
- **同名同段**再次发 `pose <name> <kf> <p>`（或仅改 p）→ **热更新 progress，不重锚**（与 `motion_progress` topic 一致）；换 kf 仍重锚。
- Teach / `save_pose` **无 keyframe 名** 写已有多帧姿态 → **拒绝**（防抹 `keyframes[]`）；请用 `keyframe:=` / `upsert_pose_keyframe`，或 `replace_multi_keyframe=True`。
- `large_wrap close` + 外部 progress：四指/拇指共用同一 `t`，YAML `thumb_delay` 仅墙钟路径生效。

`gesture_node._handle_pose_cmd`：今日忽略多余参数 → 改为解析；非法参数 warn。

内置（P3）：

```text
<prim> prep <progress>     # 仅 PHASED 且该相位为纯定位
<prim> close <progress>    # 若 close 含力控 → 拒绝并英文 warn
```

### 5.2 Phase 2 — 高频通道（已落地）

| Topic | 类型 | 含义 |
|-------|------|------|
| `/{side}_hand/motion_progress` | `std_msgs/Float32` | 更新当前 NamedPose 的段内 p；无 active pose / 非 PRIMITIVE 则忽略 |
| `/{side}_hand/gesture_cmd` | `std_msgs/String` | **仍可用**：`pose name kf p` 切换关键帧或一次性设 p |

建议发布频率：**10～50 Hz**（与默认 `control_rate=10` 对齐或略高）。
手部关节命令下发频率仍由 **`control_rate`（默认 10 Hz）** 决定，与 progress 输入频率解耦。

`gesture_cmd` 保留离散切换（启停、换 keyframe、AUTO 全跑）；progress 热更新优先走
`motion_progress`。**同名同段**再次发 `pose <name> <kf> <p>` 也会走热更新（不重锚），
与 topic 行为对齐。

### 5.3 状态观测（P2，验收必需）

`gesture_status`（或并列字段）建议增加：

```text
drive=progress|auto
motion_ref=pose/my_grip
keyframe=pre2
progress=0.35
```

便于 Teach / 脚本 / Framework 验收 scrub。

### 5.4 Framework / Plan（P4）

- Skills / backend 增加可选字段：`motion_ref` / `keyframe` / `progress`（与 ROS 同 schema）。
- `PlanGesture`：对纯位置 `MotionProgram` 按段采样 `time_s` + `positions`；力控原语仍不可名义规划。
- 自定义 pose 是否 planable：建议 `source=teach` 且全 keyframe 为 `semantic_v3+rad` 时可开（白名单或 flag）。

---

## 6. 模块改造清单（给程序员）

### 6.1 新增

| 文件 | 职责 |
|------|------|
| `motion_program.py` | `KeyframeSegment` / `MotionProgram` / `resolve_t` / 从 `PoseEntry` 构建 |
| `keyframe_runner.py` | 锚定、段切换、auto 前进、progress 热更新、lerp |
| `test/test_keyframe_runner.py` | 锚定、scrub 可逆、段切换、无 keyframes 兼容 |
| `test/test_pose_keyframes.py` | JSON round-trip、镜像末帧、非法空间拒绝 |

### 6.2 修改

| 文件 | 改动要点 |
|------|----------|
| `pose_library.py` | `PoseEntry` 增 `keyframes`；校验；upsert 镜像顶层 positions；schema 策略见 §4.1 |
| `primitives/named_pose.py` | 用 `MotionProgram` + `KeyframeRunner` 替换「仅 StaticPoseEngine 单目标」；单点路径可仍委托 Static 或 Runner 单段 |
| `gesture_node.py` | `_handle_pose_cmd` 解析 kf/progress；可选订阅 `motion_progress` |
| `executor.py` | status 聚合 drive/kf/progress（若从 primitive 查询）；**不**在 executor 内复制 lerp |
| `save_pose.py` / pose services | `save_pose <name> [--keyframe kf] [--duration]`；更新某帧或追加 |
| `teach_panel/` | 存多帧 UX；显示当前 kf；试播支持 progress（可 P1.5） |
| `pose_bundle.py` | bundle 携带 keyframes；合并策略不变 |
| `trajectory_planner.py` | 多段采样（P4） |
| `docs/interfaces.md` 等 | 指令与 status 同步 |

### 6.3 明确不改（本期）

- `SequentialForceCloseEngine` / progressive 冻结逻辑
- GraspGate 集合
- STREAM 路径
- 为每个内置原语重写成 MotionProgram（远景可选）

### 6.4 拒绝矩阵（实现时写死）

| 请求 | 结果 |
|------|------|
| `pose` + progress，但 pose 不存在 | reject |
| keyframe 名不存在 | reject + 列出可用 |
| 同 pose 关键帧 `space` 不一致 | upsert 失败 |
| 新建多关键帧使用 `hardware` | reject 或强警告后拒绝（推荐拒绝） |
| `tripod_by_force close 0.3` 等力控相位 + progress | warn + ignore progress / reject |
| TEACH/STREAM 下 `pose` | 与现网相同：先 `mode primitive` |
| progress ∉ [0,1] | clamp 或 reject（推荐 clamp + debug 日志） |

---

## 7. 兼容性与迁移

| 场景 | 期望 |
|------|------|
| 旧单点 pose，无 keyframes | 字节级行为与现网一致（时长、空间、NamedPose done） |
| 旧 `pose name extra` | 今日忽略 extra → 升级后若 extra 像 kf/progress 则生效；纯垃圾参仍 warn |
| 旧 hardware 单点 | 继续可读回放（NamedPose hardware 整 DOF 路径保留） |
| export / bundle / GUI 预设 | 无 keyframes 时仍只看顶层 positions；有则导出完整列表 |
| 文档示例 | 更新为 `semantic_v3` + `rad`；删除「仅 hardware」过时表述 |

---

## 8. 实施分期（推荐排期）

| 阶段 | 交付 | 验收 |
|------|------|------|
| **RFC** | 本文 + 更新 `control_modes_*.md` 目标/非目标 | Review 通过 |
| **P0** | `resolve_t` + `KeyframeRunner` + **单段** `pose name <p>`（无多帧也可先支持整段 progress） | 单测：scrub 0→1→0.5 几何可逆；不重锚 |
| **P1** | Pose `keyframes[]` + NamedPose 多段 AUTO/PROGRESS + `save_pose --keyframe` | 示教 3 帧 → `pose x` 顺序到位；`pose x pre2 0.4` 停正确姿 |
| **P2** | `gesture_status` 字段 + `motion_progress` topic | 外部 10Hz 改 p 无 parse 卡顿；status 可读 |
| **P3** | 内置纯定位 prep 接 Runner；力控拒绝矩阵 | `thumb_adduction_grip prep 0.3` OK；force close + p 拒绝 |
| **P4** | PlanGesture 多段；Framework 字段 | plan 返回多段 samples |
| **P5** | （可选）pinch/twist 手写多段迁 Runner；能力矩阵单源 | 减重复代码 |

**建议开发顺序**：P0 → P1 → P2；P3/P4 可并行由不同人做，但依赖 Runner API 稳定。

### 8.1 P0 最小接口草图（可直接开写）

```python
# keyframe_runner.py（示意，非最终签名）

def resolve_t(
    elapsed: float,
    duration: float,
    progress_override: float | None,
) -> float:
    if progress_override is not None:
        return max(0.0, min(1.0, float(progress_override)))
    if duration <= 0.0:
        return 1.0
    return max(0.0, min(1.0, elapsed / duration))


class KeyframeRunner:
    def __init__(self) -> None: ...
    def reset(self, program: MotionProgram, start_q: list[float]) -> None: ...
    def select_segment(self, name: str, current_q: list[float]) -> None: ...
    def set_drive_progress(self, p: float) -> None: ...
    def set_drive_auto(self) -> None: ...
    def compute(self, elapsed: float) -> list[float]: ...
    @property
    def keyframe(self) -> str: ...
    @property
    def progress(self) -> float: ...
    @property
    def drive_mode(self) -> str: ...
```

`NamedPose.compute`：把 Runner 输出填进 `PrimitiveResult.target_angles`，`space` 与 program 一致（semantic_rad 路径优先，与现 `NamedPose` 分支对齐）。

---

## 9. 架构层还需完善的优化（本需求之外）

以下不阻塞 P0，但影响长期可维护性；建议列入包维护 backlog：

| 优先级 | 项 | 说明 |
|--------|-----|------|
| 高 | 更新 `control_modes_and_custom_pose.md` | STREAM 已落地；多关键帧改为目标；示例改 `semantic_v3`+`rad` |
| 高 | 能力矩阵单源 | `primitive_catalog` ↔ `GraspGate` ↔ `planable` ↔ YAML allowlist 校验测试 |
| 中 | status / `PrimitiveResult` 扩展 | `grasp_state`、motion 字段统一出口 |
| 中 | Teach UI 量纲与库默认对齐 | 避免 255 UI vs rad 落盘认知分裂 |
| 中 | 文档计数与 `primitives.md` 对齐 | flick/twist/named_pose/gated 口径 |
| 低 | 重名 `HandConfig` | `SemanticJointIndex` vs `HardwareHandConfig` |
| 低 | 静态内置 YAML 显式 keyframes | 减少 builder 与数据双轨（远景） |
| 低 | Launch：`TEACH_PANEL` 缺依赖勿拖垮整机 | fastapi 缺失时降级而非 shutdown_on_exit |

### 9.1 对「自定义多中间态 + 进度细控」方案本身的架构优化点

在定稿本方案时，相对「只改 NamedPose + PoseEntry.stages」草稿，**必须保留**的优化：

1. **抽 KeyframeRunner**，禁止三处复制 `resolve_t`。
2. **字段名用 `keyframes`**，避开 YAML `stages`。
3. **新建多帧锁定 `semantic_v3`+`rad`**，与现网默认落盘一致（勿再规定 hardware 255 为新标准）。
4. **progress 与力控拒绝矩阵**写进实现，而非文档愿望。
5. **观测（status）与 P1 同周期或紧随**，否则示教无法验收。
6. **高频 progress 预留 topic**，String 仅作兼容入口。

---

## 10. 测试计划

### 10.1 单测（无 ROS）

- `resolve_t`：override 优先；duration≤0；clamp。
- Runner：同段 scrub 不重锚；换段重锚；AUTO 跨段；单段无 keyframes 兼容。
- Pose IO：keyframes round-trip；末帧镜像；非法名/长度/空间。
- 拒绝矩阵：力控相位 + progress（P3）。

### 10.2 集成 / 真机

- MOCK：`pose` 三帧顺序；`pose x kf p` 与 FK/可视化目视。
- 回归：`run_gesture_cmd_suite` 旧用例仍过；新增 keyframe 小组。
- Teach：存两帧 → 回放 → scrub。
- 模式：TEACH 下 pose 仍拒绝。

### 10.3 非功能

- progress 热更新不分配新 Engine（无周期性 reset）。
- 过流 / 限幅仍只经 Executor。
- 10Hz `motion_progress` 下 CPU 与日志可控。

---

## 11. 风险与缓解

| 风险 | 缓解 |
|------|------|
| 与 YAML `stages` 混淆 | 强制 `keyframes` 命名 + 文档术语表 |
| progress 误用于力控 | 拒绝矩阵 + 单测 |
| 量纲混拼关键帧 | upsert 校验同一 space/unit |
| 只改 NamedPose 继续分叉 | Code review 门禁：新插值必须走 Runner |
| 文档/代码再漂移 | P0 合并时同步 `interfaces.md` / control_modes |
| Teach 255 vs rad | P1 存帧路径统一走现网默认 save space |

---

## 12. 决策记录（ADR 摘要）

| 决策 | 选择 | 理由 |
|------|------|------|
| 是否第四 ControlMode | 否；progress 为驱动子模式 | 控制权模型保持简单 |
| 关键帧字段名 | `keyframes` | 避开 YAML `stages` |
| 新多帧落盘空间 | `semantic_v3` + `rad` | 与 `DEFAULT_SAVE_SPACE` 一致 |
| 插值归属 | `KeyframeRunner` | 唯一 `resolve_t` / 锚定 |
| 力控 + progress | 拒绝 | 语义冲突 |
| 全局 progress | 本期不做 | 与段内 p 混淆 |
| STREAM | 保持独立 | 绝对流 vs 关键帧插值 |

---

## 13. 开发入口检查清单

开始写代码前确认：

- [ ] 已读本文 §2.4 术语表与 §6.4 拒绝矩阵
- [ ] 已读 `units_radians_migration.md` 与 Pose 默认 space
- [ ] P0 仅引入 Runner + 单段 progress，不一次改 Teach 全 UX
- [ ] 单测先于真机
- [ ] 不修改 force close / progressive 核心路径
- [ ] 同步更新 `interfaces.md` 指令表

联系/评审：以本文件为唯一开发契约；若实现偏离，先更新本文再合入。

# 手势原语库

本文档介绍预定义手势原语的分类、行为、参数与使用方式。启动时会递归扫描
`primitives/` 下任意公共包层级中定义的具体 `HandGesturePrimitive` 子类并构建
`PRIMITIVE_REGISTRY`。任意层级以 `_` 开头的私有包或模块不会被导入。
手型 allowlist / 能力见 `config/{o20,o6,l6,l25}.yaml`；ROS 接口见 [`interfaces.md`](interfaces.md)。

> **量纲**：有 `robot_description` 时，执行核内部为 **semantic_rad**（S30 语义布局 + rad）；
> YAML / 代码中的 0～255 为标定源或硬件边界，经 Converter / ``AngleUnits`` 换算。
> 位置和量纲说明见[工作台数据约定](../../docs/workbench.md#数据与公共接口)。
> 下文关节表中的 0～255 描述的是 **legacy 标定语义**，不是执行核运行时量纲。

---

## 手型与力反馈

力控类原语统一：**触觉优先**，其次按手型使用电流或力矩；无反馈时依赖位置到达 / 闭合上限。

| 手型 | 反馈通道 | 能力 token | YAML 接触量纲 |
|------|----------|------------|---------------|
| O20 / L25 | 关节电流 (mA) | `current` | `contact_delta` / `hold_safe_current` |
| O6 | 关节力矩 (0~100%) | `motor_torque` | `contact_delta_torque_pct` / `hold_safe_torque_pct` |
| L6 | 触觉（电流占位不可用） | `touch`（`current=false`） | 配置可保留电流键，运行时以触觉停指 |

`primitive_catalog` 中 `current` ↔ `motor_torque` ↔ `touch` 可互为力控能力替代，避免 L6 因缺电流被误拒。

O6/L6 无独立 tip/rot（索引 `-1`）。legacy 闭合路径须用 `set_joint` / `tip_or_base` / `present_joints`（`primitive_base.py`），禁止直接下标写入。

---

## 原语总览

| # | 指令名 | 类别 | 参与手指 | 感知输入 | 力控 | 分阶段 |
|---|--------|------|---------|---------|------|--------|
| 1 | `init` | 安全 | 全部 | — | — | — |
| 2 | `open` | 安全 | 全部 | — | — | — |
| 3 | `release` | 安全 | 全部 | — | — | — |
| 4 | `relax_grip` | 安全 | 全部 | — | — | — |
| 5 | `fist` | 固定 | 全部 | — | — | — |
| 6 | `pinch` | 固定 | 拇+食 | — | — | — |
| 7 | `point` | 固定 | 全部 | — | — | — |
| 8 | `ok_sign` | 固定 | 全部 | — | — | — |
| 9 | `v_sign` | 固定 | 全部 | — | — | — |
| 10 | `index_ring` | 自适应 | 拇+食 | object_size | 触觉/电流·力矩 | — |
| 11 | `large_wrap` | O6/L6 固定包络 | 全部 | — | — | prep/close/auto |
| 12 | `thumb_adduction_grip` | 侧向夹持 | 拇+(食中无小) | object_size | 触觉/电流·力矩 | prep/close |
| 13 | `index_middle_adduction_grip` | 侧向夹持 | 拇+食+中 | object_size | 触觉/电流·力矩 | prep/close |
| 14 | `middle_ring` | 力控包络 | 拇+中 | object_size | 触觉/电流·力矩 | — |
| 15 | `ring` | 力控包络 | 拇+食+中 | object_size | 触觉/电流·力矩 | — |
| 16 | `small_warp` | 力控包络 | 拇+食+中+无+小 | object_size | 触觉/电流·力矩 | — |
| 17 | `no_index_warp` | 力控包络 | 拇+中+无+小 | object_size | 触觉/电流·力矩 | — |
| 18 | `hook` | 力控包络 | 食+中+无+小 | object_size | 触觉/电流·力矩 | — |
| 19 | `tripod` | 固定 / 两阶段 | 拇+食+中 | — | — | auto（默认）/prep/close/full |
| 20 | `palmar` | 力控包络 | 全部（掌心支撑） | object_size | 触觉/电流·力矩 | — |
| 21 | `index_pinch` | 定位捏取 | 拇+食 | tcp_pose / object_pose | — | — |
| 22 | `middle_pinch` | 定位捏取 | 拇+中（食避让） | tcp_pose / object_pose | — | — |
| 23 | `parallel_extension` | 定位包络 | 拇 vs 食+中+无+小 | tcp_pose / object_pose | — | — |
| 24 | `disk` | 定位包络 | 全部 C 形 | tcp_pose / object_pose | — | 内部 P1–P3 |
| 25 | `hammer_grip` | O6 固定握持 | 全部 | — | 逐关节限速 | prep/close |
| 26 | `tripod_by_force` | 三指接触停指捏取 | 拇+食+中 | 电流/触觉（按手型） | 逐指冻结 | auto（默认）/prep/close/full |

> **GraspGate 门控（11 个）**：`ring_by_vision` `small_warp_by_vision` `no_index_warp_by_vision` `middle_ring_by_vision` `hook_by_vision` `tripod_by_vision` `palmar_by_vision` `parallel_extension_by_vision` `index_pinch_by_vision` `middle_pinch_by_vision` `disk_by_vision` — 须经 H/XY/Fwd 三判定后才转发至 Executor。
> **Executor 可达性**：除 4 个安全原语外，所有原语激活时检查 TCP–物体距离（默认 15 cm）；定位类原语在运行中还会根据 `object_pose` 做掌心前方 2–15 cm 保持判定。

---

## 类别说明

### 安全类（4 个）

不受可达性检查限制，任何时刻均可调用。用于复位、释放、紧急保护。

| 指令 | 行为 | 过渡时间 |
|------|------|---------|
| `init` | 半张开自然姿态（所有原语 fallback 目标） | 0.8s |
| `open` | 五指完全伸直 | 0.6s |
| `release` | 从当前位置相对张开 | 0.6s |
| `relax_grip` | 先放松再回中（重置抓取状态） | 0.6s |

### 固定类（6 个）

预设目标角度 + 平滑插值（lerp），不依赖感知输入，不经 GraspGate。

| 指令 | 手形描述 | 运动特征 |
|------|---------|---------|
| `fist` | 五指完全握紧 | 4 阶段避免拇指碰撞：四指闭 → 拇指侧摆 → 旋转 → 下压 |
| `pinch` | 拇食对捏，其余张开 | 单阶段 lerp |
| `point` | 食指伸出，其余握拢 | 单阶段 lerp |
| `ok_sign` | 拇食成圆，其余伸直 | 单阶段 lerp |
| `v_sign` | 食中伸开，其余握拢 | 单阶段 lerp |
| `tripod` | 拇指与食指、中指形成三角支撑 | 全手型 `prep`/`close`；`auto`/`full` 见下 |

#### tripod 两阶段

所有手型都支持 `tripod prep` / `tripod close`（时长见各手 `gestures.tripod.stages` /
`tripod_two_stage`）。

- **auto**（**默认**，裸发 `tripod` / `tripod auto` / `tripod full` 同效）：自动串联
  prep → 停顿 → close。L6 为两阶段（停顿见 `pause_duration`，默认 1s）。
  O20/L25/G20/O6 的 `tripod` 仍走既有一次到位引擎；`tripod_by_force` 的 auto
  亦拆成 prep→close。
- **prep**：侧摆/旋转等到 tripod 瞄准位，拇/食/中屈曲保持 `init` 预成型，不捏合；
  到位后 `grasp_state=ready`，**等待** `tripod close`。
- **close**：只闭合拇/食/中屈曲，其余关节保持当前值。

L6 prep 额外把拇指屈曲强制完全张开、四指进 init，适配欠驱动行程。

#### O6/L6 large_wrap 两阶段

- `large_wrap prep`：用 0.5s 进入准备位；拇指完全张开（屈曲 255）且侧摆为 0，
  食指、中指、无名指和小指沿用 `init` 姿态。
- `large_wrap close`：从当前姿态用 0.5s 闭合四指，拇指延迟 0.2s 后开始闭合；
  五个屈曲关节的硬件目标均为 `0`（完全弯曲），拇指侧摆保持当前值。
- `large_wrap auto`（以及兼容的无参数 `large_wrap`）：先完成 prep，保持准备位
  1.0s，再自动执行 close。

#### tripod_by_force

动作时序复用同手型 `tripod`：**默认 auto**（裸发同效）；仅 `prep` 等待 close。
O20/L25/G20/O6 的 `tripod_by_force auto` 走 prep→close，力控只在 close 采集基线并逐指冻结。

接触策略在 `tripod_force.py`，由各手 `gestures.tripod_by_force.contact` 配置：

| 手型 | `feedback_source` | 默认阈值 | 无反馈时 |
|------|-------------------|----------|----------|
| L6 | `motor_current` | CAN `0x36` 增量 30 | 等待，不降级定时闭合 |
| O20 / L25 / G20 | `auto`（触觉优先，否则电流 Δ mA） | `contact_thresholds.current_delta` | 继续位置闭合，有反馈再冻 |
| O6 | `tactile` | 压感阈值 | 无压感则不冻（不用空载力矩） |

新手型：allowlist 加入 `tripod_by_force`，并按上表补 `contact` 段即可，不必改原语。

```yaml
tripod_by_force:
  contact:
    fingers: [thumb, index, middle]
    feedback_source: auto          # auto | motor_current | current | torque | tactile
    feedback_delta: 270            # 省略则用手型默认
    require_feedback: false
```

### O6 两阶段握锤

`hammer_grip` 仅加入 O6 allowlist，使用 O6 原生 6 关节硬件空间参数。
`prep` 使用 O6 默认速度 200 到初始化位姿；`close` 会先确认初始化位姿，
未到位时自动补完 prep，连续 3 帧到位后从实际反馈位置开始闭合，并使用
下列逐关节速度。等待 2 秒仍未到位时保持初始化位姿，不强行闭合。速度值
由 10 Hz Executor 解释为每个关节每秒最多变化的 0–255 位置单位。

```yaml
initial: [100.0, 50.0, 50.0, 50.0, 50.0, 50.0]
target:  [23.53, 96.08, 9.80, 11.76, 15.69, 11.76]
speeds:  [50, 50, 50, 100, 100, 50]
```

```bash
# 分阶段命令
ros2 topic pub --once /left_hand/gesture_cmd std_msgs/String \
  "data: 'hammer_grip prep'"
ros2 topic pub --once /left_hand/gesture_cmd std_msgs/String \
  "data: 'hammer_grip close'"
```

### 感知自适应类（2 个）

根据 `PrimitiveContext.object_size` 实时调整闭合量，支持触觉或**关节电流/力矩**闭环（按手型；无传感器时自动 fallback）。

| 指令 | 手形 | 自适应逻辑 | 闭环策略 |
|------|------|-----------|---------|
| `index_ring` | 拇+食环形包络 | 物体截面直径 → 调拇/食弯曲量 | progressive + 触觉/电流·力矩冻结 |
| `large_wrap_by_vision` | 五指包络 | 物体最大截面 → 调五指弯曲 | progressive + 触觉/电流·力矩冻结 |

**三阶段执行流程：**

```
P1 (0.4s): 非参与手指先张开
P2 (0.5s): 参与手指闭合到视觉目标
P3 (持续): 无接触 → 渐进闭合 (25–30/s)；接触后冻结（触觉或电流/力矩增量，阈值见手型 YAML / `contact_thresholds.yaml`）
```

**接触阈值**：可在手型 `config/*.yaml` 的 `gestures.*.contact`、或 `config/contact_thresholds.yaml` / `gesture_node` ROS 参数中调节。O20 默认电流增量约 **270 mA**；O6 用 `contact_delta_torque_pct`；L6 以触觉为主。
**尺寸自适应**：物体尺寸变化超过 2mm 时触发目标更新，经 250ms blend 平滑过渡。

### 侧向夹持类（2 个）

拇指或食中指侧面并拢夹持，**支持 prep/close 两阶段控制**（`PHASED_GRASP_PRIMITIVES`）。

| 指令 | 夹持方式 | 典型场景 |
|------|---------|---------|
| `thumb_adduction_grip` | 拇指侧面下压贴紧食指侧面 | 卡片、钥匙、薄片 |
| `index_middle_adduction_grip` | 食指与中指侧面并拢 | 笔杆、香烟、细棒 |

**分阶段命令格式：**

```bash
# 完整执行（默认 full）
ros2 topic pub --once /left_hand/gesture_cmd std_msgs/String "data: 'thumb_adduction_grip'"

# 分步执行
ros2 topic pub --once /left_hand/gesture_cmd std_msgs/String "data: 'thumb_adduction_grip prep'"
# ... 机械臂就位 ...
ros2 topic pub --once /left_hand/gesture_cmd std_msgs/String "data: 'thumb_adduction_grip close'"
```

**`thumb_adduction_grip` 执行流程（full 模式）：**

```
P1 (0.35s): 四指弯曲收拢形成承接面
P2 (0.40s): 拇指侧摆外展 (abd)
P3 (0.35s): 拇指旋转就位 (rot)
P4 (0.65s): 拇指根部+指尖下压闭合
P5 (持续):  渐进夹紧 → 触觉冻结
```

`prep` 在 P3 结束后暂停（`grasp_state = "ready"`），等待 `close` 命令继续 P4+。

### 力控包络类（7 个）

基于 `object_size`（及可选 mesh 预选型）预成型，逐指接触检测后停止（触觉 / 电流 / 力矩，见上文「手型与力反馈」）。O20 路径常经 GraspGate；O6/L6 多用 `sequential_force_close` 引擎（MCP 力控）。

| 指令 | 对抗手指 | 适用场景 |
|------|---------|---------|
| `middle_ring` | 拇 vs 中 | 中等物体对捏 |
| `ring` | 拇 vs 食+中 | 柱形/瓶状物体 |
| `small_warp` | 拇 vs 食+中+无+小 | 小型多面体全手抓取 |
| `no_index_warp` | 拇 vs 中+无+小 | 食指需保持伸出的场景 |
| `hook` | 无拇指，食+中+无+小 | 把手/挂钩/环形物体 |
| `palmar` | 掌心 vs 食+中+无+小 | 强力掌心包络、球体/多面体 |

**统一两阶段算法（middle_ring / ring / small_warp / no_index_warp / hook / palmar）：**

```
Phase 1 — 预成型 (0.5s):
  根据 object_size（或 mesh 宽度）选择 thumb_rot 等预备角（无 tip/rot 的手型跳过）
  快速 lerp 到预备姿态（手指半张开）

Phase 2 — 力控闭合 (持续):
  各手指独立以固定速率闭合
  触觉优先；否则监测电流/力矩增量超阈 → 该指停止
  全部参与指停止 → grasp_state = "contact"
  持握安全上限: O20 `hold_safe_current`（默认 800 mA）；O6 `hold_safe_torque_pct`
```

运行中需 `tcp_pose`；有 `object_pose` 时物体须在掌心前方 2–15 cm，否则 `_hold()` 保持。
### 定位捏取 / 定位包络类（4 个）

固定目标姿态 + lerp，**不经 object_size 自适应、不用电流力控**，但须经 **GraspGate** 并在运行中校验 TCP/物体相对位姿。

| 指令 | 手形 | 典型场景 | 过渡 |
|------|------|---------|------|
| `index_pinch` | 拇+食指尖对捏，其余伸直 | 螺丝、针、薄片 | 0.5s lerp |
| `middle_pinch` | 拇+中对捏，食指外展避让 | 需保持食指自由的捏取 | 0.5s lerp |
| `parallel_extension` | 五指伸直，拇 vs 四指平行夹持 | 书本、卡片、平板 | 0.6s lerp |
| `disk` | 四指 C 弧 + 拇指侧摆合拢 | 盘、碟、扁圆物体 | 内部三阶段 1.4s |

**`disk` 内部三阶段（非 prep/close 命令）：**

```
P1 (0.4s): 拇指 abd 侧摆至中指对齐位
P2 (0.5s): 四指弯成 C 弧，拇指 flex 关节保持
P3 (0.5s): 拇指 base/rot/tip 合拢握住
```

**运行约束（与力控包络类相同）：**

- 无 `tcp_pose` → 保持当前姿态
- 有 `object_pose` → 掌心前方 2–15 cm，否则 hold
- 无 `object_pose` 时跳过前方距离检查（GraspGate 已做 H/XY/Fwd）

---

## auto 命令映射

发送 `auto` 时，根据感知输出的 `grasp_type` 自动选择原语（`grasp_mapping.py`）：

```
感知 grasp_type    →    原语
─────────────────────────────
precision          →    index_ring
lateral            →    index_middle_adduction_grip
power              →    large_wrap
```

无 `grasp_type` 或未知类型时不执行任何动作。新增定位/力控原语暂未纳入 auto 映射。

---

## 原语生命周期

```
cmd "pinch" 到达
       │
       ▼
┌─ set_primitive() ─────────────────────────────┐
│  1. 可达性检查 (非安全原语)                      │
│  2. 旧原语 on_exit()                           │
│  3. 新原语 on_enter(current_angles)            │
│  4. 记录 start_time                           │
└───────────────────────────────────────────────┘
       │
       ▼ (每 100ms)
┌─ _tick() ─────────────────────────────────────┐
│  ctx = PrimitiveContext(tcp, bbox, current...) │
│  result = primitive.compute(angles, elapsed, ctx)│
│  if result.feasible:                           │
│      target = result.target_angles             │
│  else:                                         │
│      target = last_target  (hold)              │
│  publish(target)                               │
└───────────────────────────────────────────────┘
       │
       ▼ (新命令到达 或 primitive.done)
  on_exit() → 切换到下一个原语
```

---

## grasp_state 状态机

力控和自适应原语通过 `grasp_state` 属性暴露内部进度：

| 状态 | 含义 | 后续行为 |
|------|------|---------|
| `approaching` | 正在向目标位置运动 | 继续 lerp |
| `ready` | prep 阶段完成，等待 close | 保持当前姿态 |
| `progressive` | 已到视觉目标，渐进闭合中 | 缓慢收紧 |
| `contact` | 检测到接触，冻结 | 保持直到新指令 |
| `closing` | 力控闭合中 | 监测电流 |

Executor 在状态变化时打印日志，方便调试。

---

## 添加新原语

1. 创建 `primitives/my_gesture.py`（输出 **semantic_30**，有 Converter 时量为
   `semantic_rad`；见[工作台数据约定](../../docs/workbench.md#数据与公共接口)）：

```python
from ..primitive_base import HandGesturePrimitive, PrimitiveContext, PrimitiveResult, lerp_angles

class MyGesture(HandGesturePrimitive):
    @property
    def name(self) -> str:
        return "my_gesture"

    def on_enter(self, current_angles):
        super().on_enter(current_angles)

    def compute(self, current_angles, elapsed, ctx) -> PrimitiveResult:
        target = [...]  # S30，长度 30
        t = min(1.0, elapsed / 0.5)
        angles = lerp_angles(self._start_angles, target, t)
        return self._move(angles)
```

2. 无需修改 `primitives/__init__.py`（自动 discovery）。撞名 / 导入失败会在
   启动期明确报错。

3. **必须**在 `primitive_catalog.py` 增加 `PrimitiveCatalogEntry`
   （`requires` / `category`；若需 GraspGate 设 `gated=True`；若
   `gesture_cmd` 支持 prep/close 设 `command_phased=True`）。
   **不要**再手改 `grasp_gate.GATED_PRIMITIVES`——门控集由 catalog 派生。

4. 各支持该原语的手型：`config/{hand}.yaml` 的 `primitives.supported` +
   必要时 `gestures.` 段。

5. 若需 PlanGesture 名义规划：加入 `planable_primitives.py`
   （仅纯静态、无接触闭环）。

6. 测试：

```bash
# 能力矩阵一致性（catalog / gate / phased / planable）
python3 -m pytest test/test_capability_matrix_consistency.py -q

ros2 topic pub --once /left_hand/gesture_cmd std_msgs/String "data: 'my_gesture'"
```

---

## 关节布局参考（O20 20-DOF）

| 索引 | 名称 | 含义 | 值域说明 |
|------|------|------|---------|
| 0 | thumb_base | 拇指根部弯曲 | 0=伸直, 255=弯曲120° |
| 1 | index_base | 食指根部弯曲 | 0=伸直, 255=弯曲180° |
| 2 | middle_base | 中指根部弯曲 | 0=伸直, 255=弯曲180° |
| 3 | ring_base | 无名指根部弯曲 | 0=伸直, 255=弯曲180° |
| 4 | pinky_base | 小指根部弯曲 | 0=伸直, 255=弯曲180° |
| 5 | thumb_abd | 拇指侧摆 | 0=0°, 255=180° |
| 6 | index_abd | 食指侧摆 | 128=中立, 0/-30°, 255/+30° |
| 7 | middle_abd | 中指侧摆 | 128=中立 |
| 8 | ring_abd | 无名指侧摆 | 128=中立 |
| 9 | pinky_abd | 小指侧摆 | 128=中立 |
| 10 | thumb_rot | 拇指旋转 | 0=0°, 255=130° |
| 11–14 | reserved | 预留位 | 恒为 0 |
| 15 | thumb_tip | 拇指指尖弯曲 | 0=伸直, 255=弯曲150° |
| 16 | index_tip | 食指指尖弯曲 | 0=伸直, 255=弯曲180° |
| 17 | middle_tip | 中指指尖弯曲 | 0=伸直, 255=弯曲180° |
| 18 | ring_tip | 无名指指尖弯曲 | 0=伸直, 255=弯曲180° |
| 19 | pinky_tip | 小指指尖弯曲 | 0=伸直, 255=弯曲180° |

> L25 (25-DOF) 新增 thumb_root2[15]、middle_root2[17]、thumb_tip→[20]、各指 tip→[21–24]，且角度方向反转（0=弯曲, 255=伸直）。详见 `primitive_base.py` 中 `HAND_CONFIGS["l25"]`。

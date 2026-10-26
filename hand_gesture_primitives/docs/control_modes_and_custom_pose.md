# 控制模式与自定义手势 — 可落地方案

> **luban 适配说明**：公开 API 为侧化 topic `/{side}_hand/gesture_cmd|status|result|stream_target`；
> 默认 `cmd_backend=ros2_control`（`/{side}_hand_controller/commands`）。
> 下文若出现全局 `/hand_gesture_cmd*` 或仅 `/cb_*`，请按侧化 + ros2_control 理解。

本文档给出 **多模式控制架构** 与 **示教自定义姿态（Pose Library）** 的可落地指导，与现有原语执行栈对齐，并为未来「模型输出手势」预留扩展点。

---

## 1. 目标与非目标

### 目标

| 能力 | 说明 |
|------|------|
| 多模式 | 统一控制权：`PRIMITIVE` / `TEACH` / `STREAM`（STREAM 已落地） |
| 示教入库 | GUI/CLI 调关节 → 捕获终态 → 按手型落盘 → `pose <name>` 回放 |
| 执行一致 | 回放经 Executor 映射/过流保护（单终态见下文；多关键帧见扩展文档） |
| 模型预留 | 离散原语名或连续关节流，不另建执行器 |

**扩展（多中间态 + 段内 progress）**：见
[`motion_keyframes_and_progress.md`](motion_keyframes_and_progress.md)
（`keyframes[]` + `KeyframeRunner`；本文仍描述单终态 Pose Library 基线）。

### 非目标（本期不做）

- 任意轨迹文件 / 宏录制回放（与「有序示教关键帧」不同；见扩展文档边界）
- 自定义力控、by_vision、PhasedLerp 可视化编辑
- 跨手型自动重映射同一组裸关节值
- 每自定义姿态单独新增 Python 原语类
- 将 STREAM 与关键帧 progress 合并为同一模式

---

## 2. 设计原则

1. **一个执行核**：`HandGesturePrimitive.compute()` → semantic 目标 → `GestureExecutor`（限幅 / 映射 / 过流 / 发布）。
2. **模式 = 控制权**，不是三套系统：谁允许写 `/cb_*_hand_control_cmd`，目标从哪来。
3. **示教产出回灌库模式**：TEACH 只负责让权与落盘；回放仍是 PRIMITIVE 下的 `NamedPose`。
4. **YAML/JSON-first**：姿态数据与代码解耦；通用 `named_pose` 一个入口。
5. **按规范手型分桶**：`resolve_hand_config_key()`（如 L20→l25）决定库文件键。

---

## 3. 总体架构

```
/hand_gesture_cmd ──► GraspGate(可选) ──► /hand_gesture_cmd_exec
                                              │
                                              ▼
                                        GestureNode
                                              │
                    ┌─────────────────────────┼─────────────────────────┐
                    ▼                         ▼                         ▼
             mode=primitive             mode=teach                mode=stream
             (现网+自定义pose)           (节点让权)                 (远期)
                    │                         │                         │
                    │  Registry / PoseLibrary │  停发 cmd                │  外部目标流
                    │  Primitive.compute      │  GUI/人手调关节           │  Watchdog
                    └─────────────────────────┴─────────────────────────┘
                                              │
                                              ▼
                                    GestureExecutor
                                      · semantic → hardware
                                      · 过流保护 → init
                                      · 10Hz publish（受 mode 门控）
                                              │
                                              ▼
                                 /cb_{side}_hand_control_cmd
```

### 三模式对照

| 模式 | 控制权 | 目标来源 | 节点是否发布 cmd | 用途 |
|------|--------|----------|------------------|------|
| **PRIMITIVE** | 节点 | 内置原语 / `pose <name>` | 是 | 现网抓取、示教回放 |
| **TEACH** | GUI / 人手 | 滑条或手掰；可写 PoseLibrary | **否** | 示教入库 |
| **STREAM** | 模型 / 外部 | 连续角或原语序列 | 是（守门后） | 远期模型输出 |

硬规则：任意时刻仅一种 Authority；切换需显式指令。

---

## 4. 模式细则

### 4.1 PRIMITIVE（已有 + 扩展）

- 现有：`open` / `ring_by_vision` / flick / twist 等（见 `config/*.yaml` allowlist）。
- 新增：`pose <name>` → `NamedPose`，从 PoseLibrary 加载终态，经 `StaticPoseEngine` 插值。
- GraspGate：`pose` **不进** `GATED_PRIMITIVES`，直通 `_exec`。
- 可达性：`NamedPose` 视为安全类，**跳过** reach 检查（或加入 `_SAFE_PRIMITIVES`）。

### 4.2 TEACH（本期必须做）

**问题**：当前 `stop` 后 Executor 仍每周期发布 `_last_target`，GUI 会被盖住。

**要求**：

```
mode teach     → 停止 _publish_cmd（保留 state/info 订阅）
mode primitive → 恢复发布；可选 hold 当前角为 _last_target
```

示教流程：

```
1. mode teach
2. GUI 调关节（或手掰；L 系 teach 力控仅辅助）
3. save_pose <name> [duration]
4. mode primitive
5. pose <name>     # 回放验证
```

### 4.3 STREAM（已落地）

模型输出分档：

| 档位 | 输出 | 接入 | 优先级 |
|------|------|------|--------|
| L1 离散 | 原语名 / `pose <name>` / grasp_type | `gesture_cmd` | 现网 |
| L2 连续 | semantic / hardware 关节角 @N Hz | `/{side}_hand/stream_target` + `mode stream` | **已落地** |
| L3 多关键帧名义轨迹 | 示教 keyframes / Plan 采样 | 见 `motion_keyframes_and_progress.md` | 规划中 |

STREAM 守门：限幅、频率上限、断流 watchdog → hold；过流保护始终生效。
**注意**：STREAM（绝对关节流）≠ 关键帧 `progress`（段内插值）；二者勿合并实现。

---

## 5. Pose Library（自定义姿态）

### 5.1 存储

0.2.0 起必须用 **`HAND_GESTURE_POSE_DIR`** 显式指定根目录，不再回退到 HOME（Docker `docker_run.sh` 默认 `/evo/config/hand_gesture_poses`）。

侧化文件（推荐）：

- `{canonical}_{side}.json`，例：`g20_right.json`、`l25_left.json`
- 旧版无侧 `{canonical}.json` 仍可读（list/load 回退；首次按侧写入会种子复制）

例：用户配置 `L20` → 规范键 `l25` → `l25_left.json`。

```json
{
  "version": 1,
  "hand_joint": "L25",
  "canonical_hand_type": "l25",
  "hand_side": "left",
  "poses": {
    "my_pinch": {
      "space": "hardware",
      "positions": [/* len == num_joints, 0..255 */],
      "num_joints": 25,
      "duration": 0.6,
      "description": "",
      "source": "teach",
      "updated_at": "2026-07-30T12:00:00"
    }
  }
}
```

JSON 损坏：隔离为 `*.corrupt.<timestamp>`，**禁止**把坏文件当空库再 upsert 覆盖。
借鉴：`linkerhand-ros2-sdk/gui_control/.../preset_store.py`。**不与** `~/.config/gui_control/presets/` 双向同步。

### 5.2 校验

- `len(positions) == HandConfig(canonical).num_joints`
- 每维 ∈ [0, 255]；`space` 仅允许 `hardware`
- 名称：`^[a-z0-9_]+$`（因 `gesture_node` 会 `.lower()`）
- **保留字禁止**：`stop`, `mode`, `pose`, `pose_list`, `save_pose`, `auto`, `target`, `target_id`, `teach`, `stream`, `primitive`, `named_pose`，以及 `PRIMITIVE_REGISTRY` 全部键名（见 `reserved_pose_names()`）

### 5.3 坐标系

| 阶段 | 约定 |
|------|------|
| 捕获 | 读 `/cb_{side}_hand_state.position`（硬件量纲） |
| 落盘 | `space: hardware` |
| 回放 | `HandConfig.from_hardware` → O20 semantic → `StaticPoseEngine` → `to_hardware` 发布 |

禁止把 O6/L6/L25 硬件向量直接当 semantic 写入。

---

## 6. 模块与代码触点

| 模块 | 职责 | 路径（建议） |
|------|------|----------------|
| `ControlMode` | 枚举 + Executor 发布门控 | `executor.py` |
| `pose_library.py` | CRUD / 校验 / 按手型 IO | 新建 |
| `NamedPose` | 通用静态回放原语 | `primitives/named_pose.py` |
| 指令扩展 | `mode` / `pose` / `pose_list` / `save_pose`(可选) | `gesture_node.py` |
| Catalog | `named_pose`: category=safe/gesture, requires=() | `primitive_catalog.py` |
| Allowlist | 各手型 YAML 增加 `named_pose` | `config/{o20,o6,l6,l25}.yaml` |
| CLI | `save_pose` 订阅一手 state 落盘 | `scripts/save_pose.py` |
| UI（二期） | **本包独立 Teach Web 面板（方案 A）** | 见 `teach_panel_architecture.md` |

### 6.1 指令约定

| 指令 | 模式 | 行为 |
|------|------|------|
| `mode teach` | → TEACH | 停发 cmd |
| `mode primitive` | → PRIMITIVE | 恢复发 cmd |
| `mode stream` | → STREAM | 远期；未实现则拒绝 |
| `pose <name>` | PRIMITIVE | 回放用户姿态 |
| `pose_list` | 任意 | 列出当前手型姿态名（`POSES:…` + `pub_gesture_cmd` stdout） |
| `stop` | 任意 | 停活跃原语；**不**自动退出 TEACH |
| 内置原语名 | PRIMITIVE | 现有逻辑 |

解析注意：整句会 `.lower().split()`；`pose` 必须在查 `PRIMITIVE_REGISTRY` **之前**特判。

### 6.2 Executor 改动要点

```text
_tick():
  if mode == TEACH:
      return  # 或仅更新 state，不 _publish_cmd
  ... 现有 primitive compute ...
  if mode == PRIMITIVE or mode == STREAM:
      _publish_cmd(...)
```

`stop()`：只清 `_active_primitive`，不改变 `mode`。

---

## 7. 与现有原语 / 手型的关系

- 内置 35 原语 × O20/O6/L6/L25(L20) 仍由 YAML allowlist 管理（见 README / `config/*.yaml`）。
- `named_pose`：所有已支持手型均可启用（库为空时 `pose x` 返回不存在）。
- L20：配置别名 → 运行 `l25`；姿态文件键 `l25`。

---

## 8. 实施阶段（可执行清单）

### Phase 0 — MVP（建议先做）

- [x] `ControlMode`：`PRIMITIVE` | `TEACH`；`mode teach|primitive` 指令
- [x] TEACH 下 **禁止** `_publish_cmd`
- [x] `pose_library.py`：upsert / load / list / delete + 校验 + 保留字
- [x] `NamedPose` + 注册 + catalog + 各 YAML allowlist
- [x] `gesture_node`：`pose` / `pose_list` 特判；跳过 reach
- [x] CLI `save_pose`（依赖先 `mode teach` 或文档要求停节点争用）
- [x] 文档：示教流程、topic、手型键说明
- [x] 单测：校验、L20→l25 桶、保留字拒绝

**验收**

1. `mode teach` 后 GUI 可拖动且不被节点覆盖
2. 保存 `my_open` 后 `mode primitive` + `pose my_open` 平滑到位
3. L20 配置下保存的姿态在 L25 桶可读回放

### Phase 1 — 可用性

- [x] Teach UI：**本包独立 Web 面板（方案 A）** — 见 [`teach_panel_architecture.md`](./teach_panel_architecture.md)
- [x] Topic 统一 `/cb_*`；一键 `mode teach`
- [x] 导入 gui_control JSON（单向）
- [x] PoseLibrary 直接读文件（不经 gesture_params lru_cache；无需额外热加载）

### Phase 2 — 增强与模型

- [x] ROS Service：`SavePose` / `ListPoses` / `DeletePose` / `ImportGuiPresets`
- [x] Pose Bundle 导入导出（`ExportPoseBundle` / `ImportPoseBundle` + CLI）
- [x] 预定义静态原语终态导出（`ExportPrimitivePoses` / `export_primitive_poses`）
- [x] `PlanGesture` 白名单与 `planable_primitives` / Framework 对齐
- [x] Teach 面板导出/导入 Bundle（下载 / 上传 / 出厂预设）
- [ ] 可选双关键帧（仍 static/phased_pose）
- [x] STREAM 模式 + `/hand_gesture/stream_target` + watchdog
- [x] 模型 L1：策略只发原语名 / `pose:xxx`（现有 cmd 通道即可）

---

## 9. 测试矩阵

| 用例 | 期望 |
|------|------|
| teach 下拖 GUI | 关节跟随，无节点抢发 |
| primitive 下 `pose` | duration 内到位，过流仍保护 |
| 保存名 `open` | 拒绝 |
| 错误 DOF 长度 | 拒绝 |
| L6/O6 方向 | 张开/握拳与硬件约定一致 |
| L20 配置存取 | 落在 `l25.json` |
| `stop` 后仍 teach | 仍不发 cmd，直到 `mode primitive` |
| 库中无此名 | 明确 reject，不 KeyError |

---

## 10. 风险与对策（摘要）

| 风险 | 对策 |
|------|------|
| stop 仍发 last_target | TEACH 门控发布 |
| 指令 lower + 注册表 | `pose` 特判；强制小写名 |
| L20/L25 双库 | canonical 键 |
| SDK 双目录 | 独立库 + 可选导入 |
| 模型连续流无守护 | STREAM 必带 watchdog |
| O20 DOF 与 GUI 不一致 | 以 `hand_state` 长度为准 |

---

## 11. 接口速查（MVP）

```bash
# 进入示教
ros2 topic pub --once /hand_gesture_cmd_exec std_msgs/msg/String "data: 'mode teach'"

# 保存（CLI 示例）
ros2 run hand_gesture_primitives save_pose --ros-args \
  -p hand_side:=left -p name:=my_pinch -p duration:=0.6

# 回到原语模式并回放
ros2 topic pub --once /hand_gesture_cmd_exec std_msgs/msg/String "data: 'mode primitive'"
ros2 topic pub --once /hand_gesture_cmd_exec std_msgs/msg/String "data: 'pose my_pinch'"

# 列表
ros2 topic pub --once /hand_gesture_cmd_exec std_msgs/msg/String "data: 'pose_list'"
```

状态反馈建议扩展 `/hand_gesture_status` 字段：`mode=teach|primitive|stream`。

---

## 12. 总结

| 项 | 决策 |
|----|------|
| 架构 | Executor 上 **ControlMode**；三输入共享执行核 |
| 自定义手势 | PoseLibrary + `NamedPose` + `pose <name>` |
| 示教关键 | **先做 TEACH 停发布**，再做保存/回放 |
| 模型 | 先 L1 离散；连续角再 STREAM |
| 借鉴 | SDK `preset_store` 的存储/CRUD，不照搬直发 JointState |

落地顺序一句话：**Mode 门控 → PoseLibrary → NamedPose 指令 → CLI → 本包 Teach Web 面板（方案 A）→（远期）Stream。**

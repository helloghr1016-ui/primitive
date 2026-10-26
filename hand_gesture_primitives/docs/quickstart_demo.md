# Gesture Primitives 快速 Demo（上手 + 测试）

面向联调 / 交付：5～15 分钟跑通 **内置原语**、**自定义多关键帧**、**progress 细控**。

更全的逐条清单见 [`gesture_cmd_topic_test.md`](gesture_cmd_topic_test.md)；架构见 [`motion_keyframes_and_progress.md`](motion_keyframes_and_progress.md)。

---

## 0. 前置

```bash
# 仓库根目录；建议 Docker 开发容器内
scripts/build.sh -p hand_gesture_primitives_msgs hand_gesture_primitives
# source 安装空间（merge-install 常见为 .ws/devel）
```

| 变量 | 含义 |
|------|------|
| `SIDE` | `left` 或 `right`（与 Teach 页眉一致） |
| `MOCK=1` | 硬件回显，不连真机 |

**发指令优先**用 `pub_gesture_cmd`（Docker `--network host` 下 `ros2 topic pub` 易卡 Waiting）。

```bash
SIDE=left
alias gcmd='ros2 run hand_gesture_primitives pub_gesture_cmd --'
```

---

## 1. 启动（选一条）

### A. 机载 MOCK + 手势（常用）

```bash
MOCK=1 GESTURE=1 GESTURE_SIDE=left \
  ros2 launch luban_bringup launch_onboard_nodes.py
```

双手：`GESTURE=both`。示教面板：`TEACH_PANEL=1 POSE_SERVER=1`（左 `http://127.0.0.1:8001/`，右 `8002`）。

### B. O6 手-only

```bash
MOCK=1 GESTURE=1 GESTURE_SIDE=left \
  ros2 launch linkerhand_hardware launch_o6_hand.py
```

### C. 真机

去掉 `MOCK=1`，确认 CAN / 手驱动已起；手型用 `LEFT_HAND_MODEL` / `RIGHT_HAND_MODEL`（`o6|l25|l6|g20`）。

约 10s 后确认：

```bash
gcmd $SIDE "mode primitive"
gcmd $SIDE open
# 或 topic：
# ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'open'}"
```

---

## 2. Demo A — 内置原语冒烟

```bash
gcmd $SIDE open
gcmd $SIDE init
gcmd $SIDE fist
gcmd $SIDE pinch
gcmd $SIDE open
```

**O6 独有：**

```bash
gcmd $SIDE "large_wrap prep"
gcmd $SIDE "large_wrap close"
gcmd $SIDE "hammer_grip prep"
gcmd $SIDE "hammer_grip close"
```

**一键套件**（按手型 allowlist 裁剪；**不含** progress / 多关键帧）：

```bash
ros2 run hand_gesture_primitives run_gesture_cmd_suite -- $SIDE --hand-type o6 --dry-run
ros2 run hand_gesture_primitives run_gesture_cmd_suite -- $SIDE --hand-type o6 --pause
# L25：--hand-type l25
```

---

## 3. Demo B — 自定义多关键帧（Teach）

1. `TEACH_PANEL=1` 启动后打开对应侧网页。
2. **开始示教** → 名称填 `grip` → **关键帧**填 `pre` → 摆姿 → **保存**。
3. 再摆下一姿态 → 关键帧改 `grasp` → 名称仍为 `grip` → **保存**。
4. **退出示教**（或勾选「播放结束后自动回到示教」后点播放）。
5. 列表点 **播放** → 等价于整段 AUTO：`pose grip`。

**注意：** 已有多帧时「关键帧」栏勿留空再保存（会拒写，防抹掉 keyframes）。

CLI 等价存帧：

```bash
gcmd $SIDE "mode teach"
ros2 run hand_gesture_primitives save_pose --ros-args \
  -p name:=grip -p keyframe:=pre -p hand_side:=$SIDE -p hand_type:=o6 \
  -p ros2_control_command_units:=radians
# 摆到下一姿态后：
# -p keyframe:=grasp ...
gcmd $SIDE "mode primitive"
```

---

## 4. Demo C — 多中间态 + progress 细控（交付重点）

面板「播放」**只有**整段 AUTO。细控用 topic / `pub_gesture_cmd`。

约定：`progress` **只在当前段** 0～1，**不跨段**；`mid 1.0` 不会自动进 `grasp`。`motion_progress` 只改同段 p。假设 `grip` 已存 `pre` → `mid` → `grasp`。

```bash
# 须在 mode primitive
gcmd $SIDE "mode primitive"

# 整段 AUTO（与面板播放相同）
gcmd $SIDE "pose grip"

# 逐段 hold（换 kf = 重锚）
gcmd $SIDE "pose grip pre"
gcmd $SIDE "pose grip mid"
gcmd $SIDE "pose grip grasp"

# 某中间态段内定点 / scrub（同段反复发 = 热更新）
gcmd $SIDE "pose grip mid 0.0"
gcmd $SIDE "pose grip mid 0.35"
gcmd $SIDE "pose grip mid 1.0"      # 可回拉；不卸原语、不自动下一段
gcmd $SIDE "pose grip grasp 0.4"    # 换段后再细控
```

**纯 topic 形式：**

```bash
ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String \
  "{data: 'pose grip mid 0.35'}"
ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String \
  "{data: 'pose grip grasp 0.4'}"

# 高频细调（另开终端；先 gesture_cmd 进目标段）
ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String \
  "{data: 'pose grip mid 0.0'}"
ros2 topic pub /${SIDE}_hand/motion_progress std_msgs/msg/Float32 \
  "{data: 0.2}" -r 20
# Ctrl+C；改 data 为 0.5 / 0.8 / 0.0 …；换段须再发 pose … <新kf>
```

| 测什么 | Topic | `data` |
|--------|--------|--------|
| 全跑 | `gesture_cmd` | `pose grip` |
| 进中间态 hold | `gesture_cmd` | `pose grip mid` |
| 段内定点 | `gesture_cmd` | `pose grip mid 0.35` |
| 同段回拉 | `motion_progress` | `0.1`（Float32） |
| 换下一段 | `gesture_cmd` | `pose grip grasp 0.3` |

> Teach 面板若提供**全局**进度滑条（跨段映射），与 topic **段内** `progress` 语义不同；topic / Skills 细控一律按段内 0～1。

**内置相位 progress（无需自定义库）：**

```bash
gcmd $SIDE "tripod prep 0.3"
gcmd $SIDE "tripod prep 0.7"
gcmd $SIDE "large_wrap prep 0.4"    # O6/L6
# 力控 close 勿加 progress（会拒绝），例如：
# tripod_by_force close 0.3  → rejected
```

观测：

```bash
ros2 topic echo /${SIDE}_hand/gesture_status
ros2 topic echo /${SIDE}_hand/gesture_result
# 细控时期望含 drive=progress|keyframe=...|progress=0.xxx
# 细控期 motion_done 不因 scrub 置位
```

---

## 5. 对照速查

| 目标 | 指令 |
|------|------|
| 内置张开 | `open` |
| 查询自定义库 | `pose_list` |
| 整段回放 | `pose <name>` |
| 进某关键帧 hold | `pose <name> <kf>` |
| 段内进度 | `pose <name> <kf> <0~1>` |
| 高频 scrub | `/${SIDE}_hand/motion_progress` Float32 |
| 示教让权 | `mode teach` |
| 回原语 | `mode primitive` |

| 工具 | 用途 |
|------|------|
| `pub_gesture_cmd` | 单次发 cmd（推荐） |
| `run_gesture_cmd_suite` | 原语回归；**不测** progress/关键帧 |
| Teach 面板 | 存多帧 + 整段播放 |
| `save_pose -p keyframe:=` | CLI 存帧 |

---

## 6. 常见问题

| 现象 | 处理 |
|------|------|
| `topic pub` Waiting | 用 `pub_gesture_cmd`；或 `ROS2CLI_NO_DAEMON=1` |
| teach 下原语被拒 | 先 `mode primitive` |
| 裸发 `grip` 未知原语 | 必须 `pose grip` |
| 空关键帧保存抹多帧 | 已拒绝；请填 `pre`/`grasp` |
| RViz 不动 | 确认 SIDE 与面板一致；看 `gesture_result` / `*/commands` 是否在发 |
| suite 测不了细控 | 正常；用 §4 |

---

## 7. 文档索引

| 文档 | 内容 |
|------|------|
| 本文 | 上手 Demo |
| [`gesture_cmd_topic_test.md`](gesture_cmd_topic_test.md) | 各手型全量 topic 清单 |
| [`interfaces.md`](interfaces.md) | Topic / 参数 / Bundle |
| [`motion_keyframes_and_progress.md`](motion_keyframes_and_progress.md) | 关键帧 + progress 设计 |
| [`control_modes_and_custom_pose.md`](control_modes_and_custom_pose.md) | 三模式 + Pose 基线 |
| 包根 [`README.md`](../../README.md) | 总览与依赖 |

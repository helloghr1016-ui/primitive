# 手势 Topic 测试指令手册

真机 / MOCK 下通过 `/{side}_hand/gesture_cmd` 逐条验证原语。
手型：`o6` / `l6` / `o20` / `l25` / `g20`。

**上手最短路径**（启动 + Teach 多帧 + progress）：[`quickstart_demo.md`](quickstart_demo.md)。

## Topic 说明

| 项目 | 值 |
|------|-----|
| 指令 Topic | `/{side}_hand/gesture_cmd` |
| 消息类型 | `std_msgs/msg/String` |
| 左手 | `/left_hand/gesture_cmd` |
| 右手 | `/right_hand/gesture_cmd` |
| 状态 | `/left_hand/gesture_status`、`/left_hand/motion_done`（右手同理） |
| 进度细控 | `/{side}_hand/motion_progress`（`Float32`，0～1） |

每条指令发一次（`--once`）。发下一条前建议等 `motion_done` 或间隔 2～3 秒。
**例外**：progress 细控期间 `motion_done` 不会因 scrub 置位；以 `gesture_status` / 关节为准。

模板：

```bash
SIDE=left   # 或 right

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'open'}"
```

若 Docker `--network host` 下 `ros2 topic pub` 卡住，可用等价命令：

```bash
ros2 run hand_gesture_primitives pub_gesture_cmd -- left open
```

### 全套顺序回归（推荐）

编译后一条命令按序跑完下列全部原语（内置每步等待；力控/侧捏步更长）：

```bash
scripts/build.sh -p hand_gesture_primitives

# 左手全套（推荐显式 hand-type；会按 YAML allowlist 跳过不支持原语）
ros2 run hand_gesture_primitives run_gesture_cmd_suite -- left --hand-type l25

# 未给 --hand-type 时，尝试读取 /{side}_hand_gesture_node 的 hand_type 并裁剪
ros2 run hand_gesture_primitives run_gesture_cmd_suite -- left --pause

# 按 O6 裁剪（含 large_wrap / hammer_grip；跳过 grasp_ready、侧夹等）
ros2 run hand_gesture_primitives run_gesture_cmd_suite -- left --hand-type o6 --dry-run

# 统一 3s 间隔
ros2 run hand_gesture_primitives run_gesture_cmd_suite -- left --hand-type l25 --wait 3

# 等 /left_hand/motion_done 再继续（超时 fallback sleep）
ros2 run hand_gesture_primitives run_gesture_cmd_suite -- left --hand-type l25 --wait-motion

# 只跑重点关注对指类
ros2 run hand_gesture_primitives run_gesture_cmd_suite -- left --hand-type l25 --tags focus

# 从 pinch 起跑到结尾
ros2 run hand_gesture_primitives run_gesture_cmd_suite -- left --hand-type l25 --from pinch

# 预览不发布（仍会打印 SKIP 原因）
ros2 run hand_gesture_primitives run_gesture_cmd_suite -- left --hand-type o20 --dry-run

# 逐步人工确认（每步按 Enter；跳过项不暂停）
ros2 run hand_gesture_primitives run_gesture_cmd_suite -- left --hand-type l25 --pause

# 强制仍发不支持原语（测节点拒绝路径）
ros2 run hand_gesture_primitives run_gesture_cmd_suite -- left --hand-type o6 --include-unsupported --only grasp_ready
```

`--hand-type`：`l25` / `o20` / `g20` / `o6` / `l6`（与 `config/*.yaml` allowlist 一致）。
`--tags` 可选：`setup,safe,static,focus,phased,force,vision,motion`。
`--no-auto-hand`：禁止探测节点参数；未给 `--hand-type` 时跑全集不裁剪。

**范围说明**：`run_gesture_cmd_suite` 只覆盖内置原语字符串，**不测** `pose` 多关键帧与 `motion_progress`。细控请用下方「多关键帧 / progress」或 [`quickstart_demo.md`](quickstart_demo.md) §4。

开了 GraspGate 时，节点订阅 `gesture_cmd_exec`，但**用户仍发到 `gesture_cmd`**，Gate 会转发。

---

## 〇、多关键帧 / progress 细控（自定义 + 内置相位）

须在 `mode primitive`。Teach 面板「播放」= 整段 `pose <name>`（AUTO）；细控走 topic。

```bash
SIDE=left

# 整段 AUTO
ros2 run hand_gesture_primitives pub_gesture_cmd -- $SIDE "pose grip"

# 进入某关键帧 hold / 段内进度
ros2 run hand_gesture_primitives pub_gesture_cmd -- $SIDE "pose grip grasp"
ros2 run hand_gesture_primitives pub_gesture_cmd -- $SIDE "pose grip grasp 0.4"

# 等价 topic
ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String \
  "{data: 'pose grip grasp 0.4'}"

# 高频 scrub（先进入 progress 段）
ros2 topic pub /${SIDE}_hand/motion_progress std_msgs/msg/Float32 "{data: 0.2}" -r 20

# 内置：tripod / large_wrap 的 prep 相位；力控 close 加 progress 会被拒绝
ros2 run hand_gesture_primitives pub_gesture_cmd -- $SIDE "tripod prep 0.3"
ros2 run hand_gesture_primitives pub_gesture_cmd -- $SIDE "large_wrap prep 0.5"
```

观测：`/${SIDE}_hand/gesture_status`、`gesture_result`（细控期常含 `drive=progress|keyframe=...`）。

示教存多帧：面板「关键帧」字段，或 `save_pose -p keyframe:=grasp`。已有多帧时勿空关键帧保存（节点会拒写）。

---

## 一、启动指令

### 1.1 机载全套（真机）

```bash
# 双手 gesture
GESTURE=both GESTURE_SIDE=both ros2 launch luban_bringup launch_onboard_nodes.py

# 仅左手
GESTURE=1 GESTURE_SIDE=left ros2 launch luban_bringup launch_onboard_nodes.py

# 仅右手
GESTURE=1 GESTURE_SIDE=right ros2 launch luban_bringup launch_onboard_nodes.py
```

可选环境变量：

```bash
GESTURE_GATE=1           # 开 GraspGate（vision / auto 需要）
POSE_SERVER=1            # 姿态库 ROS 服务
TEACH_PANEL=1            # Web 示教 http://127.0.0.1:8001（右手 8002）
GESTURE_CONTROL_RATE=10
```

### 1.2 机载 MOCK（无真机）

```bash
MOCK=1 GESTURE=both ros2 launch luban_bringup launch_onboard_nodes.py
```

### 1.3 单手仅 gesture 节点（不调硬件）

```bash
# L25
ros2 launch hand_gesture_primitives gesture_node.launch.py hand_type:=l25 hand_side:=left launch_gate:=false
ros2 launch hand_gesture_primitives gesture_node.launch.py hand_type:=l25 hand_side:=right launch_gate:=false

# O20 / G20 / O6 / L6
ros2 launch hand_gesture_primitives gesture_node.launch.py hand_type:=o20 hand_side:=left launch_gate:=false
ros2 launch hand_gesture_primitives gesture_node.launch.py hand_type:=g20 hand_side:=left launch_gate:=false g20_joint_order:=l25_urdf
ros2 launch hand_gesture_primitives gesture_node.launch.py hand_type:=o6  hand_side:=left launch_gate:=false
ros2 launch hand_gesture_primitives gesture_node.launch.py hand_type:=l6  hand_side:=left launch_gate:=false
```

双手 gesture（无 onboard）：

```bash
ros2 launch hand_gesture_primitives bimanual_gesture.launch.py hand_type:=l25 launch_gate:=false
```

### 1.4 手-only 硬件 + gesture（单右手）

```bash
MOCK=1 GESTURE=1 GESTURE_SIDE=right ros2 launch linkerhand_hardware launch_l25_hand.py
MOCK=1 GESTURE=1 GESTURE_SIDE=right ros2 launch linkerhand_hardware launch_g20_hand.py
MOCK=1 GESTURE=1 GESTURE_SIDE=right ros2 launch linkerhand_hardware launch_o20_hand.py
MOCK=1 GESTURE=1 GESTURE_SIDE=right ros2 launch linkerhand_hardware launch_o6_hand.py
MOCK=1 GESTURE=1 GESTURE_SIDE=right ros2 launch linkerhand_hardware launch_l6_hand.py
```

---

## 二、观测

```bash
# 左手
ros2 topic echo /left_hand/gesture_status
ros2 topic echo /left_hand/motion_done

# 右手
ros2 topic echo /right_hand/gesture_status
ros2 topic echo /right_hand/motion_done
```

---

## 三、通用控制（先发 mode primitive）

### 左手

```bash
SIDE=left

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'mode primitive'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'stop'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'pose_list'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'pose my_open'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'mode teach'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'target bottle'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'target_id 1'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'auto'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'auto precision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'auto lateral'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'auto power'}"
```

### 右手

```bash
SIDE=right

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'mode primitive'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'stop'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'pose_list'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'pose my_open'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'mode teach'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'auto'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'auto precision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'auto lateral'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'auto power'}"
```

**auto 映射：**

| 手型 | precision | lateral | power |
|------|----------|---------|-------|
| L25 / O20 / G20 | `index_ring_by_vision` | `index_middle_adduction_grip` | `large_wrap_by_vision` |
| O6 / L6 | `middle_ring` | `thumb_adduction_grip` | `ring` |

---

## 四、L25 全量测试（`hand_type:=l25`）

L25 独有 runtime：`grasp_ready`、`index_flick`、`twist*`。

### 左手

```bash
SIDE=left

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'mode primitive'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'init'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'open'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'release'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'relax_grip'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'fist'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'pinch'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'index_pinch'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'middle_pinch'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'ok_sign'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'grasp_ready'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'point'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'v_sign'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'parallel_extension'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'thumb_adduction_grip'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'thumb_adduction_grip prep'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'thumb_adduction_grip close'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'index_middle_adduction_grip'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'index_middle_adduction_grip prep'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'index_middle_adduction_grip close'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'tripod'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'tripod prep'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'tripod close'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'tripod_by_force'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'tripod_by_force prep'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'tripod_by_force close'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'ring'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'middle_ring'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'index_ring_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'large_wrap_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'middle_ring_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'ring_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'small_warp_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'no_index_warp_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'hook_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'index_pinch_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'middle_pinch_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'tripod_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'palmar_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'parallel_extension_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'disk_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'index_flick'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'index_middle_flick'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'twist2digit'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'twist3digit'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'twist5digit'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'open'}"
```

### 右手

```bash
SIDE=right

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'mode primitive'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'init'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'open'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'release'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'relax_grip'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'fist'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'pinch'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'index_pinch'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'middle_pinch'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'ok_sign'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'grasp_ready'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'point'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'v_sign'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'parallel_extension'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'thumb_adduction_grip'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'thumb_adduction_grip prep'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'thumb_adduction_grip close'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'index_middle_adduction_grip'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'index_middle_adduction_grip prep'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'index_middle_adduction_grip close'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'tripod'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'tripod prep'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'tripod close'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'tripod_by_force'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'tripod_by_force prep'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'tripod_by_force close'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'ring'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'middle_ring'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'index_ring_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'large_wrap_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'middle_ring_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'ring_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'small_warp_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'no_index_warp_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'hook_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'index_pinch_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'middle_pinch_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'tripod_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'palmar_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'parallel_extension_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'disk_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'index_flick'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'index_middle_flick'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'twist2digit'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'twist3digit'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'twist5digit'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'open'}"
```

---

## 五、O20 全量测试（`hand_type:=o20`）

相对 L25：**无** `grasp_ready` / `index_flick` / `index_middle_flick` / `twist*`。

### 左手

```bash
SIDE=left

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'mode primitive'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'init'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'open'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'release'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'relax_grip'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'fist'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'pinch'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'index_pinch'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'middle_pinch'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'point'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'ok_sign'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'v_sign'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'parallel_extension'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'thumb_adduction_grip'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'thumb_adduction_grip prep'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'thumb_adduction_grip close'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'index_middle_adduction_grip'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'index_middle_adduction_grip prep'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'index_middle_adduction_grip close'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'tripod'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'tripod prep'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'tripod close'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'tripod_by_force'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'tripod_by_force prep'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'tripod_by_force close'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'ring'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'middle_ring'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'index_ring_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'large_wrap_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'middle_ring_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'ring_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'small_warp_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'no_index_warp_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'hook_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'index_pinch_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'middle_pinch_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'tripod_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'palmar_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'parallel_extension_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'disk_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'open'}"
```

### 右手

```bash
SIDE=right

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'mode primitive'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'init'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'open'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'release'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'relax_grip'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'fist'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'pinch'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'index_pinch'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'middle_pinch'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'point'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'ok_sign'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'v_sign'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'parallel_extension'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'thumb_adduction_grip'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'thumb_adduction_grip prep'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'thumb_adduction_grip close'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'index_middle_adduction_grip'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'index_middle_adduction_grip prep'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'index_middle_adduction_grip close'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'tripod'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'tripod prep'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'tripod close'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'tripod_by_force'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'tripod_by_force prep'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'tripod_by_force close'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'ring'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'middle_ring'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'index_ring_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'large_wrap_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'middle_ring_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'ring_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'small_warp_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'no_index_warp_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'hook_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'index_pinch_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'middle_pinch_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'tripod_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'palmar_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'parallel_extension_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'disk_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'open'}"
```

---

## 六、G20 全量测试（`hand_type:=g20`）

相对 O20：**多** flick/twist；**无** `grasp_ready`。整机常加 `g20_joint_order:=l25_urdf`。

### 左手

```bash
SIDE=left

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'mode primitive'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'init'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'open'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'release'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'relax_grip'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'fist'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'pinch'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'index_pinch'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'middle_pinch'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'point'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'ok_sign'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'v_sign'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'parallel_extension'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'thumb_adduction_grip'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'thumb_adduction_grip prep'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'thumb_adduction_grip close'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'index_middle_adduction_grip'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'index_middle_adduction_grip prep'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'index_middle_adduction_grip close'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'tripod'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'tripod prep'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'tripod close'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'tripod_by_force'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'tripod_by_force prep'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'tripod_by_force close'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'ring'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'middle_ring'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'index_ring_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'large_wrap_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'middle_ring_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'ring_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'small_warp_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'no_index_warp_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'hook_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'index_pinch_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'middle_pinch_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'tripod_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'palmar_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'parallel_extension_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'disk_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'index_flick'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'index_middle_flick'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'twist2digit'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'twist3digit'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'twist5digit'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'open'}"
```

### 右手

```bash
SIDE=right

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'mode primitive'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'init'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'open'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'release'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'relax_grip'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'fist'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'pinch'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'index_pinch'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'middle_pinch'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'point'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'v_sign'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'ok_sign'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'parallel_extension'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'thumb_adduction_grip'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'thumb_adduction_grip prep'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'thumb_adduction_grip close'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'index_middle_adduction_grip'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'index_middle_adduction_grip prep'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'index_middle_adduction_grip close'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'tripod'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'tripod prep'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'tripod close'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'tripod_by_force'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'tripod_by_force prep'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'tripod_by_force close'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'ring'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'middle_ring'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'index_ring_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'large_wrap_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'middle_ring_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'ring_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'small_warp_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'no_index_warp_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'hook_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'index_pinch_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'middle_pinch_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'tripod_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'palmar_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'parallel_extension_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'disk_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'index_flick'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'index_middle_flick'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'twist2digit'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'twist3digit'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'twist5digit'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'open'}"
```

---

## 七、O6 全量测试（`hand_type:=o6`）

**独有：** `large_wrap` · `hammer_grip` · `middle_pinch`
**无：** `index_middle_adduction_grip` · `parallel_extension` · flick/twist

### 左手

```bash
SIDE=left

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'mode primitive'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'init'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'open'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'release'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'relax_grip'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'fist'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'pinch'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'index_pinch'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'middle_pinch'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'point'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'ok_sign'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'v_sign'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'large_wrap'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'large_wrap prep'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'large_wrap close'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'large_wrap auto'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'hammer_grip prep'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'hammer_grip close'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'thumb_adduction_grip'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'thumb_adduction_grip prep'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'thumb_adduction_grip close'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'tripod'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'tripod prep'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'tripod close'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'tripod_by_force'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'tripod_by_force prep'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'tripod_by_force close'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'ring'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'middle_ring'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'ring_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'middle_ring_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'index_ring_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'index_pinch_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'middle_pinch_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'tripod_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'large_wrap_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'small_warp_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'no_index_warp_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'hook_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'palmar_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'parallel_extension_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'disk_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'open'}"
```

### 右手

```bash
SIDE=right

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'mode primitive'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'init'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'open'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'release'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'relax_grip'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'fist'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'pinch'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'index_pinch'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'middle_pinch'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'point'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'ok_sign'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'v_sign'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'large_wrap'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'large_wrap prep'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'large_wrap close'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'large_wrap auto'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'hammer_grip prep'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'hammer_grip close'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'thumb_adduction_grip'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'thumb_adduction_grip prep'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'thumb_adduction_grip close'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'tripod'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'tripod prep'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'tripod close'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'tripod_by_force'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'tripod_by_force prep'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'tripod_by_force close'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'ring'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'middle_ring'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'ring_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'middle_ring_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'index_ring_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'index_pinch_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'middle_pinch_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'tripod_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'large_wrap_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'small_warp_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'no_index_warp_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'hook_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'palmar_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'parallel_extension_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'disk_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'open'}"
```

---

## 八、L6 全量测试（`hand_type:=l6`）

比 O6 **少：** `middle_pinch` · `middle_pinch_by_vision` · `parallel_extension_by_vision` · `hammer_grip`

### 左手

```bash
SIDE=left

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'mode primitive'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'init'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'open'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'release'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'relax_grip'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'fist'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'pinch'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'index_pinch'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'point'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'ok_sign'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'v_sign'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'large_wrap'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'large_wrap prep'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'large_wrap close'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'large_wrap auto'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'thumb_adduction_grip'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'thumb_adduction_grip prep'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'thumb_adduction_grip close'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'tripod'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'tripod prep'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'tripod close'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'tripod_by_force'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'tripod_by_force prep'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'tripod_by_force close'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'ring'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'middle_ring'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'ring_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'middle_ring_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'index_ring_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'index_pinch_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'tripod_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'large_wrap_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'small_warp_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'no_index_warp_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'hook_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'palmar_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'disk_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'open'}"
```

### 右手

```bash
SIDE=right

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'mode primitive'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'init'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'open'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'release'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'relax_grip'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'fist'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'pinch'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'index_pinch'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'point'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'ok_sign'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'v_sign'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'large_wrap'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'large_wrap prep'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'large_wrap close'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'large_wrap auto'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'thumb_adduction_grip'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'thumb_adduction_grip prep'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'thumb_adduction_grip close'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'tripod'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'tripod prep'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'tripod close'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'tripod_by_force'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'tripod_by_force prep'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'tripod_by_force close'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'ring'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'middle_ring'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'ring_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'middle_ring_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'index_ring_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'index_pinch_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'tripod_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'large_wrap_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'small_warp_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'no_index_warp_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'hook_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'palmar_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'disk_by_vision'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'open'}"
```

---

## 九、L25 P0 快速回归

对应本次 pinch 标定 / 弧度迁移 / rot_progressive 主测项。

### 左手

```bash
SIDE=left

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'mode primitive'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'init'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'open'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'pinch'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'index_pinch'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'middle_pinch'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'ok_sign'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'grasp_ready'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'thumb_adduction_grip prep'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'thumb_adduction_grip close'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'open'}"
```

### 右手

```bash
SIDE=right

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'mode primitive'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'init'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'open'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'pinch'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'index_pinch'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'middle_pinch'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'ok_sign'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'grasp_ready'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'thumb_adduction_grip prep'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'thumb_adduction_grip close'}"

ros2 topic pub --once /${SIDE}_hand/gesture_cmd std_msgs/msg/String "{data: 'open'}"
```

---

## 十、手型差异速查

| 指令 | O6 | L6 | O20 | L25 | G20 |
|------|:--:|:--:|:---:|:---:|:---:|
| `grasp_ready` | | | | ✅ | |
| `parallel_extension` | | | ✅ | ✅ | ✅ |
| `middle_pinch` | ✅ | | ✅ | ✅ | ✅ |
| `large_wrap` | ✅ | ✅ | | | |
| `hammer_grip` | ✅ | | | | |
| `index_middle_adduction_grip` | | | ✅ | ✅ | ✅ |
| flick / twist | | | | ✅ | ✅ |

## 备注

1. `*_by_vision` / `auto` 需 `GESTURE_GATE=1` 且感知 topic（`bboxes_3d` 等）有数据。
2. 不要直接发 `named_pose`；示教回放用 `pose <name>`。
3. 分阶段原语：`prep` 到位后再发 `close`；`tripod` / `tripod_by_force` 裸发默认 auto。
4. 相关文档：[`interfaces.md`](./interfaces.md)、[`primitives.md`](./primitives.md)。

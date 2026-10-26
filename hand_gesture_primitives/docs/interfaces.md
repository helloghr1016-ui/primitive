# 接口文档

本文档描述 `hand_gesture_primitives` 包对外暴露的 ROS2 接口：Topic、参数、消息格式与节点关系。

---

## 节点概览

| 节点名 | 可执行文件 | 功能 |
|--------|-----------|------|
| `{side}_hand_gesture_node` | `gesture_node` | 指令解析 + 可配置频率原语执行器（默认 10Hz） |
| `{side}_grasp_gate` | `grasp_gate` | 抓取可行性三条件门控（可选） |
| `joint_state_bridge` | `joint_state_bridge` | 硬件反馈 → URDF 弧度转换（RViz 可视化） |
| `mock_perception` | `mock_perception` | 模拟感知输出（测试用） |
| `teach_panel` | `teach_panel` | Web 示教面板（可选 `[teach]` 依赖） |
| `pose_library_server` | `pose_library_server` | Pose Library ROS Service |

---

## 数据流

**双手合同**（`side` = `left` | `right`；空参数自动侧化）：

```
/{side}_hand/gesture_cmd
/{side}_hand/gesture_cmd_exec      ← gate 开启时
/{side}_hand/gesture_status
/{side}_hand/gesture_result
/{side}_hand/tcp_pose
/{side}_hand/object_pose
/{side}_hand/grasp_gate/status
/{side}_hand/grasp_gate/selected_label
```

感知 `bboxes_3d` / `labeled_poses` 可左右共享。遗留全局名（`/hand_gesture_cmd` 等）仍可通过显式参数覆盖。

**上游责任（本包不发布）**

| Topic | 谁发布 | 说明 |
|-------|--------|------|
| `/{side}_hand/tcp_pose` | 臂/TF 桥或上层 | vision / GraspGate 必需，由集成方提供 |
| `/{side}_hand/gesture_cmd` | teleop / program / 人工 | 按目标侧发送；节点通过 `hand_side` 配置侧别，JSON 可带 `hand_side` |
| 感知 bbox | `robot_perception` | 可共享 |

迁移：旧集成若仍发 `/hand_gesture_cmd_exec`，需改为 `/{side}_hand/gesture_cmd(_exec)`，或启动时显式 `cmd_topic:=/hand_gesture_cmd_exec`（单手临时兼容；双手勿共用同一全局 topic）。

### 安装与运行入口

独立 Mock、ROS 原语节点和双手面板的完整启动步骤见
[安装与启动指南](../../docs/getting-started.md)。ROS 接入要求先准备设备驱动、
关节反馈和实际控制器 URDF；本仓库的 launch 文件只启动原语及相关消息处理节点。
感知示例配置见[感知部署说明](../../docs/perception-setup.md)。

### 手型力反馈

| 手型 | `capabilities` 要点 | 接触检测 | 备注 |
|------|---------------------|----------|------|
| O20 / L25 | `current` | 触觉 → 电流 Δ (mA) | `hand_info.current` |
| O6 | `motor_torque` + `touch` | 触觉 → 力矩 Δ (%) | `contact_delta_torque_pct` |
| L6 | `touch`；`current=false` | **触觉优先** | SDK 无 `getCurrent`，电流占位 0；`touch` 可替代目录中的电流要求 |
```
                    ┌─────────────────────────────┐
                    │  外部感知 (robot_perception)  │
                    │  bboxes_3d / labeled_poses   │
                    └──────────────┬──────────────┘
                                   │
/{side}_hand/gesture_cmd ──┐       │
                           ▼       ▼
                     ┌──────────┐   ┌──────────────────────────┐
                     │GraspGate │──▶│ {side}_hand_gesture_node │
                     │(可选门控) │   │  GestureNode + Executor  │
                     └──────────┘   └────────────┬─────────────┘
                                                 │
                       ┌─────────────────────────┼──────────────────────────┐
                       ▼                         ▼                          ▼
    /{side}_hand_controller/commands   /{side}_hand/gesture_status   /{side}_hand/gesture_result
```

---

## hand_gesture_node

节点名：`{side}_hand_gesture_node`（launch）；ROS 节点类仍可命名 `hand_gesture_node`。

### 订阅 Topic

| Topic | 消息类型 | 频率 | 说明 |
|-------|---------|------|------|
| `/{side}_hand/gesture_cmd`（无门控默认）或 `.../gesture_cmd_exec` | `std_msgs/String` | 按需 | 空 `cmd_topic` 默认 `gesture_cmd`；有门控时 launch 设为 `gesture_cmd_exec` |
| `/{side}_hand/motion_progress` | `std_msgs/Float32` | 建议 10～50 Hz | 热更新**当前**段内 progress∈[0,1]：NamedPose，以及支持 `set_progress` 的定位相位（如 `tripod`/`large_wrap` prep）。切换 keyframe / 启停仍用 `gesture_cmd`。无 active 或不支持时忽略/告警 |
| `/{side}_hand/stream_target` | `sensor_msgs/JointState` | 跟 stream | 仅 `mode stream` |
| `/joint_states` | `sensor_msgs/JointState` | ~100Hz | 机载手关节反馈（默认）；L25 按 `l/r_hand_*` 16 主动关节映射为 native 25 |
| `/cb_{side}_hand_state` | `sensor_msgs/JointState` | ~50Hz | 旧驱动反馈（可通过 `state_topic` 切回） |
| `/cb_{side}_hand_info` | `std_msgs/String` | ~10Hz | 手部信息 JSON（`current` / 历史字段 `current_current`）；**L6 在 C++ 驱动下电流为占位 0** |
| `/{side}_hand/tcp_pose` | `geometry_msgs/PoseStamped` | ~30Hz | 机械臂末端 TCP 位姿（空参自动） |
| `/{side}_hand/object_pose` | `geometry_msgs/PoseStamped` | ~5Hz | 目标物体位姿（空参自动） |
| `bboxes_3d` | `robot_perception_msgs/LabeledBBox3DArray` | ~5Hz | 感知 3D 包围盒（含 size/grasp_type） |
| `labeled_poses` | `robot_perception_msgs/LabeledPoseArray` | ~5Hz | 感知物体位姿数组（world frame） |
| `/cb_{side}_hand_force` | `hand_force_msg/HandForce` | ~10Hz | 指尖力 + 掌心压力矩阵（`finger_force` 按 finger/row/col 展平，`finger_names` 携带指序；触觉判定取每指峰值） |
| `/cb_{side}_hand_normal_force` | `std_msgs/Float32MultiArray` | ~10Hz | SDK 法向力（每指一个值） |
| `/cb_{side}_hand_matrix_touch_mass` | `std_msgs/String` | ~10Hz | 矩阵压感 JSON（每指总压力质量 g） |
| `/cb_{side}_hand_motor_torque` | `std_msgs/Float32MultiArray` | ~10Hz | 关节力矩反馈 |
| `/{side}_hand/grasp_gate/selected_label` | `std_msgs/String` | 按需 | 门控选中的物体 label |

> `{side}` = `left` 或 `right`，由参数 `hand_side` 决定。
> `robot_perception_msgs` 为可选依赖，缺失时自动降级（仅使用 object_pose）。

### 发布 Topic

| Topic | 消息类型 | 频率 | 说明 |
|-------|---------|------|------|
| `/{side}_hand_controller/commands` | `std_msgs/Float64MultiArray` | `control_rate`（默认 10Hz） | L25：16；O6：6；**O20：20**（`cmd_backend=ros2_control` 默认） |
| `/cb_{side}_hand_control_cmd` | `sensor_msgs/JointState` | `control_rate`（默认 10Hz） | `cmd_backend:=legacy_cb` 时旧驱动命令 |
| `/{side}_hand/gesture_status` | `std_msgs/String` | 0.5Hz | 当前状态：`原语名\|grasp=状态\|type=...\|suggest=...` |
| `/{side}_hand/gesture_result` | `std_msgs/String` | ~1Hz | 执行状态文本：`IDLE` / `EXECUTING: name` / `DONE: name` |
| `/cb_{side}_hand_fingertip_positions` | `std_msgs/Float32MultiArray` | `control_rate` | FK 指尖位置 [5×3] xyz (m)，手基坐标系 |
| `/cb_{side}_hand_fingertip_markers` | `visualization_msgs/MarkerArray` | `control_rate` | RViz 指尖球体 marker（5 个彩色球体） |

### 参数

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `hand_side` | string | `"left"` | 手侧：`left` / `right` |
| `hand_type` | string | `"o20"` | 手型：`o20` / `l25` / `o6` / `l6` / `g20` |
| `cmd_topic` | string | `""` → `/{side}_hand/gesture_cmd` | 指令输入；空则侧化自动 |
| `state_topic` | string | `"/joint_states"` | 手关节反馈；L25/O6/O20 机载按名抽取，旧驱动可设 `/cb_{side}_hand_state` |
| `cmd_backend` | string | `""` | `ros2_control` / `legacy_cb`；空则 **L25/O6/O20→ros2_control** |
| `ros2_control_command_units` | string | `radians` | `radians`=URDF 弧度（mock / MuJoCo / 真机默认，与 RViz·MoveIt·`/joint_states` 一致）；`hardware`=遗留 SDK 0~255（仅当硬件插件 `interface_units:=hardware`） |
| `control_rate` | float | `10.0` | 原语 `commands` 下发频率 Hz（1~200；常用 `10` / `100`）。机载/手-only 可用 `GESTURE_CONTROL_RATE`。YAML/`FINGER_STEP` 等按 10Hz 标定的 settle/步进/HOLD 回退会自动换算，保持墙钟时间与闭合速度 |
| `hand_cmd_topic` | string | `""` | 手命令 topic；空则按 backend 自动选择 |
| `tcp_pose_topic` | string | `""` → `/{side}_hand/tcp_pose` | TCP 位姿 topic |
| `object_pose_topic` | string | `""` → `/{side}_hand/object_pose` | 物体位姿 topic |
| `bboxes_3d_topic` | string | `"/camera_head/detection_bbox/bboxes_3d"` | 3D 包围盒 topic |
| `labeled_pose_topic` | string | `"/camera_head/perception/labeled_poses"` | 感知位姿数组 topic |
| `target_object_label` | string | `""` | 目标物体 label |
| `result_topic` | string | `""` → `/{side}_hand/gesture_result` | 执行结果 topic |
| `selected_label_topic` | string | `""` → `/{side}_hand/grasp_gate/selected_label` | 门控选中 label topic |
| `status_topic` | string | `""` → `/{side}_hand/gesture_status` | 状态 topic |
| `reach_check_strict` | bool | `false` | 严格模式：不可达时拒绝执行 |
| `reach_max_distance` | float | `0.15` | TCP→目标最大允许距离 (m) |
| `palm_offset_x` | float | `0.0` | 掌心相对 TCP 偏移 X (m) |
| `palm_offset_y` | float | `0.0` | 掌心相对 TCP 偏移 Y (m) |
| `palm_offset_z` | float | `-0.05` | 掌心相对 TCP 偏移 Z (m) |
| `publish_fingertips` | bool | `true` | 是否发布 FK 指尖位置 |
| `fingertip_frame` | string | `""` | 指尖 marker frame_id（空=自动） |
| `urdf_path` | string | `""` | URDF 路径（空=自动搜索） |
| `contact_config_path` | string | `""` | 接触/电流阈值 YAML（空=包内 `config/contact_thresholds.yaml`） |
| `contact_pressure_threshold` | float | `20.0` | 触觉压感接触阈值 (0–255) |
| `contact_mass_threshold` | float | `2.0` | 矩阵触觉质量阈值 (g) |
| `contact_current_delta` | float | `270` | 无触觉时电流增量接触阈值 (mA) |
| `contact_current_delta_narrow` | float | `250` | 窄物体力控原语电流增量 (mA) |
| `contact_current_settle_frames` | int | `10` | 电流基线稳定帧数（按 10Hz 标定；随 `control_rate` 自动缩放，默认约 1s） |
| `hold_safe_current` | float | `800` | 力控持握安全电流上限 (mA) |
| `overload_current_threshold` | float | `1000` | 全局过载保护阈值 (mA) |
| `overload_duration_sec` | float | `2.0` | 过载保护持续时间 (s) |

默认阈值见安装目录 `share/hand_gesture_primitives/config/contact_thresholds.yaml`，可按现场负载修改 YAML 或通过 launch 参数覆盖。

> **位置与量纲**：有真实 URDF 转换器时，原语结果使用 S30 语义布局中的弧度。
> 旧 0–255 数据在兼容边界转换；姿态与控制器位置的区别见
> [工作台数据约定](../../docs/workbench.md#数据与公共接口)。

---

## grasp_gate

### 订阅 Topic

| Topic | 消息类型 | 说明 |
|-------|---------|------|
| `/{side}_hand/gesture_cmd` | `std_msgs/String` | 用户原始手势指令（空参自动） |
| `/{side}_hand/tcp_pose` | `geometry_msgs/PoseStamped` | TCP 位姿 |
| `bbox_topic` | `robot_perception_msgs/LabeledBBox3DArray` | 3D 包围盒 |
| `labeled_pose_topic` | `robot_perception_msgs/LabeledPoseArray` | 感知位姿（备选） |

### 发布 Topic

| Topic | 消息类型 | 说明 |
|-------|---------|------|
| `/{side}_hand/gesture_cmd_exec` | `std_msgs/String` | 过滤后的指令（转发给 gesture_node） |
| `/{side}_hand/grasp_gate/status` | `std_msgs/String` | 门控判定结果：`OK: cmd, 'label' fwd=Xm` / `REJECT: cmd, reason` |
| `/{side}_hand/object_pose` | `geometry_msgs/PoseStamped` | 选中物体位姿（world frame，供 gesture_node 可达性检查） |
| `/{side}_hand/grasp_gate/selected_label` | `std_msgs/String` | 选中物体 label |

### 参数

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `hand_side` | string | `"left"` | 手侧 |
| `tcp_pose_topic` | string | `""` → `/{side}_hand/tcp_pose` | TCP 位姿 topic |
| `bbox_topic` | string | `"/camera_head/detection_bbox/bboxes_3d"` | 3D 包围盒 topic |
| `labeled_pose_topic` | string | `"/camera_head/perception/labeled_poses"` | 感知位姿 topic |
| `input_cmd_topic` | string | `""` → `/{side}_hand/gesture_cmd` | 原始指令输入 |
| `output_cmd_topic` | string | `""` → `/{side}_hand/gesture_cmd_exec` | 过滤后指令输出 |
| `status_topic` | string | `""` → `/{side}_hand/grasp_gate/status` | 状态输出 topic |
| `selected_pose_topic` | string | `""` → `/{side}_hand/object_pose` | 选中物体位姿输出 |
| `selected_label_topic` | string | `""` → `/{side}_hand/grasp_gate/selected_label` | 选中物体 label 输出 |
| `palm_z` | float | `0.06` | TCP → 掌心 Z 偏移 (m) |
| `tip_z` | float | `0.18` | TCP → 指尖 Z 偏移 (m) |
| `half_w` | float | `0.20` | 掌面左右半宽 (m) |
| `half_h` | float | `0.15` | 掌面上下半高 (m) |
| `height_tol` | float | `0.05` | 高度 Z 容差 (m) |
| `data_timeout` | float | `2.0` | 数据过期时间 (s) |
| `de_bounce` | int | `3` | 连续通过帧数阈值 |

### 门控逻辑

对下列 **11 个** `*_by_vision` 包络/定位原语执行三条件判定，其余指令（含 `index_ring_by_vision` / `large_wrap_by_vision` / `auto` 映射）直接转发：

`ring_by_vision` `small_warp_by_vision` `no_index_warp_by_vision` `middle_ring_by_vision` `hook_by_vision` `tripod_by_vision` `palmar_by_vision` `parallel_extension_by_vision` `index_pinch_by_vision` `middle_pinch_by_vision` `disk_by_vision`

| 条件 | 判定方式 | 通过条件 |
|------|---------|---------|
| **H** (高度) | 物体世界 Z 范围 ∩ 手世界 Z 可达范围 | 有交叠 |
| **XY** (掌面) | 物体中心在手坐标系掌面矩形内 | \|cx\| < half_w 且 \|cy\| < half_h |
| **Fwd** (前方) | 物体在手坐标系中的 Z 坐标 | cz > 0 |

坐标系转换：世界 → TCP → R_z(90°) → 手坐标系

```
hand +X = -TCP_Y (掌面左右)
hand +Y = +TCP_X (掌面上下)
hand +Z = +TCP_Z (手指前方)
```

---

## joint_state_bridge

将手部硬件 O20/L25 格式反馈转为 URDF 弧度，供 `robot_state_publisher` 驱动 RViz 模型。

### 订阅 Topic

| Topic | 消息类型 | 说明 |
|-------|---------|------|
| `/cb_{side}_hand_state` | `sensor_msgs/JointState` | 硬件关节反馈 |

### 发布 Topic

| Topic | 消息类型 | 说明 |
|-------|---------|------|
| `/joint_states` | `sensor_msgs/JointState` | URDF 弧度关节状态 |

### 参数

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `hand_side` | string | `"left"` | 手侧 |
| `hand_joint` | string | `"L25"` | 手型标识 |
| `urdf_path` | string | `""` | URDF 路径（空=自动搜索） |

---

## 指令协议

通过 `/{side}_hand/gesture_cmd`（或参数 `cmd_topic`）接收 `std_msgs/String`，格式：

```
<command> [args...]
```

### 指令列表

| 指令 | 格式 | 说明 |
|------|------|------|
| `<原语名>` | `pinch` / `open` / `fist` / `tripod` ... | 执行注册表及当前手型 allowlist 中的原语 |
| `<原语名> <label>` | `index_ring screwdriver` | 执行原语 + 设置目标 label 过滤 |
| `<原语名> prep` | `tripod prep` / `hammer_grip prep` | 仅预备阶段，到位后等待 close |
| `<原语名> close` | `hammer_grip close` | 执行分阶段原语的闭合阶段 |
| `<原语名>` / `auto` / `full` | `tripod` / `tripod auto` | 自动 prep→停顿→close（tripod 裸发=auto；full 为别名） |
| `large_wrap auto` | `large_wrap auto` | O6/L6 依次执行 prep、等待 1.0s、close |
| `auto` | `auto` / `auto [label]` / `auto [label] prep` | 根据感知 grasp_type 自动选择原语 |
| `target <label>` | `target screwdriver` | 设置目标物体 label 过滤（不执行） |
| `target_id <id>` | `target_id 3` | 按 instance_id 过滤（0=清除） |
| `stop` | `stop` | 停止当前原语，保持最终姿态 |
| `mode <teach\|primitive\|stream>` | `mode teach` | 控制权切换；`teach` 停发 cmd；`stream` 订阅 `stream_target` |
| `pose <name>` | `pose my_open` | 回放 Pose Library（**必须**带 `pose` 前缀，勿裸发名字） |
| `pose <name> <kf> [p]` | `pose my_grip pre2 0.35` | 多关键帧：进入 `kf` 并以 progress∈[0,1] 细控；仅 `<kf>` 则 progress=0 hold。同名同段重复发只热更新 progress（不重锚） |
| `pose_list` | `pose_list` | 列出当前手型/侧；`gesture_result`/`status` 发 `POSES:a,b`；推荐用 `pub_gesture_cmd` 在 **stdout** 打印 |

高频 scrub（推荐）：先 `pose my_grip grasp`（或带初始 p）进入段，再向
`/{side}_hand/motion_progress` 发 `std_msgs/Float32`（建议 **10～50 Hz**；手部命令下发仍跟
`control_rate`，默认 **10 Hz**）。`gesture_cmd` 的 `pose <name> <kf> <p>` 同名同段重复发亦热更新（不重锚）；换 kf / 裸 `pose <name>` 仍重进。

内置定位相位（有限）：`tripod prep 0.3` / `large_wrap close 0.5`；
`tripod_by_force` / `*_adduction_grip close` / `hammer_grip close` **拒绝** progress。

**控制模式**：同一时刻仅一种 Authority——`primitive`（默认，执行原语/`pose`）、`teach`（停发 hand cmd，示教面板独占）、`stream`（跟 `stream_target`）。回放自定义姿态须在 `primitive`。

示教保存（独立节点，非 gesture_cmd）：

```bash
# 先 mode teach；真机/mock 默认 radians（须 robot_description）
ros2 run hand_gesture_primitives save_pose --ros-args \
  -p name:=my_open -p hand_side:=left -p hand_type:=g20 \
  -p ros2_control_command_units:=radians \
  -p g20_joint_order:=l25_urdf \
  -p robot_description:="..."
# 再 mode primitive → pose my_open
```

姿态文件：

- 0.2.0 起：必须显式设置 `HAND_GESTURE_POSE_DIR`，文件为 `{hand}_{side}.json`；不再回退到 HOME
- 所有读写同一姿态库的进程使用相同持久化目录，例如 `export HAND_GESTURE_POSE_DIR="$HOME/hand-gesture-data/poses"`
- JSON 损坏时会改名为 `*.corrupt.<ts>`，避免 upsert 静默清空整库

### STREAM 连续角

`mode stream` 后向 `/{side}_hand/stream_target`（`sensor_msgs/JointState`，hardware 0–255，长度=`num_joints`）发目标；超过 `stream_watchdog_sec`（默认 0.5s）无新帧则 hold。

### Pose Library 服务

| Service | 类型 | 说明 |
|---------|------|------|
| `/{side}_hand/save_pose` | `hand_gesture_primitives_msgs/SavePose` | 保存；`positions` 空则用缓存 state；可选 `keyframe`（追加/替换关键帧）、`space` |
| `/{side}_hand/list_poses` | `ListPoses` | 列出姿态名 |
| `/{side}_hand/delete_pose` | `DeletePose` | 删除 |
| `/{side}_hand/import_gui_presets` | `ImportGuiPresets` | 单向导入 gui_control JSON |
| `/{side}_hand/export_pose_bundle` | `ExportPoseBundle` | 导出当前手型/侧 Pose Library → `*.pose_bundle.json` |
| `/{side}_hand/import_pose_bundle` | `ImportPoseBundle` | 导入 bundle（`strategy=skip\|overwrite\|rename`） |
| `/{side}_hand/export_primitive_poses` | `ExportPrimitivePoses` | 预定义静态原语终态 → 库和/或 bundle（默认名 `preset_<prim>`） |

### 跨机迁移（Pose Bundle）

传输格式：`hand_gesture_pose_bundle` schema v1（仅 `hardware` 0~255）。落盘真相源仍是 `HAND_GESTURE_POSE_DIR`。

```bash
# 源机器导出自定义姿态
ros2 run hand_gesture_primitives pose_bundle_export -- \
  --hand-type g20 --hand-side left --output /tmp/left_g20.pose_bundle.json

# 源机器：预定义终态出厂包（离线，不需 gesture_node）
ros2 run hand_gesture_primitives export_primitive_poses -- \
  --hand-type l25 --hand-side left --set planable \
  --output /tmp/factory_l25_left.pose_bundle.json

# 目标机器导入（默认同名 skip）
export HAND_GESTURE_POSE_DIR="$HOME/hand-gesture-data/poses"
ros2 run hand_gesture_primitives pose_bundle_import -- \
  --input /tmp/left_g20.pose_bundle.json --strategy skip

# 旧 ~/.config 目录迁移到当前 HAND_GESTURE_POSE_DIR
ros2 run hand_gesture_primitives pose_bundle_import -- \
  --from-dir ~/.config/hand_gesture_primitives/poses --strategy skip --default-side left
```

`PlanGesture`（`/{side}_hand/plan_gesture`）仅做名义轨迹预览，可规划静态原语见
`planable_primitives.PLANABLE_STATIC_PRIMITIVES`。
vision/force 语义不可规划，须在满足感知/反馈条件后经 `gesture_cmd` 执行。

多关键帧 Pose 规划：`primitive` 传 `pose <name>` / `pose:<name>` / `pose/<name>` /
`named_pose:<name>` 时，按 MotionProgram 多段 AUTO 采样（要求 `semantic_v3|o20`，
禁止 `hardware`）。
响应新增 `segments_json` 字段（JSON 数组 `[{name, start_index}]`），标记各段在
`positions` 中的起始采样点索引，供 GUI 分段回放/高亮。

### Teach Web 面板

```bash
# 先按 ../../docs/getting-started.md 创建独立 Jazzy venv 并加载其 overlay。
# Pydantic 2、Web 依赖与 colcon 入口必须使用同一解释器。
export PYTHONNOUSERSITE=1
# gesture_node 已在跑；默认 ros2_control_command_units:=radians，须传 robot_description
ros2 launch hand_gesture_primitives teach_panel.launch.py \
  hand_side:=left hand_type:=g20 \
  ros2_control_command_units:=radians \
  g20_joint_order:=l25_urdf \
  robot_description:="..."
# 浏览器打开 http://127.0.0.1:8001/（右手 8002）
```

面板内部始终 hardware 0–255；`cmd_backend`/`ros2_control_command_units` 与 gesture_node 对齐（含 MOCK radians）。

**跨机迁移（面板）**：姿态库侧栏提供「导出下载 / 导入文件 / 导入出厂预设」：
- `GET /api/pose_bundle/export` → 下载当前手型/侧 `*.pose_bundle.json`
- `POST /api/pose_bundle/import?strategy=skip|overwrite|rename&verify_checksum=true`（body=bundle JSON）→ 合并入库；过滤后 0 导入返回 HTTP 409
- `POST /api/pose_bundle/import_factory?overwrite=false` → 写入 `preset_*` 静态终态

构建和消息依赖按[安装指南](../../docs/getting-started.md)准备；运行面板时加载同一 ROS overlay。

**L25 指尖**：与指中机械耦合，且 ros2_control 无 tip 主动关节；Teach 面板将 tip 标为**耦合/只读**（灰底），拖指中时 tip 自动跟随；保存/下发前服务端也会强制 tip=mid。

### 可用原语名

| 类别 | 指令 |
|------|------|
| 安全/复位 | `init` `open` `release` `relax_grip` |
| 固定手势 | `fist` `pinch` `point` `ok_sign` `v_sign` `tripod`（默认 auto，prep 才等 close）、`large_wrap`（O6/L6，支持 prep/close/auto） |
| O6 专用 | `hammer_grip` |
| 精细捏取 | `index_pinch` `middle_pinch` |
| 感知自适应 | `index_ring_by_vision` `large_wrap_by_vision` |
| 侧向夹持 | `thumb_adduction_grip` `index_middle_adduction_grip` |
| 力控包络 | `middle_ring` `ring` `small_warp` `no_index_warp` `hook` `palmar` `tripod_by_force` |
| 定位捏取/包络 | `index_pinch` `middle_pinch` `parallel_extension` `disk` |

### 状态反馈格式

`/{side}_hand/gesture_status` 发布格式（`|` 分隔）：

```
<active_primitive>|mode=<primitive|teach|stream>
<active_primitive>|mode=...|grasp=<grasp_state>
<active_primitive>|mode=...|type=<grasp_type>|suggest=<suggested_primitive>
rejected:<reason>
```

示例：
```
index_ring|mode=primitive|grasp=progressive
hold|mode=teach
hold|mode=primitive|type=precision|suggest=index_ring
rejected:too_far:0.312m>(max:0.150m)
```

`/{side}_hand/gesture_result` 发布格式：

```
IDLE                              # 无活跃原语
EXECUTING: <name>                 # 正在执行
EXECUTING: <name>, obj=<label>    # 执行中，有目标物体
DONE: <name>                      # 原语完成
```

---

## 消息格式详解

### JointState 控制指令

发布到 `/cb_{side}_hand_control_cmd`：

```yaml
header:
  stamp: <当前时间>
name: [thumb_base, index_base, middle_base, ring_base, pinky_base,
       thumb_abd, index_abd, middle_abd, ring_abd, pinky_abd,
       thumb_rot, reserved_11, reserved_12, reserved_13, reserved_14,
       thumb_tip, index_tip, middle_tip, ring_tip, pinky_tip]
position: [0.0 ~ 255.0] × 20    # O20: 0=伸直, 255=弯曲 (L25 反转)
velocity: [0.0] × 20
effort: [0.0] × 20
```

### hand_info JSON 格式

订阅 `/cb_{side}_hand_info`（驱动端发布）：

```json
{
  "current_current": [120, 85, 92, 78, 65, 150, 0, 0, 0, 0, 200, 0, 0, 0, 0, 95, 88, 72, 60, 55],
  "temperature": [35, 34, 36, ...]
}
```

`current_current` 数组长度 = num_joints，单位 mA。

### Float32MultiArray 指尖位置

发布到 `/cb_{side}_hand_fingertip_positions`：

```yaml
layout:
  dim:
    - label: "finger"
      size: 5
      stride: 15
    - label: "xyz"
      size: 3
      stride: 3
data: [x0, y0, z0, x1, y1, z1, ..., x4, y4, z4]  # 15 floats
# 顺序: thumb, index, middle, ring, pinky
# 坐标系: hand_base_link, 单位: 米
```

### matrix_touch_mass JSON 格式

订阅 `/cb_{side}_hand_matrix_touch_mass`：

```json
{
  "thumb_mass": 12.5,
  "index_mass": 8.3,
  "middle_mass": 0.0,
  "ring_mass": 0.0,
  "little_mass": 0.0
}
```

单位：克 (g)。作为触觉信号的 fallback 来源（仅在 `hand_force` 无数据时使用）。

---

## Launch 使用

### 基础启动（单侧）

```bash
ros2 launch hand_gesture_primitives gesture_node.launch.py \
  hand_side:=left hand_type:=o20
# 指令发到 /left_hand/gesture_cmd
```

### 启用门控

```bash
ros2 launch hand_gesture_primitives gesture_node.launch.py \
  hand_side:=left launch_gate:=true
# 用户 → /left_hand/gesture_cmd → gate → /left_hand/gesture_cmd_exec → node
```

### 双手同时

```bash
ros2 launch hand_gesture_primitives bimanual_gesture.launch.py \
  hand_type:=l25 launch_gate:=false
# 左右各自 /left_hand/... 与 /right_hand/...
```

### 带 Mock 感知测试

```bash
ros2 launch hand_gesture_primitives test_with_mock.launch.py \
  hand_side:=left shape:=cylinder
```

### RViz 手模型可视化

`hand_model` 选择手型 URDF；`hand_joint` 仅 **live** 模式用于 FK 映射。
L25/L6 等未随包提供的模型须按[模型说明](../../data/urdf/README.md)配置外部 URDF 与 mesh。
live 若二者不一致，launch 会把 URDF 对齐到 `hand_joint`。

```bash
# GUI：滑条拖动（默认 L25）
ros2 launch hand_gesture_primitives view_hand_urdf.launch.py \
  hand_side:=left hand_model:=L25

# GUI：L6 官方 mesh
ros2 launch hand_gesture_primitives view_hand_urdf.launch.py \
  hand_side:=left hand_model:=L6

# Live：真机/驱动反馈驱动模型（验证 FK）；建议 hand_model 与 hand_joint 同设
ros2 launch hand_gesture_primitives view_hand_urdf.launch.py \
  mode:=live hand_side:=left hand_model:=L6 hand_joint:=L6

# 无头：只起 RSP + joint_state_bridge
ros2 launch hand_gesture_primitives view_hand_urdf.launch.py \
  mode:=live hand_joint:=L6 launch_rviz:=false
```

---

## 典型集成拓扑

### 最小配置（无感知）

```
LinkerHand 驱动
  ├── /joint_states ────────────────────▶ {side}_hand_gesture_node
  ◀── /{side}_hand_controller/commands ─┘
```

仅支持安全/固定类原语（`init`、`open`、`fist`、`pinch` 等）。

### 标准配置（感知 + 门控）

```
robot_perception                    机械臂驱动
  ├── bboxes_3d ──▶ {side}_grasp_gate ◀── /{side}_hand/tcp_pose
  │                     │
  │                     ▼ /{side}_hand/gesture_cmd_exec
  │              {side}_hand_gesture_node
  │                     │
  │                     ▼
  │              /{side}_hand_controller/commands
  │
  └── bboxes_3d ──▶ gesture_node (自适应闭合)
```

支持全部包络/力控原语。

### 最小感知配置（无门控）

```bash
# 无门控时默认订阅 /{side}_hand/gesture_cmd
ros2 launch hand_gesture_primitives gesture_node.launch.py \
  hand_side:=left
ros2 topic pub --once /left_hand/gesture_cmd std_msgs/String "data: 'open'"
```

跳过门控，所有指令直达 gesture_node。包络原语不经可行性过滤。

---

## 保护机制

| 机制 | 触发条件 | 行为 |
|------|---------|------|
| 电流过载保护 | 任一关节电流 > `overload_current_threshold` 持续 > `overload_duration_sec` | 自动执行 `init`（全手复位） |
| 可达性检查 | TCP→目标距离 > `reach_max_distance` | 严格模式拒绝 / 非严格模式警告 |
| infeasible 保持 | 原语返回 `feasible=false` | 保持上帧目标继续按 `control_rate` 发送 |
| 感知超时 | bbox 连续 15 帧无更新 (~3s) | 清除物体缓存（不影响当前原语执行） |
| 门控去抖 | 连续 `de_bounce` 帧三条件通过 | 才转发指令（防抖动，仅 11 个门控原语） |

---

## 与 LinkerHand 驱动的对接

本包依赖 LinkerHand ROS2 SDK 提供的以下 Topic：

| 驱动 Topic | 方向 | 说明 |
|-----------|------|------|
| `/cb_{side}_hand_control_cmd` | 本包 → 驱动 | 20/25-DOF 目标角度 |
| `/cb_{side}_hand_state` | 驱动 → 本包 | 关节角度反馈 |
| `/cb_{side}_hand_info` | 驱动 → 本包 | 电流/温度 JSON |
| `/cb_{side}_hand_force` | 驱动 → 本包 | 触觉压感 |
| `/cb_{side}_hand_normal_force` | 驱动 → 本包 | 法向力 |
| `/cb_{side}_hand_matrix_touch_mass` | 驱动 → 本包 | 矩阵压感合值 |
| `/cb_{side}_hand_motor_torque` | 驱动 → 本包 | 关节力矩 |

兼容 LinkerHand O20 / L25 / O6 / L6 驱动，通过 `hand_type` 参数自动适配关节数量与角度方向。
L6 与 O6 同为 6-DOF 欠驱动布局；当前 C++ SDK 无真实电流（`capabilities.current=false`），力控以触觉优先；`hand_type:=l6`。
硬件驱动按设备厂商提供的部署说明启动，再将其反馈和目标话题与上述接口对接。

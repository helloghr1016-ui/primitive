> 历史记录：本文保留 2026-09 的站点调试流程和实测结果，其中硬件工作区、旧姿态路径与构建方式仅描述当时环境。新安装请先阅读[安装与启动指南](getting-started.md)；0.2.2 的验收范围见[发布说明](releases/0.2.2.md)。命令中的 `~/primitive` 按公开克隆目录举例，硬件路径需按实际部署调整。

# O6 右手链路调试 Runbook（发布前验证）

适用范围：`hand_gesture_primitives` 作为**独立手势原语包发布前**，基于 O6 右手链路的验证。
硬件 SDK 与驱动保留在 `luban_framework`，本仓只发布手势原语。

- 真机侧别：**right**
- 本流程**不接**视觉感知、相机、`robot_perception`、GraspGate
- 不测 `*_by_vision`、依赖 TCP / object pose / 3D bbox 的原语，以及需要感知给出 `grasp_type` 的全局 auto

---

## 0. 调试会话记录头

每次调试前填写，便于回溯：

| 项                      | 值                         |
| ----------------------- | -------------------------- |
| 日期                    | 2026.09.17                 |
| commit hash             |                            |
| 分支                    | `debug/o6-gesture-local` |
| `o6.yaml` 版本 / 哈希 |                            |
| 启动参数                |                            |
| `cmd_backend`         |                            |
| 控制单位                |                            |
| `control_rate`        |                            |
| 反馈来源                | MOCK / 真机                |

> 记录头对应的现场数据（`/joint_states`、`/right_hand_controller/commands`、`gesture_status`、`gesture_result`）建议一并抓包留存。

---

## 1. 保护工作区

```bash
cd ~/primitive
git status --short

# 历史调试时该模型未跟踪；当前版本已随仓库交付，现场修改仍可单独备份
cp -r data/urdf/o6/ /tmp/o6-urdf-backup/
git diff > /tmp/o6-gesture-before-pull.patch
```

- 当前分支已是 `debug/o6-gesture-local`，不要再用 `git switch -c` 创建同名分支。
- 调试期间**不执行 `git pull`**。

---

## 2. 纯软件回归

```bash
cd ~/primitive
colcon build --merge-install --packages-select \
  hand_gesture_primitives_msgs hand_force_msg robot_perception_msgs hand_gesture_primitives
source install/setup.bash

cd hand_gesture_primitives
pytest -q
ruff check hand_gesture_primitives test
```

**基线（2026-09-17 实测）**：`366 passed, 1 skipped`；ruff 无告警。

唯一跳过项是 `test_o30_support.py` 里「O30 URDF 未部署」，与 O6 无关。

注意：

- `--merge-install` 是因为本机 `install/` 目录由 merged layout 创建。**全新 workspace 不需要**该选项（README 的通用编译说明没有它）。
- 必须同时选中另外三个消息包，否则报 `Failed to find .../package.sh`。
- `install/` 被删除重建后**必须重新 source**，否则报 `ModuleNotFoundError: hand_gesture_primitives`。
- pytest 必须在 `hand_gesture_primitives/` 目录下执行（pytest 配置在该包的 `setup.cfg`）。

---

## 3. 构建硬件侧（luban 最小子集）

`linkerhand_hardware` 不在本仓，位于 `~/luban_framework/src/system/hardware/`。该 workspace **没有 `install/`**，需先构建。

### 3.1 环境前置（apt 系统包）

```bash
sudo apt install -y ros-jazzy-ros2-control ros-jazzy-ros2-controllers \
                    ros-jazzy-xacro ros-jazzy-pinocchio
```

- `ros2_control`（含 `controller_manager`、`hardware_interface`）：`control` 与 `linkerhand_hardware` 编译需要，launch 启动 `controller_manager` 也需要。缺它时 `control` 报 `find_package(controller_manager)` 失败。
- `xacro`：launch 顶层 `import xacro`，缺失时 launch 直接抛 `ModuleNotFoundError`。
- `pinocchio`：**可选**。缺它手势原语照常执行，仅 FK 降级——指尖坐标话题不发布、`joint_state_bridge` 启动即退出。注意 import 名是 `pinocchio`，不是 `pin`。

### 3.2 构建（必须限定 `--base-paths`）

```bash
cd ~/luban_framework
source /opt/ros/jazzy/setup.bash
source ~/primitive/install/setup.bash   # 提供 hand_force_msg 等依赖

colcon build --merge-install \
  --base-paths src/system src/robot_model src/control \
  --packages-select \
    linkerhand_hardware linkerhand_sdk_vendor luban_robot_model control \
    ar508_l20_workstation_description competition_device_description env_description \
    l20_left_description l20_right_description l25_left_description l25_right_description \
    l6_left_description l6_right_description o20_left_description o20_right_description \
    o30_left_description o30_right_description o6_left_description o6_right_description \
    "AR5-5_08L-W4C4A6-ZY2_description" "AR5-5_08R-W4C4A6-ZY2_description"
```

**为什么必须加 `--base-paths`**（不加则构建失败）：

- `src/perception/config/AR5-5_08/` 下有一份 `*_description20260511` 旧备份，与 `src/robot_model/evo_work/` 下的正式包**重名**，colcon 直接拒绝：`Duplicate package names not supported`。限定 base-paths 后 colcon 看不到那份备份，且**不修改 luban 任何文件**。
- 同时排除 `src/robotskill_framework/`，避开 luban 里那份**迁移前**的 `hand_gesture_primitives` 副本，防止它被编进 luban 的 install。

其余说明：

- `control` 提供 `controllers_o6.yaml`；`luban_robot_model` 提供 `o6_hand_only.urdf.xacro`。缺任一个，launch 在 `get_package_share_directory` 处即报错。
- `linkerhand_sdk_vendor` 是 vendor 包，若需联网拉取 SDK 源码可能在此卡住。

### 3.3 改 luban 配置后的同步坑

launch 通过 `get_package_share_directory("control")` 读的是 **`install/share/control/config/`** 里的副本。改 `config/control/*.yaml` 源码**不会生效**，且没有任何报错，只表现为"改了没用"。必须二选一：

```bash
# 快：直接同步到 install
cp ~/luban_framework/config/control/controllers_o6.yaml \
   ~/luban_framework/install/share/control/config/controllers_o6.yaml

# 稳妥：重新构建该包
colcon build --merge-install --packages-select control
```

调 O6 的接触/力矩阈值时会反复踩这个坑。

---

## 4. overlay 顺序（每个新终端都要确认）

两仓存在 **5 个重名包**：`hand_gesture_primitives`、`hand_gesture_primitives_msgs`、`hand_force_msg`、`robot_perception_msgs`、`robot_perception`。

luban 中那份 `hand_gesture_primitives` 是**迁移前**的旧代码，source 顺序错误会导致整套测试跑在旧实现上。

```bash
export ROS_DOMAIN_ID=77

source /opt/ros/jazzy/setup.bash
source ~/luban_framework/install/setup.bash
source ~/primitive/install/setup.bash   # 必须最后

# 校验：输出必须指向 release 仓
python3 -c "import hand_gesture_primitives as m; print(m.__file__)"
```

**`ROS_DOMAIN_ID=77` 不是可选项。** 默认域（0）里还有**另一台机器**在跑底盘/升降栈（`ds2_lift_driver`、`water_chassis_driver`），它的 `robot_state_publisher` 同样往 `/robot_description` 发消息（整机 URDF）。该话题是 TRANSIENT_LOCAL（锁存），订阅者拿到的可能是**对方**那份——里面没有 `r_hand_*` 关节。具体后果见 §5.1。

给验证环境一个独立域是最干净的隔离手段，且不引入任何非发布版的特殊配置。

确认隔离生效：

```bash
ros2 node list
```

应该只有 6 个节点：`controller_manager`、`joint_state_broadcaster`、`o6_hand_right`、`right_hand_controller`、`right_hand_gesture_node`、`robot_state_publisher`。出现 `ds2_lift_driver` / `water_chassis_driver`，或 `robot_state_publisher` 有两个，就说明隔离没生效（检查各终端是否都 export 了同一个域号）。

---

## 5. MOCK 打通 ROS 链路

```bash
MOCK=1 GESTURE=1 GESTURE_SIDE=right \
ros2 launch linkerhand_hardware launch_o6_hand.py
```

启动日志确认：`mock=True`、`hw_sides=['right']`、`gesture_sides=['right']`。

```bash
ros2 topic list | rg 'right_hand|joint_states|cb_right'
ros2 topic echo /joint_states
ros2 topic echo /right_hand/gesture_status
ros2 topic echo /right_hand/gesture_result
ros2 topic echo /right_hand_controller/commands
```

基础指令：export ROS_DOMAIN_ID=77

```bash
ros2 run hand_gesture_primitives pub_gesture_cmd -- right "mode primitive"
ros2 run hand_gesture_primitives pub_gesture_cmd -- right open
ros2 run hand_gesture_primitives pub_gesture_cmd -- right init
```

**验收**：命令被接受；输出为 6 个 O6 控制量；`joint_states` 跟随；`gesture_result` 出现 EXECUTING → DONE。

### 5.1 两个会浪费半天的坑

**1) `/joint_states` 是空消息（`name: []`）**

症状：JSB 日志一切正常（`Configured and activated joint_state_broadcaster`），也在 100Hz 发布，但消息里没有任何关节名和值。

根因：跨机 ROS 域污染（见 §4）。JSB 用 `/robot_description` 的 URDF 过滤状态接口（`use_urdf_to_filter` 默认 `true`），拿到对方整机 URDF 后，把 18 个 `r_hand_*` 接口**全部滤掉**。

排查顺序（前两条能一刀切开责任面）：

```bash
ros2 control list_controllers -v               # JSB 的 claimed interfaces 是否为空
ros2 topic echo /dynamic_joint_states --once   # 有 6 关节值 = 硬件侧正常，问题只在 JSB
ros2 topic info /robot_description --verbose   # Publisher count > 1 = 域里有别的机器
ros2 node list                                 # 出现非本栈节点即确认污染
```

正确修法是**设 `ROS_DOMAIN_ID` 隔离**（§4）。不要改 `use_urdf_to_filter`——那只是让 JSB 不读 URDF，绕过症状；其他读 `/robot_description` / `/tf` 的组件仍会被污染，而且这类撞名是竞态，会表现为"时好时坏"，比稳定故障更难查。

**2) `gesture_result` 看起来"没有输出"**

`executor.py` 的 `_publish_result` **只在文本变化时发布**：

```python
if msg_text != self._last_result_msg:
    self._result_pub.publish(result_msg)
```

状态稳定在 `DONE: xxx` 时话题就是静默的，不是坏了。验证时必须**先挂订阅、再发命令**制造状态变化：

```bash
timeout 18 ros2 topic echo /right_hand/gesture_result > /tmp/gr.txt 2>&1 &
sleep 3
ros2 run hand_gesture_primitives pub_gesture_cmd -- right "init"
sleep 12; cat /tmp/gr.txt
```

直接 `ros2 topic echo ... --once` 打稳态抓不到东西，容易误判成功能故障。

---

## 6. 无感知原语

**先 dry-run 确认 tag 过滤选中了哪些原语**（避免 auto 变体被漏掉，或把依赖感知的全局 auto 带进来）：

```bash
ros2 run hand_gesture_primitives run_gesture_cmd_suite -- right \
  --hand-type o6 --tags setup,safe,static,focus,phased,force --dry-run
```

确认 `tripod auto`、`large_wrap auto` 在列表内且无 vision 项后，去掉 `--dry-run` 实跑（保留 `--pause` 逐步确认）：

```bash
ros2 run hand_gesture_primitives run_gesture_cmd_suite -- right \
  --hand-type o6 --tags setup,safe,static,focus,phased,force --pause
```

覆盖原语：

`init` `open` `release` `relax_grip` `fist` `pinch` `index_pinch` `middle_pinch` `point` `ok_sign` `v_sign` `tripod` `large_wrap` `hammer_grip` `thumb_adduction_grip` `ring` `middle_ring` `tripod_by_force` `named_pose`

**套件覆盖不到、需手动补的三条**（`DEFAULT_SUITE` 里没有定义）：

```bash
ros2 run hand_gesture_primitives pub_gesture_cmd -- right "hammer_grip"   # 套件只有 prep/close
ros2 run hand_gesture_primitives pub_gesture_cmd -- right "tripod auto"   # 套件只有 large_wrap auto，无 tripod auto
# named_pose 留到 §10 存完姿态后再测
```

**tag 归属**（容易漏）：`ring` / `middle_ring` 的 tag 是 **`force`**，不加会被静默排除；`tripod_by_force` 是 `phased` 而不是 `force`。

**MOCK 下的预期**：`ring` / `middle_ring` 属 force 原语，而 O6 的停指判定只认压感（`contact_detection.py` 里 `uses_torque_feedback` 分支直接 `return False`）。MOCK 无触觉源，这两条会一路闭到底、不中途停——**属正常行为**，逐指停止只能在 §8 真机带软物体时验证。

**跳过**：`*_by_vision`、需要感知提供 `grasp_type` 的全局 auto、GraspGate。

`--tags` 可选值：`setup,safe,static,focus,phased,force,vision,motion`。
`--include-unsupported` 用于验证节点对不支持原语的拒绝路径。

### 原语逐条记录表

> **本轮数据来源**：2026-09-17 13:45–13:48 MOCK 实跑，`MOCK=1 GESTURE=1 GESTURE_SIDE=right`、`ROS_DOMAIN_ID=77`。
> 开始时间取自 gesture_node 日志的 `原语切换` / `激活原语` 行；`完成` 依据 `/right_hand/gesture_result` 的 `DONE`。
> **`最终关节状态` 与 `gesture_status` 两列未采集**——当时 `/joint_states` 订阅用了 `--field position`，丢掉了 `header.stamp`，无法按时间对齐到具体原语。补采方法见表格下方。

| 命令                       | 接受    | 开始时间 | 完成 | 最终关节状态 | gesture_status | gesture_result      | 触发保护 | 异常现象                                                                                            |
| -------------------------- | ------- | -------- | ---- | ------------ | -------------- | ------------------- | -------- | --------------------------------------------------------------------------------------------------- |
| init                       | ✅      | 13:45:58 | ✅   | 未采集       | 未采集         | `EXECUTING→DONE` | 无       | —                                                                                                  |
| open                       | ✅      | 13:46:02 | ✅   | 未采集       | 未采集         | `EXECUTING→DONE` | 无       | —                                                                                                  |
| release                    | ✅      | 13:46:05 | ✅   | 未采集       | 未采集         | `EXECUTING→DONE` | 无       | —                                                                                                  |
| relax_grip                 | ✅      | 13:46:11 | ✅   | 未采集       | 未采集         | `EXECUTING→DONE` | 无       | **重复 `DONE`**（见下方说明）                                                               |
| fist                       | ✅      | 13:46:16 | ✅   | 未采集       | 未采集         | `EXECUTING→DONE` | 无       | —                                                                                                  |
| pinch                      | ✅      | 13:46:20 | ✅   | 未采集       | 未采集         | `EXECUTING→DONE` | 无       | —                                                                                                  |
| index_pinch                | ✅      | 13:46:26 | ✅   | 未采集       | 未采集         | `EXECUTING→DONE` | 无       | —                                                                                                  |
| middle_pinch               | ✅      | 13:46:31 | ✅   | 未采集       | 未采集         | `EXECUTING→DONE` | 无       | —                                                                                                  |
| ok_sign                    | ✅      | 13:46:35 | ✅   | 未采集       | 未采集         | `EXECUTING→DONE` | 无       | —                                                                                                  |
| point                      | ✅      | 13:46:39 | ✅   | 未采集       | 未采集         | `EXECUTING→DONE` | 无       | —                                                                                                  |
| v_sign                     | ✅      | 13:46:43 | ✅   | 未采集       | 未采集         | `EXECUTING→DONE` | 无       | —                                                                                                  |
| large_wrap                 | ✅      | 13:46:46 | ✅   | 未采集       | 未采集         | `EXECUTING→DONE` | 无       | —                                                                                                  |
| large_wrap prep            | ✅      | 13:46:51 | ✅   | 未采集       | 未采集         | `EXECUTING→DONE` | 无       | —                                                                                                  |
| large_wrap close           | ✅      | 13:46:54 | ✅   | 未采集       | 未采集         | `EXECUTING→DONE` | 无       | —                                                                                                  |
| large_wrap auto            | ✅      | 13:46:59 | ✅   | 未采集       | 未采集         | `EXECUTING→DONE` | 无       | —                                                                                                  |
| hammer_grip                | ✅      | 13:54:43 | ✅   | 未采集       | 未采集         | `EXECUTING→DONE` | 无       | 收敛未达标通道`thumb_mcp_abd` / `thumb_mcp_flex` / `index_mcp_flex`（误差 ≤5.0，rad→255eq） |
| hammer_grip prep           | ✅      | 13:47:04 | ✅   | 未采集       | 未采集         | `EXECUTING→DONE` | 无       | —                                                                                                  |
| hammer_grip close          | ✅      | 13:47:08 | ✅   | 未采集       | 未采集         | `EXECUTING→DONE` | 无       | —                                                                                                  |
| thumb_adduction_grip       | ✅      | 13:47:13 | ✅   | 未采集       | 未采集         | `EXECUTING→DONE` | 无       | —                                                                                                  |
| thumb_adduction_grip prep  | ✅      | 13:47:17 | ✅   | 未采集       | 未采集         | `EXECUTING→DONE` | 无       | —                                                                                                  |
| thumb_adduction_grip close | ✅      | 13:47:22 | ✅   | 未采集       | 未采集         | `EXECUTING→DONE` | 无       | —                                                                                                  |
| tripod                     | ✅      | 13:47:26 | ✅   | 未采集       | 未采集         | `EXECUTING→DONE` | 无       | —                                                                                                  |
| tripod prep                | ✅      | 13:47:30 | ✅   | 未采集       | 未采集         | `EXECUTING→DONE` | 无       | —                                                                                                  |
| tripod close               | ✅      | 13:47:34 | ✅   | 未采集       | 未采集         | `EXECUTING→DONE` | 无       | —                                                                                                  |
| tripod auto                | ✅      | 13:58:09 | ✅   | 未采集       | 未采集         | `EXECUTING→DONE` | 无       | 首次发送报"超时未见订阅者"但命令实际已送达（见下方说明）                                            |
| tripod_by_force            | ✅      | 13:47:38 | ✅   | 未采集       | 未采集         | `EXECUTING→DONE` | 无       | —                                                                                                  |
| tripod_by_force prep       | ✅      | 13:47:45 | ✅   | 未采集       | 未采集         | `EXECUTING→DONE` | 无       | —                                                                                                  |
| tripod_by_force close      | ✅      | 13:47:49 | ✅   | 未采集       | 未采集         | `EXECUTING→DONE` | 无       | —                                                                                                  |
| ring                       | ✅      | 13:47:55 | ✅   | 未采集       | 未采集         | `EXECUTING→DONE` | 无       | MOCK 无力反馈，不停指属预期                                                                         |
| middle_ring                | ✅      | 13:48:04 | ✅   | 未采集       | 未采集         | `EXECUTING→DONE` | 无       | MOCK 无力反馈，不停指属预期                                                                         |
| open（归位）               | ✅      | 13:48:13 | ✅   | 未采集       | 未采集         | `EXECUTING→DONE` | 无       | —                                                                                                  |
| named_pose                 | ⬜ 未测 | —       | —   | —           | —             | —                  | —       | 不在`DEFAULT_SUITE`，需先 `save_pose`（见 §10）                                                |

**汇总**：33 条中 32 条实跑通过（全部 `EXECUTING→DONE`，无一卡死），1 条待补测（`named_pose`，需先 `save_pose`，见 §10）。

**`pub_gesture_cmd` 的"超时未见订阅者"可能是误报**：`--timeout` 默认仅 **3.0 秒**，节点忙碌时容易等不到 DDS 发现。但实测出现过**报了超时、命令却已送达节点**的情况（节点日志 13:57:24 有 `收到指令: 'tripod auto'`）。所以看到这条报错先别断定"没发出去"——查节点日志的 `收到指令` 行确认，或直接加大超时重发：

```bash
ros2 run hand_gesture_primitives pub_gesture_cmd -- right "tripod auto" --timeout 30
```另 `grasp_ready`、`parallel_extension`、`index_middle_adduction_grip`(含 prep/close) 共 5 步被 allowlist 正确拒绝（O6 缺指侧摆/独立指尖旋转），未发布到话题。

**`relax_grip` 重复 `DONE` 的成因**：完成判定有两条路径。`_publish_result` 的反馈收敛分支（`_feedback_command_converged`）先发一次；随后主循环在 `primitive.done` 为真时再调 `publish_user_result`（`executor.py:1600`，**无条件发布、不去重**）又发一次。文本相同、幂等，不影响功能；但下游若按"DONE 条数"计数需知悉。

**补采 `最终关节状态` / `gesture_status` 两列**：订阅时**不要用 `--field`**，抓完整消息以保留 `header.stamp`：

```bash
ros2 topic echo /joint_states > /tmp/suite_joints.log 2>&1 &          # 含 header.stamp
ros2 topic echo /right_hand/gesture_status > /tmp/suite_status.log 2>&1 &
```

再用时间戳与上表的开始时间对齐。`gesture_status` 只在状态变化时发布，需在开跑前先挂订阅。

---

## 7. 分阶段动作

```bash
ros2 run hand_gesture_primitives pub_gesture_cmd -- right "large_wrap prep"
ros2 run hand_gesture_primitives pub_gesture_cmd -- right "large_wrap close"

ros2 run hand_gesture_primitives pub_gesture_cmd -- right "hammer_grip prep"
ros2 run hand_gesture_primitives pub_gesture_cmd -- right "hammer_grip close"

ros2 run hand_gesture_primitives pub_gesture_cmd -- right "tripod prep"
ros2 run hand_gesture_primitives pub_gesture_cmd -- right "tripod close"
```

**验收**：

- prep 后进入 ready 或保持态
- close 只执行后半段动作
- auto 能自动串联
- 中途重复发命令不造成目标跳变
- `release` / `relax_grip` 能恢复到安全状态
- `stop` 能中止当前原语并**就地保持**（注意 `stop` 不是回安全位，语义见 §7.1）

### 7.1 实测结果（MOCK，2026-09-17）

用连续 `/joint_states` 轨迹判定，不依赖目视：

| 验收项               | 结果 | 证据                                                                                                                                                             |
| -------------------- | ---- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| prep 后保持态        | ✅   | prep 结束后连续**1134 帧位姿完全不变**，持续约 11.3s，直到 close 命令到达                                                                                  |
| close 只跑后半段     | ✅   | close 前最后一帧与 prep 结束位姿**逐位相等**（`[0.3386×4, 0.0, 1.57]`），无跳回起始位姿；close 结束于全闭合位姿 `[1.57×6]`                           |
| 中途重复发命令不跳变 | ✅   | prep 完成后 1s 重发 prep，运动平滑起步（首个台阶仅 0.077 rad），全程无异常跳变                                                                                   |
| auto 自动串联        | ✅   | 见 §6：`large_wrap auto`、`tripod auto` 均 `EXECUTING→DONE`                                                                                              |
| `stop` 中止并保持  | ✅   | `ring` 跑 4s 后发 `stop`，节点日志 `停止原语: ring`；其后 2 秒内**位姿完全冻结**（仅 1 种位形），停在中断处 `[1.57, 1.57, 0.0, 0.0, 0.536, 0.585]` |

**`stop` 的语义**（`executor.py:863`）：`"""停止原语，保持当前姿态继续发送。"""` —— **中止当前原语并就地冻结**，不回安全位；要回安全位请用 `release` / `relax_grip`。`stop` 的分支在 allowlist 检查**之前**处理，任何手型都可用。

**`DONE` 的含义**：表示**反馈收敛**（`_feedback_command_converged`），**不是"动作序列执行完毕"**。所以 `ring` 能在启动约 3~4 秒后就报 `DONE`（mock 下反馈跟得上），而 `stop` 打断后报的是 `IDLE`。批量判读时不要把 `DONE` 当作"整段动作走完"。

**判读要点（避免把量化当故障）**：mock 硬件"保持上一帧指令"，而指令下发 10Hz、状态读取 100Hz，所以轨迹天然呈**台阶状**。单个台阶 0.31 rad 属正常量化粒度——判据是**邻近台阶是否同量级**（实测 0.312 / 0.313 / 0.317 / 0.314，同量级 ⇒ 匀速段，非跳变）。只有某一步显著大于邻近台阶，才是真正的目标跳变。

**另一个坑**：`/joint_states` 的关节名顺序在**单次进程内稳定**，但**跨进程重启会变**（JSB 内部用无序容器）。逐列比较关节值前，必须先确认该次日志里 `name` 数组的实际顺序，否则会把换序误读成跳变：

```bash
awk '/^name:/{inn=1;n=0;ord="";next} inn && /^- /{ord=ord $2" ";n++;if(n==6){print ord;inn=0}}' /tmp/joints.log | sort | uniq -c
```

若输出只有一行，说明该次运行顺序稳定，逐列分析有效。

---

## 8. 力控反馈

`o6.yaml` 配置：`touch: true`、`current: false`、`motor_torque: true`。

```bash
ros2 topic echo /cb_right_hand_force          # 压感
ros2 topic echo /cb_right_hand_normal_force
ros2 topic echo /cb_right_hand_motor_torque   # 0~100%
```

**实现要点（决定观察重点）**：

- **O6 的停指判定只认压感**：`pinch_motion_should_stop` / `pinch_motion_should_stop_toque` 在 `uses_torque_feedback(o6)` 为真时直接 `return False`，不走力矩阈值。
- 「是否继续加力闭合」(`feedback_blocks_motion`) 与 `tripod_force` 使用 `torque_delta_pct`（默认 30%）。
- 原因见 `config/o6.yaml:349` 与 `contact_detection.py:230`：
  > O6 motor_torque 空载收指常态 60~80%，绝对值 / Δ 均会误触
  >

因此 **MOCK 阶段只能验证话题连通与数值解析**；误触发与逐指停止必须在真机、带软物体或受控治具时才有结论。

```bash
ros2 run hand_gesture_primitives pub_gesture_cmd -- right "tripod_by_force prep"
ros2 run hand_gesture_primitives pub_gesture_cmd -- right "tripod_by_force close"
ros2 run hand_gesture_primitives pub_gesture_cmd -- right ring
ros2 run hand_gesture_primitives pub_gesture_cmd -- right middle_ring
```

**重点记录**：

- 是否误触发接触检测
- 是否能逐指停止
- 是否超过 `hold_safe_torque_pct`
- 无触觉时是持续闭合还是安全保持
- 是否触发过载保护

> **不要**把 O6 的力矩数值当电流参数调节。`uses_torque_feedback` 仅 O6 为真（L6 关节数同为 6 但走触觉），torque% 与 mA 不可混用。

---

### 8.1 真机阶段待确认（MOCK 遗留）

MOCK 实跑时在 gesture_node 日志里观察到两条与预期不符的记录，需在真机接通力反馈后确认：

**1) 停指条件不止"接触触发"**

```
[middle_ring] grasp=contact | feedback=current tactile=none(current-fallback) mode=none contact=False stopped=['middle', 'thumb']
```

`contact=False` 却出现 `stopped=['middle','thumb']`。需确认 `stopped` 的确切语义——是"真的停止闭合"，还是仅表示"纳入停止监测的手指"。这直接关系到 §8 里"是否能逐指停止"这条验收怎么判定。

**2) 反馈源回落到了 current**

日志显示 `feedback=current ... (current-fallback)`，而 O6 的配置是 `motor_torque: true`、`current: false`。MOCK 下没有真实力矩源可以理解，但真机接通后要确认它**确实切到 `motor_torque`**——否则本节所有阈值验证都是在错误的信号源上做的。

---

## 9. 切换真机

### 9.0 上电前的安全前置

- **手部悬空、无人接触**，周围无障碍物，操作者手边能随时切断电源
- ⚠️ **launch 一启动手就会动**：gesture_node 启动时会**自动激活 `init`** 并立即下发目标角度（MOCK 阶段日志里的 `激活原语: init`）。也就是说"启动"不等于"等你发命令"，上电前必须确认手是空的、姿态安全的
- 首轮全程空载，不要抓任何东西
- **控制单位无需猜测**：launch 中 `cmd_backend="ros2_control"`、`ros2_control_command_units="radians"` 是硬编码默认值，不存在 0~255 直发弧度控制器的风险。只有显式改用 `legacy_cb` + `hardware` 时才需另行确认

### 9.1 拉起并确认 CAN 通道

xacro 里 `can_bitrate: 1000000`（1 Mbps），`RIGHT_CAN` 默认 `can0`。

**接口默认是 DOWN 的，必须先拉起**（每次 USB 重插或机器重启后都要重做）：

```bash
sudo ip link set can0 type can bitrate 1000000
sudo ip link set can0 up

# 确认：应显示 state UP、bitrate 1000000
ip -details link show can0 | head -4
```

**本机实测（2026-09-17）**：`can0` 是 PCAN-USB 适配器（驱动 `pcan_usb`，`parentbus usb`），初始状态 `state DOWN` / `can state STOPPED`；**`can1` 不存在**——本机只有一个 CAN 口，右手接在 `can0`，与 `RIGHT_CAN` 默认值一致，无需覆盖。

想观察报文可装 can-utils（可选，拉起接口不需要它）：

```bash
sudo apt install -y can-utils
candump can0        # 另开终端跑；发手势命令时应能看到流量
```

注意：手空闲时不一定有周期报文，所以 `candump` 无输出**不能**证明链路故障；但**发命令时应该看到流量**——这是最直接的"CAN 真通了"的证据。

只有在接口被占用、或 bitrate 不匹配时才会出问题，表现是"命令发出但手不动"或通信报错，容易误判成软件问题。

### 9.2 启动（去掉 MOCK）

```bash
export ROS_DOMAIN_ID=77        # 必须，理由见 §4
source /opt/ros/jazzy/setup.bash
source ~/luban_framework/install/setup.bash
source ~/primitive/install/setup.bash

GESTURE=1 GESTURE_SIDE=right ros2 launch linkerhand_hardware launch_o6_hand.py
```

启动日志里**两行必须确认**：

```
[launch_o6_hand] mock=False can=R:can0/L:can1 hw_sides=['right'] gesture_sides=['right']
[controller_manager]: Loaded hardware 'o6_hand_right' from plugin 'linkerhand_hardware/LinkerHandHardware'
```

`mock` 必须是 `False`；插件必须是 `linkerhand_hardware/LinkerHandHardware`。若显示 `mock_components/GenericSystem`，说明 `MOCK=1` 没去掉。

### 9.3 首次验证顺序（从最安全的开始）

逐条发、逐条观察，每条之间确认 `/joint_states` 实际值与 `/right_hand_controller/commands` 指令一致：

1. `pub_gesture_cmd -- right "open"` —— 张开，行程最大、最安全
2. `-- right "init"` —— 回初始姿态
3. `-- right "fist"` / `"pinch"` —— 静态抓握位形
4. 分阶段：`"large_wrap prep"` → **确认停在中间位姿** → `"large_wrap close"`
5. 最后才碰力控：`"ring"` / `"middle_ring"` / `"tripod_by_force"`（带软物体，见 §8；触觉不通时见 §9.4 的说明）

**真机实测（2026-09-17）**：第 1~4 步 ✅ 全部通过。

- `open` / `init` / `fist` 动作与预期一致
- `large_wrap prep` 后手停在**预抓取中间位姿并保持不动**（拇指张开、四指半伸）
- `large_wrap close` **从该中间位姿继续闭合**，不是重新起手

节点日志佐证分阶段语义：prep 与 close 两次命令的切换记录为 `原语切换: large_wrap → large_wrap`，说明 close 是在 prep 的上下文里执行的，没有新起原语实例。

> 另观察到一次**未先 prep 直接发 `large_wrap close`**（前态为 `open`，日志 `原语切换: open → large_wrap`）。该路径本次未单独评估其动作正确性。

### 9.4 力反馈话题（MOCK 下不存在，真机才有）

真机实测（2026-09-17）：

| 话题                                 | 发布者 | 说明                                                                                            |
| ------------------------------------ | ------ | ----------------------------------------------------------------------------------------------- |
| `/cb_right_hand_force`             | ✅ 1   | **主触觉通路**（`hand_force_msg`，5 指 × 10 行 × 4 列 = 200 值，另有 `palm_force`） |
| `/cb_right_hand_info`              | ✅ 1   | 聚合信息                                                                                        |
| `/cb_right_hand_motor_torque`      | ✅ 1   | 6 关节力矩 0~100%                                                                               |
| `/cb_right_hand_normal_force`      | ❌ 0   | 降级备用，本驱动**不发布**                                                                |
| `/cb_right_hand_matrix_touch_mass` | ❌ 0   | 降级备用，本驱动**不发布**                                                                |

gesture_node 会订阅全部四个（`executor.py:388-415`），但**主通路是 `hand_force`**；`normal_force` / `matrix_touch_mass` 仅在 `hand_force_msg` 类型不可用时降级使用。**后两个没有发布者属正常，不是故障。**

空载静止时：`motor_torque` 接近 0（`o6.yaml` 里"空载 60~80%"指的是**收指过程中**的力矩，不是静止保持）；`hand_force` 全 0，正确。

⚠️ **`ros2 topic echo` 会截断长数组**：`/cb_right_hand_force` 每条 200 个值，默认只显示前 ~130 个、其余显示 `'...'`，极易误判成"消息是空的"。看全用 `--truncate-length 20000`。

**验证触觉是否真的响应**——下面这条只打印**非零**读数，静止时完全静默，按指尖时才有输出：

```bash
ros2 topic echo /cb_right_hand_force --truncate-length 20000 \
  | grep --line-buffered -E "^ *- [0-9]" | grep --line-buffered -vE "0\.0$"
```

按下去仍全程无输出，说明触觉通路有问题——那 §8 的 `tripod_by_force` / `ring` / `middle_ring` 就没有输入源，需先解决再往下走。

同时确认 §8.1 的两条待确认项：反馈源是否真的切到 `motor_torque`、`stopped` 的确切语义。

#### 实测发现：触觉数据全为零（2026-09-17 真机）

`/cb_right_hand_force` 以 **48.9 Hz 稳定发布**，但**单帧 200 个值全为 0**，用力按压指腹也无任何变化。

逐层排查结论：

1. **驱动侧正常**：`poll_and_publish_feedback()` 在跑，初始化日志 `force_feedback=on`
2. **调用成功**：`hand_api_->getForce()` 返回了结构正确的 5×10×4 矩阵（`mats.size() >= 5` ⇒ `got_force = true`）
3. **零值来自 SDK**：消息里的数值就是 `mats[i][j][k]` 原样拷贝（`linkerhand_hardware.cpp:647-649`），全 0 说明 **SDK 返回的就是全 0**
4. `getPalmForce()` 返回空 ⇒ `palm_force` 字段为空
5. **SDK 没有"开启触觉"类接口**（`LinkerHandApi.h` 只有读取方法 `getForce` / `getFingerForceSum` / `getNormalForce` 等），所以不是漏调初始化

⚠️ **`/cb_right_hand_info` 里的 `is_touch: true` / `touch_type: 2` 不能当作"该手有触觉"的证据**：驱动是用 `got_force` 算出来的（`linkerhand_hardware.cpp:553-554`），而 `got_force` 只表示"`getForce()` 返回了 ≥5 项"，**不代表有真实数据**。

SDK 支持两种触觉硬件（`TSSP_JZG` 掌 20×28 / `TSSP_HWK` 掌 14×16，五指合力每指 3 或 1 个值），说明触觉属**选配**硬件。

**结论**：该手（`Device Serial: LHO6-04.1-307-R-Z-1-E`）触觉在 SDK 层返回全零，是硬件/SDK 侧问题，与 `hand_gesture_primitives` 无关。**查清之前，§8 的 `tripod_by_force` / `ring` / `middle_ring` 力控验证无法进行**；但 §9.3 的分阶段动作、静态手势、§10 示教链路不受影响，可继续。

### 9.5 需要降速时

速度参数在 `hand_gesture_primitives/config/o6.yaml`：

- `speeds: [...]`（`two_stage_speed`，单位是 0-255 位置单位/秒，由 10Hz 控制循环解释）
- 各处 `duration` / `preshape_duration` / `grip_duration`
- `finger_step`（每 tick 增量，随 `control_rate` 自动缩放）

⚠️ 改完**必须同步到 install**，否则不生效（同 §3.3 的坑）：

```bash
cp ~/primitive/hand_gesture_primitives/hand_gesture_primitives/config/o6.yaml \
   ~/primitive/install/share/hand_gesture_primitives/config/o6.yaml
# 或重新构建
cd ~/primitive && colcon build --merge-install --packages-select hand_gesture_primitives
```

---

## 10. named pose 与示教链路

```bash
ros2 run hand_gesture_primitives pub_gesture_cmd -- right "mode teach"
ros2 run hand_gesture_primitives save_pose --ros-args \
  -p name:=o6_test -p hand_side:=right -p hand_type:=o6 -p keyframe:=pre

ros2 run hand_gesture_primitives pub_gesture_cmd -- right "mode primitive"
ros2 run hand_gesture_primitives pub_gesture_cmd -- right "pose o6_test"
```

### 10.1 `save_pose` 必须显式传 `robot_description`

**单独运行 `save_pose` 会失败**：

```
[ERROR] [save_pose]: radians 模式初始化失败: radians 模式需要非空 robot_description
```

原因：`save_pose` 默认 `ros2_control_command_units:=radians`（与真机一致），该模式要用 URDF 的关节信息构建弧度转换器（`save_pose.py:97-109`）。launch 会给所有 gesture 节点自动传 `robot_description`，但 `save_pose` 是独立进程，**必须自己传**。

用与 launch **完全相同的 xacro 参数**生成（参数不一致会导致关节集合对不上）：

```bash
ROS_DESC="$(xacro /home/ubuntu/luban_framework/install/share/luban_robot_model/config/o6_hand_only.urdf.xacro \
  mock:=false right_can_channel:=can0 left_can_channel:=can1 enable_right:=1 enable_left:=0)"

ros2 run hand_gesture_primitives save_pose --ros-args \
  -p name:=o6_test -p hand_side:=right -p hand_type:=o6 -p keyframe:=pre \
  -p robot_description:="$ROS_DESC"
```

### 10.2 快速验证姿态库往返（不需要示教面板）

`save_pose` 直接从 `/joint_states` 抓当前关节，**不依赖 `mode teach`**。所以可以先用一个原语把手指到已知位姿再存：

```bash
ros2 run hand_gesture_primitives pub_gesture_cmd -- right "fist"      # 到握拳位，等停稳
# （用上面的命令 save_pose 存下来）
ros2 run hand_gesture_primitives pub_gesture_cmd -- right "open"      # 换到张开位
ros2 run hand_gesture_primitives pub_gesture_cmd -- right "pose o6_test"   # 应回到握拳位
```

检查存储（文件名带侧别、内容是 6 个值）：

```bash
ls ~/.config/hand_gesture_primitives/poses/
cat ~/.config/hand_gesture_primitives/poses/*_right.json
```

### 10.3 `mode teach` 的语义

`executor.py:106` —— **TEACH 模式停发指令，把控制权让给示教 GUI / save_pose**。也就是 gesture_node 停止下发，由示教面板（或直接往 `/right_hand_controller/commands` 发指令的一方）接管；退出 TEACH 时 hold 最近的下发命令，避免跳变。

⚠️ **没带 `TEACH_PANEL=1` 启动时，进 TEACH 后没有东西接管**——手保持当前位置，但无法通过面板摆姿势。要用面板示教必须在 launch 时加 `TEACH_PANEL=1`（右侧 Web 面板 8002）。

**验收**：

- 文件名带侧别（`o6_right.json` / `o6_left.json`），文件中 `hand_side` 字段正确 ⇒ **左右手不串库**
- `hand_joint` 记为 `O6`、`source` 为 `teach`、`unit` 为 `rad`
- ⚠️ **姿态库内部是 30 通道 `semantic_v3`，不是 6 轴原始值**（`num_joints: 30`，字段名如 `thumb_mcp_flex` / `index_pip` / `wrist_flex` / `reserved`）。这是设计如此：O6 的 6 轴在存取时经语义空间转换，`hand_joint` 记录来源手型。**不要用"是不是 6 个值"来判断成败。**

真机实测（2026-09-17）：`o6_test` 存入 `~/.config/hand_gesture_primitives/poses/o6_right.json`，`hand_side: right`、`hand_joint: O6`、`num_joints: 30`、keyframe `pre` 正确写入，存的是握拳位姿（四指 `mcp_flex=1.57`，拇指 `mcp_flex=1.262` / `mcp_abd=0.954`）。

### 10.4 Web 示教面板（`TEACH_PANEL=1`）

launch 加 `TEACH_PANEL=1` 后启动面板：**右手 8002 / 左手 8001**（`launch_utils.py:184`，避开 eViz 的 8765），默认绑 `127.0.0.1`，浏览器开 `http://localhost:8002`。

0.2.1 面板要求 Pydantic 2；Ubuntu 24.04 的 apt Web 栈会提供 Pydantic 1，
不能沿用仅安装 `python3-fastapi`/`python3-uvicorn` 的步骤。请先按
[独立 Jazzy 环境与安装验收](workbench.md#ros-与双手) 创建系统 Python venv，
使用同一解释器构建并加载对应 overlay。该流程保持 apt NumPy/SciPy，验证已安装
入口与双手 Mock；现场动作验证仍须单独完成。0.2.2 恢复示教页为首页，
卡片工作台位于 `/workbench`，`/legacy` 保留同一示教页兼容入口。

面板能做：实时驱动手、编辑/保存姿态、pose bundle 导出导入（`/api/poses`、`/api/pose_bundle/export|import|import_factory`）。**不显示力/触觉数据**——排查触觉（§9.4）用不上它。

**「导入出厂预设」只导入 11 个原语，不是全部。这是设计边界，不是功能缺失。**

姿态库读的是显式 `HAND_GESTURE_POSE_DIR` 中当前手型、当前侧的文件；
旧版本曾使用 `~/.config/hand_gesture_primitives/poses/`，现在不再隐式回退。
内置原语由独立目录直接展示和执行，与用户保存的姿态是两套数据。点「导入出厂预设」会调 `/api/pose_bundle/import_factory`，把可规划静态原语的终态写成 `preset_<名字>` 存进用户库，之后才出现在列表里。两种模式：普通（已存在则跳过）/ 覆盖（标定更新后用）。CLI 等价入口是 `export_primitive_poses`。

**四个容易混的数字**：

| 数字 | 含义 |
| --- | --- |
| 40 | 全局注册原语总数（所有手型） |
| 32 | O6 allowlist 支持的原语数（40 减去 8 个硬件做不到的） |
| 30 | 回归套件步数（含 `mode primitive` / 归位等非原语步） |
| **11** | O6 的**可规划静态原语** ← 面板导入的就是这批 |

O6 的 11 个：

```
init, open, release, relax_grip, fist, pinch, index_pinch, middle_pinch, ok_sign, point, v_sign
```

导出条件很严，必须**同时**满足：**有单一静态终态** + **能被 FK 规划** + **在 `o6.yaml` 中配置过**。被排除的：13 个 `*_by_vision`（依赖感知）、分阶段类（`large_wrap` / `hammer_grip` / `thumb_adduction_grip` / `tripod` / `tripod_by_force`，多终态）、力控类（`ring` / `middle_ring`，靠接触反馈停指）、`named_pose`（需传名字）。

**姿态库存的是"固定位姿"，不适合表达带时序的动作——其余原语直接用命令发**：

```bash
ros2 run hand_gesture_primitives pub_gesture_cmd -- right "large_wrap prep" --timeout 30
ros2 run hand_gesture_primitives pub_gesture_cmd -- right "ring" --timeout 30
```

随时可以核对这两个集合：

```bash
python3 -c "
from hand_gesture_primitives.export_primitive_poses import resolve_primitive_list
for s in ('planable','static_all'):
    print(s, len(resolve_primitive_list(None, hand_type='o6', set_name=s)))
"
```

---

## 11. 发布前检查

- [ ] **公开 API 兼容**：`luban_framework` 的 `launch_o6_hand.py` 直接 import 本仓三个接口——`contact_config.clamp_control_rate_hz`、`launch_utils.all_gesture_nodes`、`launch_utils.resolve_hand_only_sides`。硬件侧会持续消费，签名不可随意变更，否则 luban 侧 launch 直接崩溃。
  **已由实跑证明**（2026-09-17）：该 launch 在 MOCK 和真机下都成功启动，说明这三个接口存在且签名兼容。改动它们时务必同步 luban 侧。
- [ ] **`data/urdf/o6/` 纳入版本控制**：已在提交 `f102609` 中入库（含 `linkerhand_o6_right.urdf` 与 meshes）。
- [ ] **独立可导入**：不依赖 luban 路径。真机启动日志证实 FK 用的是本仓 URDF（`urdf=/home/ubuntu/hand_gesture_primitives_release/data/urdf/o6/right/linkerhand_o6_right.urdf`），`urdf_utils.py` 中对 `src/robot_model/evo_work` 的兼容回退未命中。
- [x] 发布前重跑全量回归：`pytest -q` + `ruff check`。
  **实测通过（2026-09-17）**：`366 passed, 1 skipped`；ruff 无告警。

⚠️ **README 的表格格式是测试契约的一部分**，不是能随便格式化的。

`test/test_readme_matches_profiles.py` 用正则解析 README 表格，**要求管道符两侧无空格**（`|O6|32|`，而不是 `| O6 | 32 |`）：

```python
rf"^\|{label}\|(\d+)\|$"        # 计数表
r"^\|\d+\|([^|]+)\|"            # 原语表
```

用 Markdown 格式化工具（如编辑器的 format-on-save）重排表格后，两个测试会同时失败，报**全部手型 / 全部原语缺失**——因为整张表都解析不到了。这个报错看起来像内容丢了，其实只是格式。

本次实测踩到过：格式化后 `test_count_table_covers_all_hands`、`test_every_registered_primitive_is_documented` 双双失败，`git restore README.md` 即恢复。**改动 README 后务必重跑 §2 的回归。**

若确实想保留格式化后的表格，可把上面两个正则改成容忍空白（`^\|\s*{label}\s*\|\s*(\d+)\s*\|$` 等），防漂移能力不变。

全部通过后再决定合并策略：

```bash
git fetch origin
git diff origin/main...HEAD
git log --oneline origin/main..HEAD
```

---

## 附：已核实事实速查

| 项               | 值                                                                                         |
| ---------------- | ------------------------------------------------------------------------------------------ |
| gesture 指令话题 | `/{side}_hand/gesture_cmd`（右手 `/right_hand/gesture_cmd`）                           |
| 状态 / 结果话题  | `/{side}_hand/gesture_status`、`/{side}_hand/gesture_result`                           |
| 力反馈话题       | `/cb_right_hand_force`、`/cb_right_hand_normal_force`、`/cb_right_hand_motor_torque` |
| 控制器命令话题   | `/right_hand_controller/commands`                                                        |
| 控制单位         | 弧度（launch 硬编码`ros2_control` + `radians`）                                        |
| `control_rate` | `GESTURE_CONTROL_RATE`，默认 10，经 `clamp_control_rate_hz` 钳位                       |
| CAN 默认         | `RIGHT_CAN=can0`、`LEFT_CAN=can1`（与侧别解耦，需实测）                                |
| 侧别选择         | `GESTURE_SIDE` / `HAND_SIDE`，`HAND_SIDE` 优先，默认 `left`                        |
| O6 力反馈配置    | `touch=true`、`current=false`、`motor_torque=true`                                   |

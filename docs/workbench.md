# 独立手势工作台（0.2.2）

原语运行时、姿态库和工作台公共逻辑由本仓库提供。linker 使用相同的
Python 服务，保留采集、遥操作和机器人接入。独立面板不依赖 LeRobot。
真机通过已有 ROS 驱动控制；本版本没有新增 SDK/CAN 直连。
首次使用请从 [下载、安装与启动](getting-started.md) 开始；网页每个按钮的用途、动作与
保存副作用见 [网页操作指南](teach-panel-user-guide.md)，视觉/力控条件见
[感知原语配置](perception-setup.md)。本文保留运行配置与公共接口约定。
原有与新增能力的逐项对应见 [功能兼容验收清单](workbench-feature-compatibility.md)。

## 安装与 Mock

Python 3.12 环境中，在仓库根目录安装：

```bash
python -m pip install './hand_gesture_primitives[teach,test]'
export HAND_GESTURE_POSE_DIR="$HOME/hand-gesture-data/poses"
teach_panel --mock --pose-dir "$HAND_GESTURE_POSE_DIR" \
  --hand left:o6 --hand right:o6 --host 127.0.0.1 --port 8001
```

浏览器访问终端显示的本机地址。首页保留原示教面板的手掌滑条、关键帧树和姿态库，
并在同一页提供内置原语、速度和侧别选择；`/legacy` 是同一界面的兼容入口。
卡片式工作台作为可选入口放在 `/workbench`，两页共用原语目录与控制路径。
Mock 只在内存模拟手，不创建 ROS 节点或硬件驱动。
仅导入/计算原语无需 Web 或 ROS；`teach` extra 安装网页运行依赖。
FK 为可选能力：ROS 环境使用发行版 Pinocchio；独立 Python 可安装 `fk` extra。
不要把不同 NumPy ABI 的 ROS 与 pip Pinocchio 混用。
安装包同时携带原仓库已有的 O6 右手 URDF 与 meshes，并支持安装后的资源发现；
其他未随仓库提供的真实模型仍可通过 `LINKERHAND_URDF_DIR` 配置。模型资源与
Pinocchio 依赖分开管理，不根据右手模型推测左手模型。

## ROS 与双手

使用 Ubuntu 24.04 的 ROS 2 Jazzy 和系统 Python 3.12。工作台要求
**Pydantic 2**；Noble 的 `python3-pydantic` 是 v1，仅通过 apt/colcon 安装
不能满足此要求。使用由系统 Python 创建、带 `--system-site-packages` 的独立
venv：NumPy、SciPy、ROS 扩展继续来自 apt，Web 依赖装入 venv。
不要使用 Conda Python 构建这个独立 ROS 环境；linker 自己的安装流程另行管理。

在新终端、仓库根目录准备宿主依赖后执行：

```bash
source /opt/ros/jazzy/setup.bash
sudo apt install python3-venv python3-numpy python3-scipy python3-yaml \
  python3-filelock python3-colcon-common-extensions
# Pydantic 由下述 venv 提供；不要让 rosdep 的 Noble 键代替 v2。
rosdep install --from-paths hand_gesture_primitives messages --ignore-src \
  --skip-keys python3-pydantic -y
export HAND_GESTURE_INSTALL="$HOME/hand-gesture-0.2.2"
/usr/bin/python3 hand_gesture_primitives/scripts/validate_ros_install.py \
  --output "$HAND_GESTURE_INSTALL"
source "$HAND_GESTURE_INSTALL/venv/bin/activate"
source "$HAND_GESTURE_INSTALL/workspace/install/setup.bash"
export PYTHONNOUSERSITE=1
export HAND_GESTURE_POSE_DIR="$HOME/hand-gesture-data/poses"
```

输出目录必须尚不存在。脚本将源码复制到该目录，建立独立 build/install/log，
不改写源码树、既有环境或姿态库。它从系统解释器生成 NumPy/SciPy 精确约束，
仅升级 Web/构建依赖，并核对安装前后数值库版本和实际导入路径保持为 apt。
ROS 环境不安装 `fk` extra，也不使用 pip 的 `--ignore-installed` 或 eager 升级。
已有 Web/构建依赖 wheelhouse 时可加 `--wheelhouse /path/to/wheels`，此时 pip
使用 `--no-index`；ROS 和 apt 依赖仍须预装。

脚本显式使用 `venv/bin/python /usr/bin/colcon`，并向消息包传入同一
`Python3_EXECUTABLE`。仅激活 venv 后运行系统 `colcon` 会被其 shebang 带回
系统解释器，不能保证入口使用 Pydantic 2。脚本检查安装后的 `teach_panel`、
`gesture_node` shebang、ROS overlay、消息导入及双手 Mock HTTP，结果写入
`acceptance.json`，详细过程保存在 `pip.log`、`colcon.log`、`mock.log`。
验证只启动 Mock 面板，不创建 ROS 节点或硬件驱动。换版本时使用新的输出目录，
继续显式指向原有姿态目录。

单手沿用 `teach_panel.launch.py`；`pose_dir` 默认读取上述环境变量。
`primitive_speed` 默认 1.0，范围 0.25–3.0，`control_rate` 默认 10 Hz。
`robot_description` 必须与实际驱动的关节顺序和弧度限位一致。

下例的 `launch_gesture_node:=true` 会启动原语执行器，启动时会选择 `init`，可能让真机运动；
不能当作只打开网页。真机运行前，先按 [安装与启动指南](getting-started.md) 中的
控制器准备、停止确认和再启用流程处理，再运行下面的命令：

```bash
ros2 launch hand_gesture_primitives teach_panel.launch.py \
  hand_type:=o6 hand_side:=right launch_gesture_node:=true \
  pose_dir:="$HAND_GESTURE_POSE_DIR" primitive_speed:=1.0 \
  robot_description:="$(cat /path/to/o6_controller.urdf)"
```

双手分别运行 gesture_node，命令和反馈话题按侧隔离。
独立面板支持 `workbench_hands` ROS 字符串数组，如 `['left:o6','right:o6']`；
空数组沿用现有 `hand_side` / `hand_type`。默认打开配置的 `hand_side`，不取
`workbench_hands` 的首项；显式 URL `?side=left` 或 `?side=right` 可选择已配置的一侧。
默认侧通过 `/api/workbench/state` 的 `default_side` 返回；Mock 双手默认右侧。
双手的 robot_description 应包含两侧关节。
面板通过 v2 工作台命令控制节点；它不再与 gesture_node 同时直接发布手目标。
既有 `legacy_cb` 和 `ros2_control` 驱动接入方式仍保留；新版编辑器在两种
接入方式下都要求真实 URDF 转换器，缺失时启动失败，不猜测真机关节限位。
双手启动示例（原语节点和面板分别在独立终端运行，加载同一环境）：

```bash
ros2 launch hand_gesture_primitives bimanual_gesture.launch.py \
  hand_type:=o6 pose_dir:="$HAND_GESTURE_POSE_DIR" \
  robot_description:="$(cat /path/to/dual_o6_controller.urdf)"
ros2 run hand_gesture_primitives teach_panel --ros-args \
  -p hand_type:=o6 -p hand_side:=right -p "workbench_hands:=['left:o6','right:o6']" \
  -p "robot_description:=$(cat /path/to/dual_o6_controller.urdf)"
```

`observe_only:=true` 可用于只读面板；观察模式不修改节点速度或发送动作。

## 真机播放排查

内置原语与已保存姿态分别显示；内置原语不需要先导入出厂预设。出厂预设只是
可规划原语的终态快照，并不代表全部支持的原语。姿态目录按手型、左右侧分库，
新验收目录为空时不会自动读取其他目录或另一只手的数据。

页面有新鲜关节反馈，只说明状态广播器正在工作，不能据此判定运动控制器已激活。
若轨迹进度前进但手不动，先核对页眉侧别，再只读检查对应控制器：

```bash
ros2 control list_controllers -c /left_hand_driver/controller_manager
ros2 control list_controllers -c /right_hand_driver/controller_manager
```

如果对应的 forward controller 为 `inactive`，不要直接激活：执行器可能仍保留
先前的目标。先停止该侧原语，等待工作台状态确认 `active=null`、
`execution_kind=idle`、`command_pending=false`、`state_confirmed=true`，
再按现场启动流程激活控制器，由用户重新明确点击播放。这里的控制器名称为
linker 双 O6 配置示例，其他部署使用自己的 controller manager 和 controller 名称。

## 数据与公共接口

- `HAND_GESTURE_POSE_DIR` 必须显式设置，且路径不得经过软链接；没有隐式 HOME 回退。
- 运行状态路径使用 `HAND_GESTURE_RUNTIME_STATUS_PATH`；默认写入
  `~/.local/state/hand_gesture_primitives/runtime-status.json`。
- `PoseDraft` 编辑位置为控制器顺序的弧度；落盘保留 `semantic_v3/rad/30`。
  既有 hardware、semantic_o20 与 legacy255 姿态仍可读取。
- 创建使用空 revision；更新/删除携带读取到的 revision，冲突拒绝覆盖。
  左右姿态分桶，菜单自定义 ID 不复用。
- 公共模块包括 `workbench_core`、`workbench_runner`、`workbench_client`、
  `workbench_edit` 和 `workbench_catalog`。宿主提供 JointAdapter、反馈及运行调度；
  runner 返回关节目标，设备写入仍由宿主唯一发送者负责。
- 原有 ROS 字符串命令和姿态 CLI 保留；示教面板为默认首页，`/legacy` 兼容入口
  与 bundle 导入导出继续可用。两种页面动作均经公共控制路径处理。

## 交互和控制语义

默认示教页与扩展工作台共用控制路径，但保存时机不同。默认页“保存”“＋ 添加”、
关键帧排序/重命名/删除立即写入姿态库；工作台编辑器的帧调整先留在草稿，点击
“保存原语”才落盘。“载入”或切帧不自动将整帧目标下发给手，但进入编辑可能停止
既有动作。逐关节拖动在示教或实时跟随启用时驱动所编辑关节；捕获、保存不执行整帧。

编辑暂停作用于同一个面板实例管理的所有手。首次进入编辑时，先停止所有侧的普通动作，
再允许编辑者跟随或试播；只读观察不发送这些命令。关闭工作台编辑器后需显式恢复控制；
默认页“退出示教”在没有其他编辑占用时恢复输入。恢复输入不自动续播动作。

有限轨迹支持按住推进、松开暂停、再次按住继续。到达终点后持续按住不循环；
重新执行由当前姿态起步。0.25–3.0 倍速仅改变后续运动时间，不改变硬件限矩，
不缩放接触保护时间，也不使进度跳变。速度由本面板实例配置的左右手共用，
仅对本次运行生效；选择侧别不会把倍率修改限定为单侧。

默认页“定点”进入所选段的 progress=0；该起点取建立这次段预览时的当前手姿，
不是保存帧的目标。全局条按各帧时长加权，跨段时建立新段预览；同段连续拖动不重新取起点。
工作台执行状态条只显示进度；编辑器的可拖动进度必须先建立本页试播。

编辑 token 的租约为 10 秒，单手同一时刻只有一个实时编辑者。
实时跟随 0.5 秒未续约即停止。停止序号使晚到的旧更新失效。
切帧、试播、离开页面和连接中断清除跟随；恢复后需要新的明确操作。
linker 的录制冻结、输入 neutral 和显式恢复遥操作仍由其 Session 执行。

轨迹结束与反馈到位分别显示。连续 3 条不同的新鲜反馈确认到位；
等待超过 1 秒报告未到位并保持终点。进程内阈值为 0.03 rad；
ROS executor 使用 3.0 硬件单位并只比较可观测通道。
停止 ACK 表示执行器已停止处理该动作，不能作为电机物理停稳的证明。

## 0.2.1 控制与进度契约

ROS 工作台协议继续使用 v2。新增 `progress_controls: 1` 能力用于绑定执行身份的
进度控制：preview 分配 `control_id`，可选 `initial_progress` 在首个控制周期前
原子进入进度模式；后续 `kind=progress` 携带相同 `control_id` 和 `progress`。
缺少能力的旧节点会明确提示升级，不退回未关联执行的进度命令。
旧 Float32 `motion_progress` 话题继续供既有客户端使用。

同一姿态内容 revision、关键帧和编辑租约的连续 scrub 只改变进度，不重新取当前
手姿作为起点。换段或换版本时重新锚定一次。停止、失权和节点重启使旧请求失效，
迟到的确认不能恢复动作。窗口失焦、页面隐藏和指针捕获丢失都会释放按住推进；
恢复焦点后需要重新按下。

Python 原语结果新增可选 `PrimitiveResult.trajectory_finished`，缺省 `None` 保留
已有 done/engine 判定兼容。明确的终点标记用于统一普通与按住执行；名义时长只
显示进度，不能替代终点或新鲜反馈。持续力控不因静止或经过足够时间而自动结束。

运行状态中的 `completed_control_id` / `completed_kind` 标识本执行自然完成后的
终态，`done` 与 `settle_status` 说明反馈确认结果。旧页面仅根据匹配身份的普通
AUTO 完成处理返回示教，PROGRESS 和 held 不会自动退出。显式不返回的停止请求
以及当前关闭的返回示教偏好始终得到遵守。

内部控制世代同时覆盖旧字符串和 v2 控制入口，阻止停止后迟到的门控批准复活
动作；无效请求不撤销原有合法动作。O6 的 hand_info 与 motor_torque 使用独立
时间戳，优先新鲜 hand_info，再选择新鲜 topic，读取或切换来源不刷新样本时间。

## 验收与证据

0.2.2 的发布验收范围与已有证据见 [发布说明](releases/0.2.2.md)。
用户已反馈左右手自定义多阶段播放、调速、进度条播放通过，并于 2026-10-24
明确跳过其余现场测试。跳过项不计通过，也不再作为本轮发布阻塞；下述流程保留为
后续部署的验收方法，不表示本轮已经执行。

在包目录运行原有与回迁测试：

```bash
cd hand_gesture_primitives
python -m pytest test
```

纯 Python/Mock 测试和 Jazzy ROS 测试分开执行。测试使用临时姿态目录和
隔离的 ROS domain，不能使用现场姿态库。发行前还需验证 wheel 安装后
静态资源、配置和命令入口完整，以及不安装 linker 时能够启动面板。

双 O6 真机验收按以下顺序记录，两侧默认接口为左 can0 / 右 can1，现场核实后使用：

1. 记录上游与 linker 提交、配置、固件、ROS domain、关节顺序和反馈频率。
2. 单侧验证 open/pinch、自定义多帧、调速、暂停续播、逐关节跟随与停止。
3. 双侧执行不同动作，确认姿态、控制权、状态、停止及节点重启互不串扰。
4. 验证断联、租约失效和陈旧反馈；恢复后动作不得自动续播。
5. 在 linker 的 inprocess 和 gesture_node 模式分别录制，核对实际命令、
   姿态 revision、稳定菜单 ID 和速度事件。

验收应区分自动化通过、模拟 ROS 通过与真机通过，并注明用户反馈的来源。
设备不可用或被占用时，记录未执行原因；明确调整验收范围时，将对应项目记录为
跳过，不计为通过。不得用 Mock 结果替代真机结果。

## 来源与升级

运行时回迁基线为内部 release `fe55014541c3abcffccb74632b992279bf4e6b12`，
对应 linker 累计补丁 17 个修改文件与 3 个新模块。公共工作台模块另由
linker `4e72f70` 抽取，来源及许可证见根目录 NOTICE。

正式版本使用 Git commit 与逐文件清单锁定；安装时不再打 linker 的累计补丁。
旧离线包兼容仅接受登记过的校验值；未知本地改动不覆盖。升级前保留原姿态目录
及旧依赖包，版本切换不自动改写用户姿态文件。

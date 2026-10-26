# 原有功能与新增控制能力验收

回迁以 `fe55014541c3abcffccb74632b992279bf4e6b12` 为原仓库基线。
要求是保留原有功能并增加控制能力；恢复原示教界面不能撤掉速度、进度或暂停续播。
原语未满足感知或硬件条件时应显示原因，不能从能力目录中删除以掩盖问题。

## 功能对应

| 能力 | 保留或新增的行为 | 主要回归证据 |
| --- | --- | --- |
| 内置原语与手型 | 原有 40 项注册及 catalog 保留；5 safe、17 gesture、5 force、13 vision；手型配置未删改 | `test_primitive_discovery.py`、`test_capability_matrix_consistency.py`、`test_readme_matches_profiles.py` |
| 感知与力控 | 13 个 vision 和 5 个 force 原语保留，`auto` 的 precision/lateral/power 映射保留；缺少反馈、目标、FK 或 gate 时列出原因 | `test_review_fix_plan.py`、`test_workbench_protocol.py`、力控原语测试 |
| 自定义姿态 | 创建、载入编辑、保存、删除，hardware/semantic 兼容、左右手独立存储继续可用 | `test_pose_library.py`、`test_named_pose_hardware.py`、`test_standalone_workbench.py` |
| 关键帧 | 添加、载入、重命名、排序、删帧，单帧与全动作 AUTO，以及原加权全局进度保留 | `test_named_pose_keyframes.py`、`test_teach_panel_keyframes.py`、`test_teach_panel_compatibility.py` |
| 示教与偏好 | 原手掌滑条、反馈同步、角度显示、返回示教偏好保留；切帧和保存不发送整帧动作 | `test_panel_controller.py`、`test_teach_layout_convert.py`、浏览器新旧页面流程 |
| 姿态迁移 | bundle 导入导出、冲突策略、校验和、出厂预设保留；导出数量与空库提示所需响应头保留 | `test_pose_bundle.py`、`test_teach_panel_bundle_api.py`、`test_standalone_workbench.py` |
| 旧控制入口 | ROS 字符串、Float32 motion_progress、旧 WS 消息与原有 13 个 CLI 入口保留 | `test_workbench_protocol.py`、`test_pub_gesture_cmd.py`、`test_run_gesture_cmd_suite.py` |
| 执行速度 | 新增 0.25–3.0 倍速；只改变轨迹时间，不改变硬件限矩或接触保护时间 | `test_runtime_control.py`、`test_workbench_core.py`、安装产物浏览器 |
| 有限动作推进 | 新增按住推进、松开暂停、重新按住续播；失焦、失联或停止后不自动续播 | `test_runtime_control.py`、`test_standalone_workbench.py`、浏览器 blur/Tab/DOM owner 回归 |
| 自定义进度细控 | 预览有身份绑定；同一段拖动不重锚，换段按关键帧处理，停止后旧进度请求失效 | `test_motion_progress.py`、`test_progress_policy.py`、`test_workbench_protocol.py` |
| 运行模型资源 | 仓库自带 O6 右手 URDF 与 meshes 必须跟随安装产物交付；外部模型覆盖入口继续保留 | 真实 wheel/sdist/colcon 安装与无源码路径查找验证 |

## 页面入口

默认 `/` 继续使用原手掌、常驻滑条、关键帧树、姿态库；内置原语、执行速度、停止和
左右手选择在同一页。原关键帧全局进度可直接拖动，不需要跳转。
`/legacy` 是此页的兼容入口。

`/workbench` 提供默认、感知、自定义三分类卡片，以及有限动作按住推进、菜单设置、
多帧编辑与预览进度。默认页用明确的“按住推进与菜单”链接进入；普通控制模式需
先停止本页手势并退出示教。只读页面无本页编辑时可以浏览另一侧和另一页面，
不替外部执行者发送停止或恢复指令。

全局状态里的轨迹进度显示与可拖动的自定义预览进度不同。持续力控和感知闭环原语
不能任意跳转到某个百分比；其有效条件、控制周期和接触保护必须继续成立。

## 配置与验收边界

- `HAND_GESTURE_POSE_DIR` 显式指定共享姿态目录，按手型、左右侧分桶；新验收目录
  不自动搬运旧目录或另一侧的姿态。原文件格式可继续导入读取。
- 感知原语的软件保留与真机感知闭环验证分别记录。原仓库只自带 O6 右模型；
  左 O6 及其他未随仓库提供的模型仍需真实外部 URDF，不用右模型镜像猜测。
- Pinocchio 仍是 FK 可选依赖，ROS 安装使用与系统数值库匹配的版本；保持 apt
  NumPy/SciPy，不借打包模型升级数值 ABI。
- 关节反馈新鲜不等于运动控制器 active。当前现场双 O6 配置关闭了力反馈且没有
  提供视觉目标，不能把普通动作或 Mock 通过当作感知真机验收完成。

## 0.2.2 发布验收结论（2026-10-24）

原语候选 `2efb4887276d9ba66bbe5a09548a00fce02364f5` 完整软件测试为
601 项通过、1 项因缺少 O30 URDF 跳过；安装产物浏览器、资源交付及远程 Mock
数据流程验证通过。Mock 数据流程单独记录为 30 项通过。

用户反馈已在真机验证左右手自定义多阶段播放、执行速度调节、拖动进度条播放三项。
这些结果的来源为用户反馈，不能扩展为所有双手或感知用例已通过。

用户于 2026-10-24 明确决定“不再进行现场测试，这一步跳过”。剩余真机停止响应、
双手同时运行的控制隔离、感知/力控闭环和 linker 真实录制均标记为用户决定跳过，
不计入通过数量，也不再作为本轮发布阻塞。

对应 linker 候选 `8d6034f0e12346e8c33f3d3240b5567d37ddd51f` 的 34 项消费测试
为该候选的历史证据；合并到新的 linker 基线后的结果由 linker 发布记录单独列出。
版本、安装要求和已知限制见 [0.2.2 发布说明](releases/0.2.2.md)。

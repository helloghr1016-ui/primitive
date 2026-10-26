> 0.2.0 的公共服务、编辑租约与 ROS v2 路径见 [工作台说明](../../docs/workbench.md)。下文保留原示教布局和接口的设计背景。

# Teach 面板架构（方案 A：本包独立面板 · Web UI）

> **luban 适配**：默认订阅/发布 `/{side}_hand/gesture_*`；关节写 `cmd_backend`
> （ros2_control Float64 / legacy JointState）。面板内部仍为 hardware 0–255。

本文档定义 **长期方案 A**：在 `hand_gesture_primitives` 内提供独立 Teach 面板，**View 层采用轻量 Web**（浏览器访问）。
控制权与姿态库约定继承 [`control_modes_and_custom_pose.md`](./control_modes_and_custom_pose.md)；本文只规定 **UI 进程边界、模块分层、状态机与接口**。

**View 决策**：不使用 PyQt / 桌面 GUI；浏览器 + 本机 HTTP/WebSocket 服务，便于本机、远程笔记本与无桌面工控机。

---

## 1. 目标与非目标

### 目标

| 能力 | 说明 |
|------|------|
| 一站示教 | 浏览器一页完成：进 TEACH → 调关节 → 保存（可多关键帧）→ 列表/删除 → 试播 / progress 细控 |
| 与现网一致 | 试播走 `pose <name> [<kf> [p]]` → `NamedPose`；细控走 `motion_progress` |
| 关键帧 | 姿态库「手势→关键帧」树（可过滤）；「N帧」=数量；＋添加需示教；树上回放走 `pose <name> <kf> auto` 单段 AUTO；↑↓改序/改名/删；全局进度按时长加权 |
| 单一姿态库 | 读写 显式 `HAND_GESTURE_POSE_DIR` 下的 `{canonical}_{side}.json` |
| 进程隔离 | Web/UI 卡顿不影响 10Hz 执行环 |
| 适用范围 | 本机浏览器、局域网远程、无桌面主机（操作员机开页即可） |
| 可选依赖 | 默认安装不含 Web 栈；`[teach]` extra 安装服务端依赖 |

### 非目标

- PyQt / 嵌入 `linkerhand-ros2-sdk/gui_control`（可做单向导入，不作运行时依赖）
- 重型前端框架与独立 SPA 工程（MVP 用单页静态资源即可）
- 时间序列宏、力控/by_vision 参数可视化编辑
- 跨手型自动关节 remap
- 在 UI 进程内再实现一套 Executor / 过流保护
- 与 `~/.config/gui_control/presets/` 双向同步
- 默认对公网开放（默认只绑本机环回）

---

## 2. 设计原则

1. **薄 UI，厚栈复用**：面板 = Authority 切换器 + 硬件关节编辑器 + PoseLibrary 浏览器。
2. **双保险控制权**：Executor 在 TEACH 停发；面板在非 TEACH / 试播时 **Mute** 滑条发布。
3. **捕获读 state，编辑写 cmd**：示教保存**默认用滑条目标**（避免未到位反馈冲库）；可选采反馈。关节写 `ros2_control` / legacy cmd。
4. **指令通道统一**：模式切换与试播只发 `/hand_gesture_cmd_exec` 的 String（与 CLI 同语义）。
5. **DOF / 手型单一真相**：`HandConfig` + `resolve_hand_config_key()`（L20→l25）。
6. **库逻辑不进前端**：校验 / 保留字 / IO 只走 `pose_library`（服务端或二期 Service）。
7. **Web 只换壳**：Controller / RosBridge / Mute 与桌面方案同构；浏览器经 WebSocket 调 Controller API。

---

## 3. 总体架构

```
  浏览器（本机或局域网）
       │  HTTP 静态页
       │  WebSocket 实时（关节 / mode / 列表 / 操作）
       ▼
┌──────────────────────────────────────────────────────────────────┐
│  teach_panel 进程（可选 [teach]）                                  │
│                                                                  │
│  ┌─────────────┐   ┌──────────────────┐   ┌───────────────────┐ │
│  │ Web Server  │◄─►│  PanelController │◄─►│  RosBridge        │ │
│  │ 静态页+WS   │   │  状态机 / Mute   │   │  pub/sub / params │ │
│  └─────────────┘   └────────┬─────────┘   └─────────┬─────────┘ │
│                             │                         │           │
│                             ▼                         │           │
│                    pose_library (CRUD)                │           │
└─────────────────────────────────────────────────────┼───────────┘
                                                      │
          cmd String                    JointState     │  JointState
          mode / pose / pose_list       control_cmd    │  hand_state
                      │                      │         │  status
                      ▼                      ▼         ▼
              /hand_gesture_cmd_exec    /cb_{side}_hand_control_cmd
                      │                 /cb_{side}_hand_state
                      ▼                 /hand_gesture_status
               gesture_node
                      │
                      ▼
               Executor (ControlMode)
               · TEACH: 不 _publish_cmd
               · PRIMITIVE: NamedPose / 内置原语
```

### 进程与包边界

| 组件 | 进程 | 包路径（建议） |
|------|------|----------------|
| `gesture_node` | 始终运行 | 现有 |
| `teach_panel` | 按需启动（ROS + HTTP/WS） | `hand_gesture_primitives/teach_panel/` |
| `save_pose` | CLI 保留 | 现有；面板服务端直接调 `upsert_pose` |

```
hand_gesture_primitives/
  pose_library.py          # 已有
  save_pose.py             # 已有 CLI
  teach_panel/
    __init__.py
    main.py                # console_scripts 入口
    server.py              # HTTP + WebSocket（FastAPI/uvicorn）
    ros_bridge.py          # 订阅/发布，无 Web 框架依赖（可单测）
    controller.py          # 状态机 + Mute + 业务编排
    static/                # 单页前端（index.html + app.js + app.css）
    launch/
      teach_panel.launch.py
```

`setup.py`：

- `console_scripts`: `teach_panel = hand_gesture_primitives.teach_panel.main:main`
- `extras_require["teach"]`: `fastapi`、`uvicorn[standard]`（实现锁定；不引入 PyQt）

---

## 4. 模块职责

### 4.1 Web View（`static/`）

- 显示：手型、左右手、当前 `mode`、连接/超时提示
- 关节滑条：长度由服务端下发的 `num_joints` / `joint_names` 动态生成
- 姿态区：名称、duration、description、列表、保存/覆盖/删除/试播
- **不**直接连 ROS；只与本机（或配置的）Teach HTTP/WS 通信

### 4.2 Web Server（`server.py`）

| 接口 | 用途 |
|------|------|
| `GET /` | 静态示教页 |
| `WS /ws` | 双向：状态推送 + 用户操作 |
| （可选）`GET /api/poses` | 调试用 REST；MVP 可全走 WS |

WebSocket 消息建议（JSON，示意）：

```text
# 服务端 → 浏览器
{ "type": "snapshot", "mode": "teach", "panel_state": "TEACH_EDIT",
  "joint_names": [...], "targets": [...], "feedback": [...], "poses": [...] }

# 浏览器 → 服务端
{ "type": "enter_teach" } | { "type": "exit_teach" }
{ "type": "set_joints", "positions": [...] }   # 经 Mute/节流后发 control_cmd
{ "type": "save_pose", "name": "my_open", "duration": 0.6 }
{ "type": "delete_pose", "name": "..." } | { "type": "play_pose", "name": "..." }
```

### 4.3 PanelController（核心状态机）

| 状态 | 滑条发布 | 允许操作 |
|------|----------|----------|
| `DISCONNECTED` | Mute | 仅改参数 / 重连提示 |
| `PRIMITIVE_IDLE` | Mute | 进示教、列表刷新、试播 |
| `TEACH_EDIT` | **Publish** | 调滑条、捕获保存、删库项；试播前先切模式 |
| `PLAYBACK` | Mute | 等待 `DONE`/`hold`；从示教发起且开启偏好时结束后自动 `mode teach`，否则 `PRIMITIVE_IDLE` |
| `ERROR` | Mute | 展示 reject（如 `teach_mode`、`pose_missing`） |

状态迁移（要点）：

```text
启动 → 读 status → PRIMITIVE_IDLE 或 TEACH_EDIT
「开始示教」→ pub "mode teach" → 等 status mode=teach → TEACH_EDIT
             → 用最近 hand_state 初始化滑条（避免从 0 猛跳）
「退出示教」→ Mute → pub "mode primitive" → PRIMITIVE_IDLE
「保存」    → 优先用最新 hand_state；校验失败只回 WS 错误，不发 cmd
「试播」    → Mute → 若在 TEACH 则先 mode primitive → "pose <name>" → PLAYBACK
              → 成功结束默认再 mode teach（可关「试播结束后自动回到示教」）
status 丢  → DISCONNECTED（Mute）
WS 全断    → Mute + 若仍在 TEACH 则 mode primitive（避免机载停在 teach）
```

硬规则：

- **非 `TEACH_EDIT` 绝不发布 `control_cmd`**
- **试播 / 退出示教 / WS 断开：先 Mute，再发 mode/pose**
- `stop` 只停活跃原语，**不**自动退出 TEACH（与 Executor 一致）

### 4.4 RosBridge

| 方向 | Topic | 消息 | 用途 |
|------|-------|------|------|
| sub | `/cb_{side}_hand_state` | `JointState` | 滑条跟随（可选）、捕获保存 |
| sub | `/hand_gesture_status` | `String` | 解析 `mode=`、活跃原语、reject 覆盖 |
| pub | `/cb_{side}_hand_control_cmd` | `JointState` | 仅 TEACH_EDIT；硬件量纲 0–255 |
| pub | cmd_topic（默认 `/{side}_hand/gesture_cmd`） | `String` | `mode` / `pose` / `pose_list` |

发布频率：滑条经 WS 上来后 **服务端 10–20Hz 节流**（与 Executor 同量级），松手可再发一帧终值。

参数（与 `gesture_node` / `save_pose` 对齐）：

| 参数 | 默认 | 说明 |
|------|------|------|
| `hand_side` | `left` | left / right |
| `hand_type` / `hand_joint` | `o20` | 经 `resolve_hand_config_key` |
| `cmd_topic` | `/{side}_hand/gesture_cmd` | 公开 API；有 gate 时由 gate 转发 exec |
| `ros2_control_command_units` | `radians` | 与机载 FCC 一致；须配 `robot_description` |
| `publish_rate` | `15.0` | TEACH 下 cmd 上限 Hz |
| `status_timeout_sec` | `3.0` | status 超时 → DISCONNECTED |
| `http_host` | `127.0.0.1` | **默认本机**；局域网示教再显式改 |
| `http_port` | `8001`（左）/ `8002`（右） | 浏览器打开 `http://host:port/`；避开 eViz 8765 |

### 4.5 PoseLibrary

- 服务端调用：`list_poses` / `load_pose` / `upsert_pose` / `delete_pose` / `validate_pose_name`
- 保存 `space=hardware`，`source=teach_panel`
- 不在浏览器做 semantic↔hardware 换算；回放换算留在 `NamedPose` + Executor

---

## 5. 控制权与数据流

### 5.1 示教编辑

```text
浏览器拖滑条
  → WS set_joints
  → Controller（TEACH_EDIT + 节流）
  → RosBridge.publish control_cmd (hardware)
  → 驱动跟关节
  → hand_state → snapshot 推回浏览器（可选跟随）
```

### 5.2 保存

```text
「保存」
  → 服务端取最新 hand_state.position（长度必须 == num_joints）
  → pose_library.upsert_pose(...)
  → WS 回成功/错误 + 刷新 poses 列表
```

默认**不以滑条目标落盘**（以反馈为准）。state 超时时可提供「用滑条目标保存（高级）」并二次确认。

### 5.3 试播

```text
Mute → mode primitive（若需要）→ pose <name>
  → Executor NamedPose：hardware→semantic→StaticPoseEngine→to_hardware
  → 过流保护仍生效
  → status / result → WS snapshot → DONE → PRIMITIVE_IDLE
```

---

## 6. UI 信息架构（一页）

单页三区，避免仪表盘堆砌：

1. **顶栏**：手型 / 左右手 / 模式徽章（teach|primitive）/ 「开始示教」「退出示教」
2. **主区**：按手指分组的关节滑条（名称来自 YAML `joint_names`）
3. **侧栏或底栏**：姿态名、duration、保存、列表、试播、删除

首屏不放：统计条、多机选择、感知/门控调试。

前端实现约束：

- 原生 HTML/CSS/JS 单页；不强制 React/Vue
- **手轮廓分列布局**（`hand_layout.js`）：**掌心朝屏幕外**；左手拇在左、小指在右（右手镜像）
- 指内自上而下：**指尖 → 指中 → 根弯 → 旋转 → 侧摆**（CSS `order` 强制，避免指尖沉底）
- O20/L25：拇指/四指分列；L25 含 mid；保留关节默认隐藏
- O6/L6：紧凑 6 轴（拇弯/拇摆 + 四指弯曲）
- 能力：重新初始化（`init`）、载入编辑二次修改、同步反馈、试播
- 交互参考：Shadow Robot 扁平滑条 → 本面板改为解剖分列，降低找轴成本

---

## 7. 与 SDK `gui_control` 的关系

| 项 | 决策 |
|----|------|
| 运行时依赖 | **无** |
| 交互借鉴 | 滑条 + preset 列表心智 |
| 数据 | 本包库独立；可选 **单向导入** 工具（A2） |
| 并存 | 禁止同时开 SDK GUI 与本面板抢 `control_cmd`；文档警告 |

---

## 8. 线程与生命周期

```text
main()
  rclpy.init
  bridge = RosBridge(node)
  controller = PanelController(bridge)
  后台线程: rclpy.spin(node)
  主线程/异步循环: uvicorn 跑 FastAPI（HTTP + WS）
  退出 / WS 全断: Mute → 可选 mode primitive → destroy
```

约定：

- ROS 回调 → Controller 状态更新 → 广播 WS（注意线程安全队列）
- `RosBridge` / `PanelController` **不** import FastAPI，便于无头 CI 单测
- 默认 `http_host=127.0.0.1`；绑 `0.0.0.0` 时文档要求仅内网并提示风险

---

## 9. 实施阶段（方案 A · Web）

### A0 — 骨架（可无真机）

- [x] 包结构 + `extras_require[teach]` + entry point
- [x] `RosBridge` + `PanelController` 状态机
- [x] Mute / mode 切换 / status 解析单测（无浏览器）

### A1 — MVP Web 面板

- [x] FastAPI 静态页 + WebSocket
- [x] 滑条按 `HandConfig` 动态生成；TEACH 节流发布
- [x] 捕获 state 保存；列表；删除；试播
- [x] launch：`gesture_node` + `teach_panel` 同参
- [x] README：安装 teach 依赖，浏览器打开 `http://127.0.0.1:8001/`

### A2 — 体验

- [x] 进示教时 state→滑条同步；断连 / WS 断开 Mute
- [x] 试播进度（读 status/result 推 WS）
- [x] 单向导入 gui_control presets
- [x] 默认 `127.0.0.1`；绑全接口需显式参数

### A3 — 服务化（与 Phase 2 对齐）

- [x] `SavePose` / `ListPoses` / `DeletePose` / `ImportGuiPresets` Service
- [x] Web / CLI / Service 共用 `pose_library`

**验收（A1）**

1. 仅起 `gesture_node` + `teach_panel`，浏览器完成示教→保存→试播
2. TEACH 下拖滑条，节点不抢发；试播 / WS 断开时 Mute
3. L20 参数下文件落在 `l25.json`
4. 未装 `[teach]` 时核心包仍可 build / 跑原语节点

---

## 10. 风险与对策

| 风险 | 对策 |
|------|------|
| UI 与 Executor 双发 cmd | 状态机 Mute + Executor TEACH 门控 |
| 未 `mode teach` 就拖滑条 | 顶栏强制「开始示教」；非 TEACH_EDIT 忽略 set_joints |
| WS 断线仍发关节 | 连接断开立即 Mute |
| 局域网误暴露 | 默认 `127.0.0.1`；绑全接口需显式参数 |
| 滑条洪水 | 服务端节流 10–20Hz |
| status 被 override 盖住 mode | 解析失败保守 Mute；A2 可加专用 mode 字段 |
| O20/L25 DOF 混用 | HandConfig 定长；state 长度不符拒绝保存 |
| Web 依赖拖垮 CI | teach 为 extra；单测只测 bridge/controller |
| 与 gui_control 同时开 | 文档警告 |

---

## 11. 接口速查（目标形态）

```bash
# 按 ../../docs/workbench.md 的独立 Jazzy 流程安装 Pydantic 2 / Web 栈，
# 使用同一系统 Python venv 构建并加载 overlay。
export PYTHONNOUSERSITE=1

ros2 launch hand_gesture_primitives teach_panel.launch.py \
  hand_side:=left hand_type:=l25

# 浏览器打开
# http://127.0.0.1:8001/（右手 8002）
```

面板内等价于：

```text
mode teach | mode primitive | pose <name> | pose_list
+ TEACH 时发布 /cb_{side}_hand_control_cmd
+ 服务端调用 pose_library.*
```

---

## 12. 总结

| 项 | 决策 |
|----|------|
| 路线 | **长期 A：本包独立 Teach 面板** |
| View | **轻量 Web**（FastAPI + 静态页 + WebSocket）；不用 PyQt |
| 角色 | Authority 切换 + 硬件滑条编辑 + PoseLibrary 浏览器 |
| 执行 | 全部委托 `gesture_node`；试播 = `pose` |
| 库 | 单一 JSON 桶；不与 SDK preset 双向同步 |
| 依赖 | `[teach]` 可选；核心栈零 Web/GUI |
| 绑定 | 默认 `127.0.0.1:8001`（左）/ `:8002`（右） |
| 顺序 | A0 状态机 → A1 MVP Web → A2 体验/导入 → A3 Service |

一句话：**浏览器管交互；服务端管 Mute 与库；Executor 管安全执行。**

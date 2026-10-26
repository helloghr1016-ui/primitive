# 原语量纲迁移方案：内部弧度 + 0～255 过渡 + 角度仅输入/显示

> **范围**：默认仅 `hand_gesture_primitives`（含包内 Teach / Pose Library / 单测 / 本包文档）。
> **驱动**（`linkerhand_hardware` 等）**本阶段不改**；机载已对 controller 发 URDF 弧度。
> **状态**：执行核为 **semantic_rad @ S30（30 维）**；0～255 仅边界 / 退化路径。

---

相关：`semantic_30_refactor.md`（协议与通道表）。

## 1. 目标与非目标

### 1.1 目标

| 项 | 说明 |
|----|------|
| 原语内部唯一量纲 | **弧度 rad**（**S30 语义 30 槽**布局，值单位为 rad） |
| 0～255 hardware | **过渡态**：SDK / 旧 YAML / Pose v1 / Teach 滑条兼容 |
| 角度 deg | **仅输入糖与 UI 显示**；不进执行核、不进 cmd topic、默认不落盘 |
| 对外控制契约不变 | `ros2_control_command_units=radians` 时，发往 `/{side}_hand_controller/commands` 的向量与**迁移前数值兼容**（同序、同限位、同 invert） |
| 驱动可晚升级 | 本方案不依赖驱动改动；驱动仍可内部做 rad↔SDK |

### 1.2 非目标（本阶段不做）

- 修改 `linkerhand_hardware` / 其它硬件插件 / 整机 URDF
- 执行核三选一并行（rad / 255 / deg 同时插值）
- 跨手型裸弧度向量互导（不经 mapping）
- 强制清空已有 Pose Library（v1 hardware 必须可读）
- 把触觉压感 0～255、电流/力矩阈值改成弧度（与位置量纲无关）

---

## 2. 量纲分层（定稿）

```
        degrees（UI 显示 / YAML 糖）
                 ↕  deg2rad / rad2deg（仅边界）
┌────────────────────────────────────────────────┐
│  原语内部：semantic_rad                         │
│  · 布局：S30 语义 30 槽（`semantic_30.py`）       │
│  · 单位：rad                                    │
│  · PrimitiveResult.space="semantic_rad"        │
└─────────────────┬──────────────────────────────┘
                  │
       ┌──────────┴──────────┐
       ▼                     ▼
 hardware 0–255           controller radians
 （过渡适配器）             （主路径，与现网一致）
 Pose v1 / Teach/SDK       Float64MultiArray
```

### 2.1 三种形式的职责

| 形式 | 职责 | 禁止 |
|------|------|------|
| **rad（内部）** | `compute()`、插值引擎、步进、PlanGesture 名义轨迹源 | 与 255 混插值 |
| **0～255** | 过渡读写、NamedPose hardware 直出、旧 bundle | 作为长期唯一内部态 |
| **deg** | YAML `space: degrees`、Teach 刻度显示 | `PrimitiveResult` / cmd topic / Bundle 默认落盘 |

### 2.2 `PrimitiveResult.space`

| 值 | 含义（迁移后） |
|----|----------------|
| `semantic_rad`（默认） | S30 语义槽 + **弧度**；有 Converter 时的执行核量纲 |
| `semantic` | S30 legacy **0～255**（无 Converter / 退化路径） |
| `hardware` | 过渡：产品 DOF 的 0～255 直出（NamedPose 等） |

不增加 `degrees` / `radians` 作为 Result.space（避免三轨执行）。
代码常量见 ``angle_bridge.SPACE_SEMANTIC_RAD`` / ``SPACE_LEGACY_SEMANTIC255`` / ``SPACE_HARDWARE``。
---

## 3. 范围边界

### 3.1 In scope（本包）

- `gesture_engine` / `executor` / `joint_state_mapping`（Converter）
- `hand_config` / `gesture_params` YAML 加载
- `fk_solver`（今日亦经 255↔URDF，须一并切内部 rad）
- 各 `primitives/*` 中写死的 255 目标与 step（改为 rad 或经适配加载）
- Pose Library / Bundle（v1 兼容 + 可选 v2）
- Teach 面板（显示单位；落盘策略）
- 包内单测与 `docs/*`

### 3.2 Out of scope（明确不改）

| 模块 | 原因 |
|------|------|
| `linkerhand_hardware` | 驱动后续升级；现已 `interface_units=radians` |
| `luban_bringup` | 已传 `ros2_control_command_units=radians`；最多补文档一句 |
| `control` / 臂规划 | 不消费手部 255 语义 |
| URDF / xacro | 限位仍作 rad 真源 |
| `framework_capabilities` | PlanGesture 输出已是 controller 弧度；保持输出兼容则可不改 |

### 3.3 对外不变式（验收硬条件）

在相同 `robot_description`、相同手型/侧、相同原语与初始状态下：

1. **radians 模式**：迁移前后 `commands.data` 逐关节误差 &lt; ε（建议 ε=1e-6，或文档约定 1e-4）
2. **hardware 模式**：仍发布 clamp 后的 0～255 产品向量
3. **PlanGesture**：`joint_names` + 弧度轨迹与现网同契约
4. **Pose v1**：旧 `{canonical}_{side}.json` 可导入、可回放

---

## 4. 内部弧度定义

### 4.1 布局

- 使用 **S30 语义索引 0～29**（`SEMANTIC_30_NAMES`；O20 经 `project_o20_to_s30` 兼容）。
- 未映射 / 禁用通道固定 `0.0` rad。
- 各手型经 **`from_semantic`**（可由 `from_o20` 合成）映射到产品关节；槽内数值单位为 rad。

### 4.2 与 URDF / controller 的关系

对当前手型，槽位 \(i\) 的 rad 值对应「该语义关节在**参考 URDF 限位**下的角位置」（与现 `Ros2ControlRadianConverter` 使用的 limit 一致）：

**迁移前（现状）**

```text
semantic_255 → ratio → invert? → lower + ratio*(upper-lower) = q_rad
```

**迁移后（等价）**

```text
semantic_rad = q_rad（语义槽）
发布：按 controller 关节序重排 + clamp(limit)   // 不再经 /255
```

过渡适配：

```text
legacy_255 → 用同一 limit/invert 表 → semantic_rad
semantic_rad → 反算 → legacy_255（Pose v1 / hardware 模式）
```

实现上应把现 Converter 的「255 中间态」收拢进 `units` 适配 API，执行热路径走 rad。

### 4.3 依赖 `robot_description`

与现 radians 模式相同：无 URDF 则无法建立 rad 语义。
**MOCK / 真机 / 手-only launch 须继续传入 `robot_description`**（已满足机载路径）。

---

## 5. 转换层设计（新建 `units.py` 建议）

集中实现，禁止各原语私自 `/255`：

| API（示意） | 用途 |
|-------------|------|
| `deg_to_rad` / `rad_to_deg` | 输入/显示 |
| `legacy_semantic255_to_rad(hand, side, vec20)` | 旧内部态 / 旧 YAML |
| `rad_to_legacy_semantic255(...)` | 回写测试、compat |
| `hardware255_to_rad` / `rad_to_hardware255` | Pose v1、Teach、NamedPose |
| `semantic_rad_to_controller` / `controller_to_semantic_rad` | Executor 发布/反馈 |

所有路径共用同一套 limit + invert + joint_order（G20 `l25_urdf` 含在内）。

---

## 6. 各子系统契约

### 6.1 YAML `gestures.*.angles.space`

| space | 加载行为（当前实现） |
|-------|----------|
| `semantic_o20`（默认） | legacy 0～255 语义向量；执行前由 ``*_as_semantic_rad`` / ``AngleUnits`` 转 rad |
| `hardware` | 0～255 产品向量 → legacy 语义 |
| `radians` / `rad` / `degrees` / `deg` | **仅** ``load_static_params_for_ctx`` / ``load_thumb_adduction_params_for_ctx``（运行时 + Converter）；import 期 ``load_static_gesture_params`` 会显式拒绝 |

新配置写物理量纲请走运行时 loader；import 期公开 loader 保持 ``semantic_o20``。

L25 上 `motion.type: static` 的 by_vision 原语经 ``load_static_params_for_ctx`` / ``static_yaml_legacy255_or_none`` **读取** `l25.yaml` 目标（不再旁路硬编码）。
### 6.2 速度 / 步进

| 旧（255 刻度） | 新 |
|----------------|-----|
| 位置 step / tick（255 单位） | rad/tick 或 rad/s，按 `control_rate` 换算 |
| YAML `speeds` 等 | 允许 `unit: rad_s \| deg_s \| legacy_255_s`，加载时规范化 |

**力/电流/力矩阈值**：保持原单位，不换算成弧度。

### 6.3 Pose Library / Bundle

| 版本 | 落盘 |
|------|------|
| **v1（过渡，默认仍支持）** | `space: hardware`，0～255 |
| **v2（可选后续）** | `space: radians` + 手型/侧/可选 `joint_names` |

- deg **不**作为 Bundle 主 space。
- 导出：T1 可继续 v1；T2 提供「导出 radians」开关。
- 导入：v1→内部 rad；v2 直读。

### 6.4 Teach 面板

- **内部/下发**：与 gesture 一致，优先 rad（`ros2_control`）。
- **显示**：可切换 deg / 0～255（显示层）。
- **保存**：T1 仍可写 Pose v1 hardware（兼容迁移）；T2 可写 v2 rad。

### 6.5 STREAM

- T1：`stream_target` 仍可接 hardware 0～255（文档标明过渡）。
- T2：增加 `units` 或并行 rad 流；默认切 rad。

### 6.6 FK

有 URDF Converter 时 Executor 走 ``compute_fingertips_rad``（内部 semantic_rad），与 ``command_units`` 发布边界解耦；无 Converter 时仍用 legacy255 ``compute_fingertips``。

### 6.7 NamedPose

- `space=hardware`：过渡直出 0～255，Executor 在 hardware 模式直发；radians 模式下先 `hardware→rad→controller`。
- 禁止新增 `space=degrees` 落盘。

### 6.8 `PrimitiveContext` 过渡字段

| 字段 | 约定 |
|------|------|
| `compute(current_angles)` | 有 Converter 时为 **semantic_rad**；无则 legacy255 |
| `angle_unit` | `semantic_rad` / `semantic` |
| `radian_converter` | 有 URDF 时注入，供 ``AngleUnits`` |
| `hand_hw_angles` | 产品 DOF **0～255**（NamedPose / 示教对齐） |
| `hand_cmd_angles` | 同上，最近下发命令的 hardware 重建 |
| 电流 / 力矩 / 触觉 | 原单位不变 |

### 6.9 日志与调试

- status/result 字符串可继续打印原语名；若打印关节值须标注单位（`rad` / `hw255`）。
- 避免无单位地 dump 一串 float，防止运维按 255 解读 rad。

---

## 7. 实施阶段

### T0 — 现状（已满足对外）

- 内部 255 语义；边界 `radians` 下发；驱动 rad↔SDK。

### T1 — 内部切 rad（**推荐首期**，仅本包）

1. 新增 `units.py` + 单测（往返、invert、G20/O6/L25）。
2. Converter / Executor / 反馈路径内部改 rad；**golden**：同输入 → 同 `commands`。
3. YAML 加载支持四 space；默认仍能读旧 `semantic_o20`。
4. FK 跟随内部 rad。
5. 文档：`interfaces.md` + 本文；README 链到本文。
6. **不改**驱动、bringup 参数默认值。

### T2 — 体验与落盘

1. Teach 显示 deg；Pose 可选 v2 radians 导出。
2. STREAM / 文档标 hardware 为 deprecated。
3. 步进/速度 YAML 显式单位。

### T3 — 淡出 255

1. 新落盘默认 radians。
2. `command_units=hardware` 与 Pose v1 标 deprecated。
3. 驱动侧若取消 255 接口，删除本包 hardware 适配器。

---

## 8. 测试与验收

| 用例 | 期望 |
|------|------|
| `legacy255 → rad → controller` vs 旧 `semantic_to_radians` | 逐关节近似相等 |
| `rad → legacy255 → rad` | 往返误差可控 |
| `deg` YAML 与等价 `rad` YAML | 同终态 |
| Pose v1 存取 + `pose name` 回放 | 与迁移前一致 |
| PlanGesture fist/open | success；轨迹连续 |
| MOCK=1 GESTURE 发 open/fist | 肉眼/日志与迁移前一致 |
| hardware 模式（若 CI 有） | 仍为 0～255 |

建议增加 **golden 向量 fixture**（固定 URDF 片段 + 固定 semantic 输入 → 固定 commands）。

---

## 9. 风险与对策

| 风险 | 对策 |
|------|------|
| 某原语硬编码 255 漏改 | grep `255` + 单测覆盖主要原语；T1 门禁 |
| 无 robot_description | 与现网相同：radians 模式直接失败并提示 |
| 力控 step 单位错 | 力控参数表集中换算；接触阈值不换位置单位 |
| Teach 与 gesture 单位不一致 | 启动参数对齐；面板标明显示单位 |
| Bundle 旧包 | v1 只读 hardware；checksum/导入策略保持 |
| 误改驱动 | Code owners / PR 描述写死 Out of scope |

---

## 10. 方案自检：已补遗漏项

相对初稿，本方案已显式纳入：

1. **FK** 必须随内部 rad 改（否则指尖/视觉门控漂）。
2. **对外 bit 兼容** 作为硬验收，不只是「概念兼容」。
3. **G20 `l25_urdf` joint_order** 走同一 Converter，不另开通道。
4. **力/触觉阈值与位置量纲分离**（避免误改电流阈值）。
5. **STREAM / NamedPose / Pose v1** 过渡策略写清。
6. **In/Out scope** 写死驱动与 bringup，避免范围膨胀。
7. **deg 不落盘、不作 Result.space**。
8. **保留位、invert、calibration** 仍在适配层，不进原语公式。
9. **控制频率缩放**（`scale_per_tick_delta`）随 rad/s 重标定。
10. **回滚**：T1 可用特性开关或 revert Converter 热路径；旧 YAML 始终可读。
11. **`PrimitiveContext.hand_*_angles`**：T1 明确保留 hardware 255，避免 NamedPose 回归。
12. **日志必须带单位**，防止 rad 被当成 255。

### 10.1 仍属后续、不阻塞 T1

| 项 | 说明 |
|----|------|
| Pose Bundle「幂等 skip 算成功」 | 产品体验，与量纲无关；可另 PR |
| O20 allowlist 有 pinch、gestures 无配置 | 规划已拒绝；执行仍可能走默认——属配置治理 |
| ImportGuiPresets 与 bundle merge 统一 | 工程债 |
| 驱动删除 255 接口 | T3 + 驱动仓 |

---

## 11. 建议开工顺序（T1）

1. `units.py` + golden 单测
2. `Ros2ControlRadianConverter` / Executor 内部 rad，保持 commands 兼容
3. `gesture_params` 四 space 加载
4. FK
5. 批量替换原语/引擎中的 255 常量（可分多 PR）
6. 文档与 README 链接

**T1 落地说明（已收口）**：

- 有 ``robot_description`` 时：Executor 向原语传入 **semantic_rad**，
  ``ctx.angle_unit=semantic_rad``；静态/力控/硬编码姿态经 ``AngleUnits`` /
  ``*_as_semantic_rad`` 统一在 rad 插值与步进。
- **0～255 仅边界**：Pose Library / STREAM / NamedPose ``hardware`` /
  ``ros2_control_command_units=hardware`` 发布；传感器压感等与位置量纲无关。
- 无 Converter 时退化为 legacy255（``AngleUnits.rad=False``），便于离线缺 URDF。
- ``PrimitiveResult.space`` 默认 ``semantic_rad``；hardware 直出须显式声明。

---

## 12. 一句话

**在 gesture primitives 包内把执行核换成弧度；0～255 只做进出适配；角度只给人看和写；发到控制器的弧度向量与现网对齐，驱动暂不改。**

# 模型 L1 接入说明

策略 / VLM / 规划器输出 **离散手势名**，直接发到现有指令通道，无需 STREAM。

## 通道（luban 侧化）

| 场景 | Topic |
|------|-------|
| 经 GraspGate | `/{side}_hand/gesture_cmd` → Gate → `/{side}_hand/gesture_cmd_exec` |
| 无门控 | 直接 `/{side}_hand/gesture_cmd` |

消息：`std_msgs/String`，整句会 `.lower().split()`。`mode` / `pose` / `pose_list` / `stop` **不进门控**，直通。

## 推荐输出

```text
open
init
index_ring_by_vision
pose my_pinch
mode teach          # 一般不由模型发；示教用人机面板
mode primitive
stop
```

自定义姿态须先经 Teach 面板 / `save_pose` / Service 入库；模型只引用已有名称。

## 与 STREAM 的边界

| 档位 | 何时用 |
|------|--------|
| **L1（本页）** | 离散原语 / `pose <name>` — **优先** |
| STREAM | 连续关节角 @N Hz；`mode stream` + `/{side}_hand/stream_target` |

过流保护在两种模式下均由 Executor 负责。

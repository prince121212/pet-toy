# 宠物陪伴微信小程序开发计划

## 1. 目标

做微信小程序，不做微信小游戏。

目标不是“播放宠物动画”，而是让用户感觉手机里有一只会生活、会回应、有状态的小宠物。

首版不要接入 AI 做实时决策。先用本地状态机保证稳定体验；后续 AI 只做高层个性、聊天、日记和权重调整。

## 2. 核心原则

| 原则 | 做法 |
|---|---|
| 动作不硬切 | 只在动作结束或安全出口切换 |
| 播放类型只保留两种 | `loop`、`transition` |
| 决策本地完成 | AI 超时或无网络时也能正常互动 |
| 资源不进包 | 动作素材放 CDN/云存储，包内只放播放器和兜底图 |
| 少素材先验证 | MVP 只做 8 个高复用动作 |
| 宠物可替换 | 猫、狗、鸟、IP 都使用同一套动作 ID |

## 3. 系统结构

```text
用户事件 / 时间 / 宠物数值
        ↓
BehaviorStateMachine 行为状态机
        ↓
DecisionEngine 本地决策器
        ↓
ActionPlayer 动作播放器
        ↓
CanvasRenderer 帧渲染
```

| 模块 | 职责 |
|---|---|
| `ActionPlayer` | 播放 `loop` 或 `transition` |
| `BehaviorStateMachine` | 管理清醒、睡眠、开心、烦躁等状态 |
| `DecisionEngine` | 决定当前动作播完后接哪个动作 |
| `ResourceManager` | 拉取 manifest、下载、缓存、释放资源 |
| `PetModel` | 保存亲密度、精力、心情、上次互动时间 |
| `MemoryGuard` | 监听内存告警，清理非当前资源 |

## 4. 播放机制

| 类型 | 说明 | 例子 |
|---|---|---|
| `loop` | 可长期循环的状态动作 | `idle_breathe_01`、`sleep_loop_01` |
| `transition` | 播放一次后进入目标动作 | `touch_affection_01`、`wake_up_01` |

用户事件不一定立刻切动作。

例子：

```text
sleep_loop_01 播到第 2 秒
用户触摸
事件进入队列
当前睡眠循环播到安全出口
播放 wake_up_01
播放完回到 idle_breathe_01
```

## 5. MVP 动作集

| 动作 ID | 类型 | 用途 |
|---|---|---|
| `idle_breathe_01` | `loop` | 默认清醒待机 |
| `sleep_loop_01` | `loop` | 睡眠呼吸 |
| `touch_affection_01` | `transition` | 被摸后的亲密反馈 |
| `touch_annoyed_01` | `transition` | 高频点击后的不满反馈 |
| `play_object_01` | `transition` | 玩具互动 |
| `sleep_enter_01` | `transition` | 清醒进入睡眠 |
| `wake_up_01` | `transition` | 睡眠醒来 |
| `idle_blink_01` | `transition` | 待机微动作 |

这 8 个动作已经可以验证生命感。

## 6. 状态和规则

| 状态 | 常驻动作 | 典型触发 |
|---|---|---|
| `idle` | `idle_breathe_01` | 默认状态 |
| `sleeping` | `sleep_loop_01` | 长时间无操作、精力低 |
| `affection` | transition 后回 `idle` | 单次轻触 |
| `annoyed` | transition 后回 `idle` | 高频点击 |
| `playing` | transition 后回 `idle` | 点击玩具 |

基础规则：

| 事件 | 当前状态 | 下一动作 |
|---|---|---|
| 单次触摸 | `idle` | `touch_affection_01` |
| 高频触摸 | 任意清醒状态 | `touch_annoyed_01` |
| 点击玩具 | 清醒状态 | `play_object_01` |
| 长时间无操作 | `idle` | `sleep_enter_01` -> `sleep_loop_01` |
| 睡觉时触摸 | `sleeping` | `wake_up_01` -> `idle_breathe_01` |
| 空闲随机 | `idle` | `idle_blink_01` |

## 7. 决策机制

动作播放期间收集事件。动作结束前冻结下一步决策。

```text
当前动作播放中
↓
事件进入队列
↓
本地状态机给出兜底动作
↓
可选 AI 给出建议或调整权重
↓
结束前 300-500ms 冻结 nextAction
↓
动作结束后播放 nextAction
```

AI 可以参与：

| 可用 AI | 不建议 AI |
|---|---|
| 今日性格权重 | 实时控制每帧 |
| 聊天回复 | 决定所有动作 |
| 宠物日记 | 阻塞动作切换 |
| 长期记忆 | 无兜底决策 |

## 8. 资源策略

小程序包内：

- 播放器代码
- 状态机代码
- 占位图
- 极轻量兜底资源

CDN/云存储：

- 动作透明帧序列
- 音效
- 角色 manifest
- 背景素材

运行时：

- 只加载当前动作。
- 可预加载下一个高概率动作。
- 内存告警时释放所有非当前资源。
- 低端机使用低清资源档。

## 9. 开发阶段

| 阶段 | 目标 | 验收 |
|---|---|---|
| P0 原型 | Canvas 播放 1 个透明动作 | 真机流畅，无明显内存增长 |
| P1 播放器 | 支持 `loop` / `transition` / 安全出口 | 动作不硬切 |
| P2 资源系统 | CDN manifest + 本地缓存 | 不发版可换资源 |
| P3 状态机 | 8 个 MVP 动作跑通 | 触摸、睡觉、醒来、玩具可用 |
| P4 体验层 | 声音、气泡、记忆数值 | 用户觉得它有状态 |

## 10. 完成感标准

做到这些，才像“小生命”：

- 睡觉时被摸，不立刻硬切，而是自然醒来。
- 连续乱点会不满。
- 很久没来，打开时状态不一样。
- 不操作时也会眨眼、呼吸、偶尔动一下。
- 动作切换不瞬移、不缩放、不跳位置。
- 声音克制，不每次都叫。
- 用户感觉“它认识我”，不是“我点了一个动画按钮”。

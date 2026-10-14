---
name: 手机控制
aliases: [android-operator, 手机操控, 手机agent, 主副agent手机]
description: 主副 Agent 手机操控框架：通用 Base Actions（看/点/输入/滑/截）+ Sub-Agent 军师模式（主玩副分析）+ 差分感知省 token。适用于电商比价、战棋对战、挂机等「看屏→决策→操作」场景。
---

# 手机控制（android-operator）— 主副 Agent 手机操控框架

> 一套把「Agent 控制手机」做成可复用能力的 skill。核心三件套：
> ① **通用 Base Actions**（看/点/输入/滑/截）② **Sub-Agent 军师模式**（主玩、副分析）
> ③ **差分感知**（只喂屏幕变化，省 token、提速）。
> 适用于多场景：App 比价、战棋对战、挂机、表单填写……换 skin 即可复用。

---

## 0. 适用前提

本 skill 运行在 **DSHA（DeepSeek Harness Android）** 环境，依赖其 3090 桥端点：

- 读屏 `/app/ui/dump`、点按 `/app/ui/tap`、输入 `/app/ui/input`、滑动 `/app/ui/swipe`、截屏 `/app/ui/screenshot` 已可用（无障碍 + 截屏权限已开）。
- 设备通道（Shizuku / ADB）可选：连上后解锁 `vscreen` 与 `adb-shell`，可用于**本地脚本循环**与**更稳的真实触摸**。

若通道不可用，先按 `shizuku-setup.md` 连接；连接前也能用无障碍通道跑本 skill 的导航部分。

---

## 1. 架构总览

```
┌──────── 手机（DSHA / 无障碍 + 截屏）────────┐
│  感知层：dump / screenshot  → 屏幕状态        │
│  执行层：tap / input / swipe / 脚本（可选）    │
└─────────────────────────────────────────────┘
              ▲ 屏幕流/状态摘要        ▲ 结构化事件
              │                       │
┌─────────────┴──────────┐   ┌────────┴──────────────┐
│  主 Agent（Operator）    │   │  副 Agent 集群（军师）  │
│  · 维护任务状态机 state   │   │  · 角色见 subagent_roles│
│  · 决定下一步操作         │◀──│  · 吃结构化输入，不碰屏  │
│  · 调用执行层            │返回│  · 返回「建议+置信度」   │
│  · 只在需算计时派发       │建议│  · 可并行、可换模型      │
└────────────────────────┘   └─────────────────────┘
```

**设计原则（务必遵守）**
1. **感知节流**：每步不裸 dump 全屏；按 `perception.md` 只取 diff。
2. **决策分流**：纯导航主 Agent 自己干；遇到「需要外部计算/分析」才 `delegate()`。
3. **子 Agent 不碰屏幕**：输入是文本化结构数据，输出是建议，绝不调用设备工具。
4. **复用优先**：换场景只换 `scenarios/` 的 skin 与副 Agent 的 role 提示词，主框架不动。

---

## 2. 主 Agent 操作纪律（must）

手机 UI 是**状态机**，不是脚本。每一步都必须：

1. **动作前**：用 `dump`（或 `screenshot`）确认当前界面，定位目标坐标/文字。
2. **动作后**：立即再次 `dump` 验证结果，**界面变了就重新观察，绝不凭记忆连点**。
3. **节奏**：点按/输入/滑动后都要重新读屏再决定下一步（见 `base_actions.md` 的 `observe_after` 约定）。
4. **失败处理**：坐标点空 / 页面没变 → 先分析新状态，不要反复点旧位置；可 `screenshot` 存图人工核对。
5. **敏感操作交还用户**：付款、验证码、改凭证、对外提交，必须停手让本人接手。

---

## 3. Sub-Agent 军师模式（delegate 契约）

主 Agent 在以下情形调用副 Agent（通过 `subagent` / `subagent_fork` 工具，或 workflow 并行）：

| 触发情形 | 派发角色 | payload（结构化） | 副 Agent 返回 |
|---|---|---|---|
| 需要横向对比多个对象 | `analyst-price` | 商品参数列表 | 推荐项 + 理由 + 置信度 |
| 战棋/对战需推演敌方 | `tactics-advisor` | 双方阵容/资源/棋面 | 下一步建议 + 风险 |
| 通用复杂分析 | `general-analyst` | 任意结构化数据 | 结论 + 依据 |

**契约（写进派发提示词）**
- 输入：纯文本/JSON，**不依赖截图权限**。
- 输出：第一条消息即结论，含 `recommendation` 与 `confidence(0~1)`。
- 约束：**你没有任何设备操作权限，只能分析**。若信息不足，明确列出缺什么。

详见 `subagent_roles.md`。

---

## 4. 差分感知（perception）

全屏 dump 每步都喂模型很费 token 且慢。规则见 `perception.md`：
- 维护一份 `last_state` 摘要（已见过的控件/关键文本）。
- 每次 `dump` 后只把「新增 / 消失 / 变化」的部分进入推理上下文。
- 仅在状态跳转不确定时，才 `screenshot` 全图人工/视觉核对。

---

## 5. 快速开始

1. 确认通道：`/app/ui/dump` 返回界面而非 `SCREEN_TARGET_UNAVAILABLE`。
2. 选场景 skin：`scenarios/shopping.md`（比价）或 `scenarios/tactics_game.md`（战棋）。
3. 按 skin 里的「主 Agent 流程」推进；遇到分析节点，按 `subagent_roles.md` 派发军师。
4. 如需本地脚本循环 / 更稳真实触摸，先连 Shizuku（见 `shizuku-setup.md`）。

---

## 6. 文件索引

| 文件 | 作用 |
|---|---|
| `SKILL.md`（本文件） | 主操作纪律 + 架构 + delegate 契约 |
| `base_actions.md` | 通用动作库，对齐 DSHA 3090 端点 |
| `perception.md` | 差分感知节流规则 |
| `subagent_roles.md` | 副 Agent 角色模板与提示词骨架 |
| `scenarios/shopping.md` | 拼多多比价实战（已验证） |
| `scenarios/tactics_game.md` | 战棋主玩 + 军师模式（设计稿） |
| `shizuku-setup.md` | Shizuku 连接最小步骤 |

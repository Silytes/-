# android-operator — 主副 Agent 手机操控框架

把「Agent 控制手机」做成可复用能力。融合三件套：
① **通用 Base Actions**（看/点/输入/滑/截）② **Sub-Agent 军师模式**（主玩、副分析）③ **差分感知**（只喂变化，省 token 提速）。

## 文件结构

```
android-operator/
├── SKILL.md          # 主操作纪律 + 架构 + delegate 契约（先看这个）
├── base_actions.md   # 通用动作库，对齐 DSHA 3090 端点
├── perception.md     # 差分感知节流规则
├── subagent_roles.md # 副 Agent 角色模板（比价军师/战棋军师/通用分析师）
├── scenarios/
│   ├── shopping.md       # 拼多多比价实战（已真机验证）
│   └── tactics_game.md   # 战棋主玩+军师（设计稿，待实战）
└── shizuku-setup.md  # Shizuku 连接最小步骤
```

## 适用场景
- 电商比价、挂机、表单填写、任何「看屏→决策→操作」任务
- 战棋 / 自走棋 / 回合制对战：**主 Agent 自己运营操作，副 Agent 当军师推演敌方**

## 快速开始
1. 确认 `/app/ui/dump` 可用（无障碍+截屏已开）。
2. 选 skin：`scenarios/shopping.md` 或 `scenarios/tactics_game.md`。
3. 按 skin 流程推进；需分析时按 `subagent_roles.md` 派发军师。
4. 要本地脚本循环/更稳真实触摸 → 先连 Shizuku（`shizuku-setup.md`）。

## 复用原则
换场景只改 `scenarios/` 的 skin 与副 Agent 的 role 提示词，主框架（SKILL/base/perception）不动。

## 来源与致谢
设计借鉴 [ghost-in-the-droid/android-agent](https://github.com/ghost-in-the-droid/android-agent) 的
Base Actions / Sub-Agent / Differential A11y 思路，落地于 DSHA 3090 桥端点；
真实触摸补充方案参考 [shenlan-ai/esp32-android-agent](https://github.com/shenlan-ai/esp32-android-agent)。

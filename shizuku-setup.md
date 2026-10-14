# Shizuku 连接最小步骤（DSHA 设备通道）

> 目的：让 DSHA 的 `vscreen` 与 `adb-shell` 从 `DEVICE_CHANNEL_UNAVAILABLE` 变成可用。
> 连上后解锁「本地脚本循环」「更稳真实触摸」「虚拟屏」。

---

## 为什么选 Shizuku（不是裸 ADB）

| | Shizuku | 裸 ADB 无线调试 |
|---|---|---|
| 权限 | API 级（读屏/触摸/自身数据），无 shell | 完整 shell |
| 风险 | 小，不监听裸端口 | 大，同 WiFi 暴露 ADB 端口 |
| 回收 | 较稳，可配防回收 | 易掉，需重配对 |
| 建议 | ✅ 优先 | 仅本地脚本深度需求时 |

**结论**：先用 Shizuku。只有要做「手机本地跑 Python 循环注入复杂命令」才上裸 ADB。

---

## 步骤（小米 / Android 12+ 通用，其他 ROM 类似）

### 1. 开开发者选项
设置 → 关于手机 → 版本号，连点 7 下 → 提示「已进入开发者模式」。

### 2. 开无线调试（Shizuku 的取权来源）
设置 → 系统 → 开发者选项 → **无线调试** → 打开。
- 进入「无线调试」页，点 **使用配对码配对设备**，记下 **IP:端口** 与 **6 位码**（每次进入会变）。

### 3. 安装并启动 Shizuku
- 应用商店 / GitHub 装 **Shizuku**（作者：RikkaApps）。
- 打开 Shizuku → 选「**通过无线调试启动**」→ 填入步骤 2 的 IP:端口 + 6 位码 → 配对。
- 配对成功后 Shizuku 显示「**正在运行**」。

### 4. 在 DSHA 里连设备通道
- 打开 **DSHA App → 设置 → 设备能力授权**。
- 选 **Shizuku**（非 ADB / 非 Root）→ 授权 DSHA 使用 Shizuku。
- 状态变为「**已连接**」。

### 5. 验证（在对话里让我跑）
```bash
curl -s -H @/root/.dsh/.bridge_headers "http://127.0.0.1:3090/app/vscreen/create?orientation=portrait"
# 期望：{"ok":true,...}  而非 DEVICE_CHANNEL_UNAVAILABLE
timeout 15 /root/dsh-bin/adb-shell "id"
# 期望：uid=...  而非 DEVICE_CHANNEL_UNAVAILABLE
```

---

## 回收处理（常见坑）

若用着用着通道又掉（`DEVICE_CHANNEL_UNAVAILABLE` 或无障碍 `SCREEN_TARGET_UNAVAILABLE`）：
1. 系统可能回收了 Shizuku / DSHA 无障碍 → 重进 Shizuku 点「启动」，重进系统无障碍确认 DSHA 开关仍开。
2. 给 DSHA 与 Shizuku 设「**电池无限制 / 自启动**」，防止后台被杀。
3. 可选装 `androidAccServerManage`（Shizuku 管理无障碍权限工具）防无障碍被自动关。
4. 杀掉 DSHA App 重开，让服务重新绑定。

---

## 连上之后能做什么（本 skill 的进阶用法）

- `adb-shell` 里跑 `input tap/swipe`：比无障碍更稳的真实触摸（部分游戏必要）。
- 写**本地脚本**循环：读屏→判断→点击，机器节奏跑，只在高层决策叫主 Agent。
- `vscreen` 虚拟屏：多任务并行不挤占真屏。

> 注意：设备 shell 受 DSHA 策略层约束（`[POLICY_BLOCKED]` 仍会拦敏感命令），Shizuku 不等于绕过安全。

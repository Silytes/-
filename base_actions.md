# Base Actions — 通用手机操控动作库

> 对应 DSHA 3090 桥端点。所有动作都遵循「**动作前观察 → 执行 → 动作后验证**」的闭环。
> 这是 ghost-in-the-droid 的 "Base Actions" 思路在 DSHA 上的落地版。

---

## 端点速查（凭据走 header，勿拼进 URL）

```
读屏   curl -s -H @/root/.dsh/.bridge_headers "http://127.0.0.1:3090/app/ui/dump"
点按   curl -s -G -H @/root/.dsh/.bridge_headers "http://127.0.0.1:3090/app/ui/tap" --data-urlencode "text=搜索"
       curl -s -G ... "/app/ui/tap" --data-urlencode "x=720" --data-urlencode "y=1600"
输入   curl -s -G ... "/app/ui/input" --data-urlencode "text=关键词"
滑动   curl -s -G ... "/app/ui/swipe" --data-urlencode "x1=720" --data-urlencode "y1=2600" \
              --data-urlencode "x2=720" --data-urlencode "y2=900" --data-urlencode "ms=600"
截屏   curl -s -H @/root/.dsh/.bridge_headers "http://127.0.0.1:3090/app/ui/screenshot"
按键   curl -s -G ... "/app/ui/key" --data-urlencode "name=back"
启动   curl -s -G ... "/app/launch" --data-urlencode "pkg=com.xunmeng.pinduoduo"
```

> 中文/空格参数一律 `--data-urlencode`，别手写编码。`tap` 优先按 `text=`，无文字再用 `x=&y=`。

---

## 动作定义（每个都是闭环）

### observe(before/after)
- **目的**：读当前界面控件树。
- **输出**：`[序号] "文字" 可点击 中心=(x,y) 区域=l,t,r,b`。
- **纪律**：动作前必做；动作后必再做一次验证。
- **节流**：见 `perception.md`，只把 diff 进上下文。

### tap(target)
- `target` 为文字 → `tap?text=...`（控件位置会变，文字不变，优先）。
- `target` 为坐标 → `tap?x=&y=`（兜底，当文字缺失 / 点按钮失败时用）。
- **实测坑**：拼多多搜索页「搜索」按钮 `tap?text=搜索` 两次失效，最后用坐标 `(1331,255)` 成功。
  → 规则：文字点不到时，**立即改坐标点**，不要反复试同一条。

### input(text)
- 填到当前焦点框；没有焦点先 `tap` 输入框。
- 验证：input 返回「已输入 N 个字符」后，再 `dump` 确认框内文本。

### swipe(x1,y1,x2,y2,ms)
- 列表翻页 / 下拉刷新。ms 建议 300~700。
- 验证：swipe 后 `dump`，确认列表项已变化（首项/末项不同）。

### screenshot()
- 仅在「状态跳转不确定 / 需要人工核对 / 视觉模型分析」时调用。
- 返回 PNG 路径（应用截图目录），可导出到工作区供查看。

### launch(pkg) / key(name)
- `launch` 启动指定包名；`key` 支持 back/home/recents/notifications/quicksettings。
- 启动后等 2~5s 再 `observe`（冷启动有广告/加载）。

---

## 状态机约定（state 对象）

主 Agent 维护一个轻量 `state`（不必每帧全屏 OCR）：

```yaml
state:
  app: com.xunmeng.pinduoduo
  page: search_results        # home / search_box / search_results / detail / cart ...
  last_action: tap_price_sort
  key_nodes:                 # 本页关键控件缓存（文字→坐标）
    search_btn: (1331,255)
    price_tab: (659,424)
  pending_delegate: null      # 若有未回的子 Agent 任务
```

- 每次 `observe` 后更新 `key_nodes` 与 `page`。
- `page` 变化即状态跳转，触发对应流程分支（见 scenarios）。

---

## 失败与重试

| 现象 | 处理 |
|---|---|
| `SCREEN_TARGET_UNAVAILABLE` | 通道掉线，先重绑无障碍（见 shizuku-setup 的「回收处理」），不要疯狂重试 |
| 文字点不到 | 改坐标点；仍不行则 `screenshot` 看真实布局 |
| 页面没变 | 可能点了非交互层；重新 `observe` 找真目标 |
| 滑动无效 | 检查坐标是否落在可滚动区；加大 ms |

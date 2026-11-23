# Antigravity CLI (`agy`) 已知问题与排查记录

> 记录日期：`2026-11-23`（`Asia/Shanghai`）  
> 测试环境：`agy 1.5.14`（macOS）、订阅：Google AI Pro

---

## 一、核心现象与本机复现

在 `agy 1.5.14` 上使用 **`Gemini 4 Neon (High)`** 执行 `/pelican-test`（新建会话，仅执行过一次 `date` 命令，总输入约 `13k` tokens）时，任务长期卡在思考阶段无法完成。

通过读取本机对话数据库（`d0b0f696-d493-4042-ae6c-da07a1254af1.db`）及 CLI 日志（`cli-20261123_122249.log`、`cli-20261123_130408.log`、`cli-20261123_130922.log`）确认：

1. **并非单纯思考慢，而是持续 `unexpected EOF` 断流后从头重试**：
   - 会话 `d0b0f696…` 中，调用 0 及调用 2–7 均在思考了 **73s–111s**（约 `3,838–5,431` thinking tokens，尚未输出任何正文或工具调用）时遭遇 `stream reading error: unexpected EOF`。
   - 断流后客户端无法从断点续传，只能整轮从零重新思考，连续失败至 `attempt 7`，累计空耗约 3.5 万 thinking tokens。
2. **排除本地代理线路单一原因**：
   - 同一台机器、同一套网络下，**`Claude Opus 5.5 (High)`** 全程 **0 次 EOF**。
   - 更换 FlClash 出口节点（洛杉矶 `107.152.46.174`）并切换至规则模式后，Neon 新会话依然在 **80s–116s** 触发同样的 `unexpected EOF`。

---

## 二、GitHub Issues 归类分析

截至 `2026-11-23`，[`google-antigravity/antigravity-cli`](https://github.com/google-antigravity/antigravity-cli/issues) 拉取最新 800 条开放记录中共有 **781 个 open issues**（`#739–#1769`，仅 `11-16` 以来新增 181 个）。与本次异常直接相关的已知问题如下：

### 1. 长思考断流（`unexpected EOF`）与无法续传
- **[#1255](https://github.com/google-antigravity/antigravity-cli/issues/1255)**：`CreateCloudCodeHTTPClient` 禁用了 HTTP/2、TCP KeepAlive 与 `IdleConnTimeout`，导致长时间无输出流式请求在 ~60s+ 后被断开报 EOF。
- **[#1563](https://github.com/google-antigravity/antigravity-cli/issues/1563)**：流式请求遇 `io.EOF` 时无法断点续传，被迫整轮重头开始。
- **[#1444](https://github.com/google-antigravity/antigravity-cli/issues/1444)**（已关闭）：缺少 SSE Keep-Alive 心跳导致长流在 ~60s 后报 `unexpected EOF` / `Model unavailable`。

### 2. 首 Token 延迟（TTFT）随上下文暴涨
- **[#1762](https://github.com/google-antigravity/antigravity-cli/issues/1762)**：7 天 502 次 `Gemini 4 Neon (High)` 调用实测，TTFT 随总上下文陡增（与缓存是否命中、思考 token 数量无关），而同账号 `Gemini 4 Hydrogen` 基本不受影响：
  - `< 50k`：p50 `5.5s`
  - `200k–300k`：p50 `25.2s`
  - `> 300k`：p50 `53.4s`（p90 `103.8s`，最长 `231s`）

### 3. Prompt Cache 频繁归零与额度空耗
- **[#1665](https://github.com/google-antigravity/antigravity-cli/issues/1665)**：两轮请求间隔超过 `25–30s` 后，Neon 与 Carbon 的隐式缓存命中率骤降至 `0`，周额度消耗约为正常的 ~2.5 倍。
- **[#1720](https://github.com/google-antigravity/antigravity-cli/issues/1720)**：`11-21` 后端缓存补丁后，`Neon (High/Medium)` 有所改善，但 `Neon (XHigh)` 命中率仍只有约 `44.6%`（在 `0%` 与 `99%` 之间交替抖动）；评论区亦有用户反馈 `Neon (High)` 仍存在同样问题。

### 4. `XHIGH` 400 中断、虚报完成与会话卡死
- **[#1616](https://github.com/google-antigravity/antigravity-cli/issues/1616)**（Ultra 用户汇总）：`XHIGH` 400 中断、任务未完成却虚报 `done`、输出一句后直接结束回合。
- **[#1609](https://github.com/google-antigravity/antigravity-cli/issues/1609)** / **[#1642](https://github.com/google-antigravity/antigravity-cli/issues/1642)**：报错 `400 Thinking level is unsupported: THINKING_LEVEL_XHIGH`（部分用户在升级至 `1.5.13+` 后不再复现）。
- **[#1724](https://github.com/google-antigravity/antigravity-cli/issues/1724)**：Gemini Enterprise 账号下 `Neon (XHigh)` 每轮均报 400。
- **[#1546](https://github.com/google-antigravity/antigravity-cli/issues/1546)** / **[#1574](https://github.com/google-antigravity/antigravity-cli/issues/1574)**：大上下文预总结卡满 5 分钟流空闲超时，导致会话无法恢复。

---

## 三、常见疑问与模型定位

### 1. 是因为 Pro 账号才不稳定吗？
**不是。** 上述问题在 **Google AI Pro**（#1762）、**Google AI Ultra**（#1616）与 **Gemini Enterprise**（#1724）中均有大量报告，属于 `agy` 客户端与 Neon 后端的通用问题。Pro 账号主要吃亏在额度池更小，反复断流重试与缓存失效会更快耗尽 5h / 周额度。

### 2. Gemini 4 三款模型能力排序
1. **Gemini 4 Neon**（最强，主力 Pro 档，支持到 `XHigh`，但当前长思考断流和缓存抖动最明显）
2. **Gemini 4 Argon**（中等，早期中间档，最高 `High`，正逐步被 Neon 替代）
3. **Gemini 4 Hydrogen**（轻量，Flash 档，响应快、TTFT 低、不易在思考阶段断流）

---

## 四、当前可行的规避方式

1. **主力复杂任务**：优先使用 **`Claude Opus 5.5`**，长思考稳定性与完成率远高于当前版本的 Neon。
2. **日常开发与快速执行**：使用 **`Gemini 4 Hydrogen (Medium/High)`**，避免在思考阶段触发长时间无数据流导致的 EOF。
3. **必须使用 Neon 时**：
   - 尽量避开 `XHigh` 和 `High`，改用 **`Medium`** 缩短单次思考时长；
   - 控制单会话上下文在 `150k–200k` 以内，及时 `/compact` 或开新会话；
   - 遇到卡住超过 2 分钟无输出时及时按 `Esc` 中断，避免后台无限重试消耗额度。
4. **网络侧建议**：代理软件保持**规则模式**并优先选用稳定 TCP 节点（如 `vless` / `anytls`）与干净出口 IP，避免长期使用机房 IP 触发额外风控。

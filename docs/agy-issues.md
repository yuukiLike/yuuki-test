# Antigravity CLI (`agy`) 长流断流排查记录

> 更新日期：`2026-11-25`（`Asia/Shanghai`）｜环境：`agy`（macOS）+ Mihomo/Clash TUN 代理

## 一、核心结论

- **现象**：使用 `Gemini 4 Neon (Medium/High)` 执行长思考任务（如 `/pelican-test`）时，单轮超过 ~60s 频繁报 `stream reading error: unexpected EOF` 并整轮重试。
- **根因**：`agy` 客户端禁用了 TCP KeepAlive（[#1255](https://github.com/google-antigravity/antigravity-cli/issues/1255)）且断流无法续传（[#1563](https://github.com/google-antigravity/antigravity-cli/issues/1563)）；在 **iPhone 无线个人热点**下，蜂窝 NAT 会在 ~60–90s 主动掐断静默长连接。
- **已排除**：与 Warp 终端、Pro 账号等级无关；单独屏蔽 QUIC 或更换代理节点无法解决无线热点 NAT 断流。

## 二、控制变量实测（2026-11-25）

| 网络环境 | 模型 | 单轮耗时 | 结果 |
| :--- | :--- | :--- | :--- |
| iPhone 无线热点 (`172.20.10.2`) | `Neon Medium / High` | `57s–120s` | 连续 12 次 `unexpected EOF`（换节点/屏蔽 QUIC/关自动断连均复现） |
| iPhone 无线热点 (`172.20.10.2`) | `Neon Low` / `Hydrogen` / `Opus` | `< 50s`（`Neon Low` 46s / `10.4 KB`） | **一次成功** |
| 切换常规网络 (`192.168.1.7`) | `Neon Medium` | **88s**（`11,988` tokens / `18.4 KB`） | **一次成功**（同代理规则） |

## 三、关联上游 Issues

- **[#1255](https://github.com/google-antigravity/antigravity-cli/issues/1255)**：客户端禁用 HTTP/2 与 TCP KeepAlive，长流无输出超过 ~60s 易被中间链路掐断报 EOF。
- **[#1563](https://github.com/google-antigravity/antigravity-cli/issues/1563)**：遭遇 EOF 无法断点续传，只能整轮重新思考。
- **[#1762](https://github.com/google-antigravity/antigravity-cli/issues/1762)** / **[#1665](https://github.com/google-antigravity/antigravity-cli/issues/1665)**：大上下文下 Neon 首字延迟上升且缓存易失效，建议单会话超过 `150k` 时及时 `/compact`。

## 四、推荐配置与规避方式

1. **网络与热点**：
   - 跑长思考任务优先用稳定 Wi-Fi；必须用 iPhone 热点时改用 **USB 数据线共享（`iPhone USB`）**。
   - iPhone 与 Mac 均关闭「低数据模式」，iPhone 关闭「低电量模式」。
2. **代理客户端（Mihomo / Clash）**：
   - 覆写规则启用 **「屏蔽 QUIC」**（`AND,((NETWORK,UDP),(DST-PORT,443)),REJECT`），强制走 TCP 443。
   - **关闭「自动关闭连接」**，并为 Google / AI 流量手动固定稳定节点。
   - 将 **「TCP 保持活动间隔」** 调至 `15 秒`，保持 **IPv6 关闭**。
3. **模型档位**：
   - 弱网/无线热点下切至 `Neon (Low)`、`Hydrogen` 或 `Opus`，将单轮控制在 60s 内；稳定网络下可正常使用 `Neon (Medium/High)`。

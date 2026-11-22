---
name: pelican-test
description: 鹈鹕骑自行车 SVG 基准测试。支持 `/pelican-test`、`/pelican-test static`、`/pelican-test anim`（或 `$pelican-test`）。先盲测生成 SVG，再读取 README 归档。
---

# `/pelican-test [static|anim]`

## 参数解析
- `/pelican-test` 或 `/pelican-test static`：静态 SVG → `samples/pelican-bicycle-<YYYY-MM-DD>-<model-slug>.svg`
- `/pelican-test anim`（或 `animation`）：循环动画 SVG → `samples/pelican-bicycle-animation-<YYYY-MM-DD>-<model-slug>.svg`

## 步骤 1：盲测生成（落盘前禁读仓库）
1. **严格盲测**：在 SVG 写入磁盘前，**严禁**读取 `samples/`、`README.md` 或 Git 历史。
2. **纯 SVG**：绘制「鹈鹕骑自行车」。仅用 SVG 矢量元素，禁嵌位图或外部脚本；`anim` 模式仅用内联 CSS `@keyframes` 或 SMIL `<animate>` / `<animateTransform>`，确保 `<img>` 引用时可自动播放。
3. **保存文件**：日期取 `Asia/Shanghai`（`YYYY-MM-DD`），`<model-slug>` 取小写模型简称（同日重测加 `-v2`）。

## 步骤 2：README 归档
SVG 保存后，读取 `README.md`，在 `## 鹈鹕骑自行车（SVG）` 表尾追加一行：

```markdown
| <YYYY-MM-DD> | <a href="./samples/<文件名>.svg"><img src="./samples/<文件名>.svg" width="220" alt="<中文短标题>"></a> | <调用入口> | `<真实模型名>` | `<推理强度>` | <用户原始输入> |
```

- **模型与推理强度（强制自检，严禁猜测或照抄旧记录）**：
  - 必须上报**执行本轮绘制的当前会话真实模型 ID** 与**当前推理强度**（优先从系统上下文、环境变量或当前会话日志中的 `model`、`effort` / `thinkingLevel` 核对）。
  - **严禁**照抄 `README.md` 历史行中的模型名或推理档位，也**严禁**自行编造。若上下文未暴露推理强度则填 `-`；若连自身精确模型名也无法从环境确认，须在归档前向用户确认。
- **调用入口**：
  - Pi 调 API：统一填 `<Provider> API（Pi）`（如 `OpenAI API（Pi）`、`Anthropic API（Pi）`、`Gemini API（Pi）` 等）。
  - 独立客户端：直填客户端名（如 `Codex`、`Antigravity`、`Claude Code`）。
- **原始提示词**：原样填入用户输入的命令（如 `/pelican-test static`）。

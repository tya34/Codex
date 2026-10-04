# Codex 当前配置盘点

更新时间：2026-10-04（Asia/Shanghai）。数据来自当前设备配置文件、技能目录计数和应用偏好的只读检查。

## 文件与规则

| 项目 | 当前状态 |
| --- | --- |
| 全局目录 | `C:\Users\tiany\.codex` |
| `AGENTS.md` | 空文件 |
| `AGENTS.override.md` | 完整 Cleanup Audit 规则，与覆盖前仓库规则一致 |
| `config.toml` | 已同步本机当前完整文本 |
| `developer_instructions` | 未显式配置 |
| 本机 `GLOBAL_CONFIG.md` | 不存在；本文为仓库说明 |
| 用户级 `rules/`、`requirements.toml`、`hooks.json` | 未发现 |

清理要求仍保存在全局覆盖指令中，当前会话也已加载该要求。配置没有重复写入中文清理规则，也没有旧仓库的 GitHub 插件优先指令。

## 显式配置

| 配置 | 值 |
| --- | --- |
| 默认模型 | `gpt-6-astra` |
| 推理强度 | `medium` |
| Windows 沙箱后端 | `elevated` |
| 后续消息 | `followUpQueueMode = "queue"` |
| Node REPL 启动超时 | 120 秒 |
| 浏览器后端 | `chrome,iab,mcpapps` |
| TinySky | `BROWSER_USE_TINYSKY_ENABLED = "1"` |
| 配置记录的应用版本 | `26.930.31730` |
| 本地插件市场 | `openai-primary-runtime`、`openai-bundled` |

以下字段未显式设置：`service_tier`、`personality`、`approval_policy`、`features.js_repl`。没有 `projects` 项目信任条目。未设置不等于关闭，也不代表没有客户端或托管配置影响。

同步期间用户切换了权限；最终配置显式保存 `sandbox_mode = "danger-full-access"`，当前会话审批策略为 `never`（未写入此配置文件）。配置还保存了默认打开方式 `systemDefault` 和 GitHub `update_file` 工具的 `approval_mode = "approve"`。Windows 的 `elevated` 后端字段不能单独用来推断完整访问权限。

## 插件与技能

`config.toml` 显式启用以下 9 个插件：

- Documents
- PDF
- Spreadsheets
- Presentations
- Template Creator
- Browser
- Unified Computer Use
- Computer Use
- Visualize

GitHub、Zotero、Sites 没有独立开关条目，但本次会话提供了这些能力；GitHub 连接器已实际完成仓库读取与此次同步。不能仅凭配置条目判断安装或授权状态。

用户技能目录排除 `.system` 后为 173 个。仓库不包含技能文件，因此这个计数不证明技能内容或版本相同。

## 应用偏好

本机应用状态中读取到：

- `composer-auto-context-enabled = false`
- `skip-full-access-confirm = true`

这些应用状态没有随配置文件上传。`skip-full-access-confirm` 不是命令审批策略。

## 相对旧快照的变化

- 使用当前设备用户名与运行时路径替换旧设备路径。
- 写入当前模型 `gpt-6-astra` 与 `medium` 推理强度，取代旧仓库不固定模型的约定。
- 移除本机未配置的 `developer_instructions`、`service_tier`、历史项目信任记录、`features.js_repl` 与市场更新时间字段。
- 插件显式开关与本机一致：不再保留旧快照中的 GitHub、Zotero、Sites 条目，加入 Computer Use 与 Unified Computer Use。
- 更新 Node REPL 的环境变量、可信服务、浏览器后端和运行时标识。
- 全局清理规则保持不变。

旧说明中的 21 个信任项目、12 个显式插件属于历史盘点；权限状态以本次重新核验值为准。本次覆盖文件内容，保留 Git 提交历史。

## 维护原则

跨设备迁移时选择性合并可迁移规则与偏好，保留目标设备自身路径和运行时。更新后核对配置文件及新会话实际加载状态。临时备份与中间文件按 `AGENTS.override.md` 的逐项普通删除规则处理。

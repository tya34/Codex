# Work Log

## 2026-05-29

### 本次目标

把 Codex 的全局清理偏好和“把这次工作整理上传”快捷命令固化到本地，并同步到 GitHub，方便之后在其他电脑继续使用。

### 已完成

- 确认 GitHub connector 已授权，当前登录账号为 `tya34`。
- 找到用户新建的仓库 `tya34/Codex-`，这是原计划中文仓库名 `codex配置` 被 GitHub 处理后的仓库名。
- 将本地 Codex 全局配置写入并上传为 `config.toml`。
- 将输出清理偏好上传为 `memories/download-cleanup.md`。
- 新增“把这次工作整理上传”快捷命令偏好，并上传为 `memories/work-upload-command.md`。
- 更新 `README.md`，说明仓库用途、当前规则、快捷命令和文件结构。
- 新增 `WORKLOG.md` 和 `NEXT_STEPS.md`，作为跨电脑继续工作的交接记录。

### 当前规则摘要

以后 Codex 生成、下载、转换、导出、解压、构建或整理任何文件时，任务结束后只保留用户明确需要的最终输出结果。任务过程中产生的临时 ZIP、压缩包、解压中间目录、缓存文件、草稿文件、预览文件、日志文件、构建中间产物等辅助文件应清理掉。清理前确认不会删除用户原本已有的文件或仍有用的工作文件。

### 快捷命令摘要

用户说“把这次工作整理上传”时，Codex 应整理最终文件、关键对话结论、操作记录、仓库链接、当前进度和后续待办；优先生成或更新 `README.md`、`WORKLOG.md`、`NEXT_STEPS.md`；只上传最终输出和整理后的记录，不上传临时文件、缓存、草稿、日志或中间产物。

### 仓库链接

- 配置仓库：https://github.com/tya34/Codex-
- 配置文件：https://github.com/tya34/Codex-/blob/main/config.toml
- 输出清理偏好：https://github.com/tya34/Codex-/blob/main/memories/download-cleanup.md
- 整理上传快捷命令：https://github.com/tya34/Codex-/blob/main/memories/work-upload-command.md

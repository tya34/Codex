# Codex 全局配置同步

这个仓库用于保存 Codex 的可迁移全局指令。目标是在其他电脑上打开 Codex 后，可以直接从 GitHub 读取这些文件，并把同样的行为规则写入那台设备的本地 Codex 配置。

## 文件作用

- `config.toml`：当前这台电脑的 Codex 全局配置快照。最重要的是其中的 `developer_instructions`，它规定了文件生成、下载、转换、导出、解压、构建或整理任务完成后，只保留用户明确需要的最终结果，并清理临时 ZIP、缓存、草稿、预览、日志和中间产物等辅助文件。
- `memories/download-cleanup.md`：同一条输出清理偏好的记忆文件。它让 Codex 在跨会话时也能记住：生成或下载文件后，清理中间辅助文件，只保留最终输出。

## 本次更新

2026-06-03 更新：`config.toml` 的 `developer_instructions` 新增了同步覆盖规则。

当用户要求上传或下载文件并需要同步到本地或远端时，如果目标位置已经存在同名文件，Codex 应直接用同步来源覆盖目标同名文件，不再为了覆盖操作额外创建本地备份文件或备份文件夹。除非用户明确要求保留历史版本，否则只保留覆盖后的最终同步结果，并清理下载 ZIP、解压目录等辅助文件。

## 在其他 Windows 设备同步

推荐做法是只同步 `developer_instructions` 和记忆文件，因为完整的 `config.toml` 里包含本机路径、插件缓存路径和本机 MCP 路径，其他电脑可能不同。

在另一台 Windows 电脑上打开 PowerShell，运行：

```powershell
$repo = "https://raw.githubusercontent.com/tya34/Codex-/main"
$codexHome = Join-Path $env:USERPROFILE ".codex"
$memories = Join-Path $codexHome "memories"
$configPath = Join-Path $codexHome "config.toml"
$remoteConfig = Join-Path $codexHome "config.github.toml"
$backup = Join-Path $codexHome ("config.toml.bak-" + (Get-Date -Format "yyyyMMdd-HHmmss"))

New-Item -ItemType Directory -Force -Path $codexHome, $memories | Out-Null
Invoke-WebRequest "$repo/config.toml" -OutFile $remoteConfig
Invoke-WebRequest "$repo/memories/download-cleanup.md" -OutFile (Join-Path $memories "download-cleanup.md")

if (Test-Path $configPath) {
    Copy-Item $configPath $backup
    $local = Get-Content $configPath -Raw -Encoding UTF8
} else {
    $local = ""
}

$remote = Get-Content $remoteConfig -Raw -Encoding UTF8
$instruction = [regex]::Match($remote, '(?s)developer_instructions\s*=\s*""".*?"""').Value
if (-not $instruction) { throw "未能从 GitHub config.toml 中读取 developer_instructions" }

if ($local -match '(?s)developer_instructions\s*=\s*""".*?"""') {
    $local = [regex]::Replace($local, '(?s)developer_instructions\s*=\s*""".*?"""', $instruction, 1)
} else {
    $local = $instruction + "`n`n" + $local
}

Set-Content -Path $configPath -Value $local -Encoding UTF8
Remove-Item $remoteConfig -Force
Write-Host "已同步 Codex 全局指令和 download-cleanup 记忆文件。请重启 Codex。"
```

这段命令会先备份其他设备原有的 `config.toml`，然后只把 GitHub 里的 `developer_instructions` 合并进去，避免覆盖那台电脑自己的路径、插件和本地运行配置。

## 完整覆盖方式

只有在另一台电脑的 Codex 安装路径、用户名、插件缓存路径都和这台电脑一致时，才建议完整覆盖：

```powershell
$repo = "https://raw.githubusercontent.com/tya34/Codex-/main"
$codexHome = Join-Path $env:USERPROFILE ".codex"
New-Item -ItemType Directory -Force -Path (Join-Path $codexHome "memories") | Out-Null
Copy-Item (Join-Path $codexHome "config.toml") (Join-Path $codexHome ("config.toml.bak-" + (Get-Date -Format "yyyyMMdd-HHmmss"))) -ErrorAction SilentlyContinue
Invoke-WebRequest "$repo/config.toml" -OutFile (Join-Path $codexHome "config.toml")
Invoke-WebRequest "$repo/memories/download-cleanup.md" -OutFile (Join-Path $codexHome "memories\download-cleanup.md")
```

同步完成后重启 Codex，新设备就会读取这些全局指令。

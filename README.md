# Codex 全局配置同步

这个仓库保存当前 Codex 全局配置快照，主要用于在其他电脑上恢复或更新相同的行为规则。

## 当前文件

- `config.toml`：当前电脑的 Codex 配置快照。里面包含清理检查硬规则、模型设置、沙盒设置、已启用插件和本机项目 trust 路径。

仓库现在只保留最终配置和说明文档；旧的临时记忆文件、缓存说明或中间辅助文件不再保留。

## 重要提醒

`config.toml` 里可能包含本机路径，例如用户名、插件缓存目录、项目路径和 marketplace 路径。不同电脑上的这些路径通常不一样。

因此，在其他电脑上更新配置时，推荐使用“安全合并”：只同步可迁移的全局指令和通用偏好，不直接覆盖整份本地配置。这样可以避免破坏那台电脑自己的沙盒、插件、本地路径和 MCP 配置。

## 推荐：安全合并配置

在另一台 Windows 电脑上打开 PowerShell，运行：

```powershell
$repo = "https://raw.githubusercontent.com/tya34/Codex/main"
$codexHome = Join-Path $env:USERPROFILE ".codex"
$configPath = Join-Path $codexHome "config.toml"
$remoteConfig = Join-Path $env:TEMP ("codex-config-" + [guid]::NewGuid().ToString() + ".toml")
$backup = Join-Path $codexHome ("config.toml.bak-" + (Get-Date -Format "yyyyMMdd-HHmmss"))

New-Item -ItemType Directory -Force -Path $codexHome | Out-Null
Invoke-WebRequest "$repo/config.toml" -OutFile $remoteConfig

if (Test-Path $configPath) {
    Copy-Item $configPath $backup
    $local = Get-Content $configPath -Raw -Encoding UTF8
} else {
    $local = ""
}

$remote = Get-Content $remoteConfig -Raw -Encoding UTF8
$instructionPattern = '(?s)developer_instructions\s*=\s*(''\'\'\'.*?\'\'\''' + '|""".*?""")'
$instruction = [regex]::Match($remote, $instructionPattern).Value
if (-not $instruction) { throw "未能从远端 config.toml 中读取 developer_instructions" }

if ($local -match $instructionPattern) {
    $local = [regex]::Replace($local, $instructionPattern, [System.Text.RegularExpressions.MatchEvaluator]{ param($m) $instruction }, 1)
} else {
    $local = $instruction + "`n`n" + $local
}

foreach ($line in @(
    'model = "gpt-5.5"',
    'model_reasoning_effort = "high"'
)) {
    $key = $line.Split('=')[0].Trim()
    if ($local -match "(?m)^$([regex]::Escape($key))\s*=") {
        $local = [regex]::Replace($local, "(?m)^$([regex]::Escape($key))\s*=.*$", $line, 1)
    } else {
        $local = $line + "`n" + $local
    }
}

if ($local -notmatch '(?m)^\[windows\]') {
    $local += "`n[windows]`nsandbox = \"elevated\"`n"
} elseif ($local -match '(?ms)^\[windows\].*?(?=^\[|\z)') {
    $local = [regex]::Replace($local, '(?ms)^\[windows\].*?(?=^\[|\z)', "[windows]`nsandbox = \"elevated\"`n", 1)
}

foreach ($plugin in @('github@openai-curated', 'browser@openai-bundled')) {
    $section = "[plugins.\"$plugin\"]"
    if ($local -notmatch [regex]::Escape($section)) {
        $local += "`n$section`nenabled = true`n"
    }
}

Set-Content -Path $configPath -Value $local -Encoding UTF8
Remove-Item $remoteConfig -Force
Write-Host "已安全合并 Codex 配置。备份文件：$backup"
Write-Host "请重启 Codex。"
```

这段脚本会：

- 备份原本的本地 `config.toml`。
- 从 GitHub 下载当前仓库里的 `config.toml`。
- 只合并 `developer_instructions`、模型设置、Windows 沙盒设置，以及 GitHub/Browser 插件开关。
- 保留另一台电脑本来已有的本地路径、插件缓存、MCP 配置和项目配置。

## 可选：完整覆盖配置

只有在你明确希望让另一台电脑完全使用本仓库的配置快照时，才使用完整覆盖：

```powershell
$repo = "https://raw.githubusercontent.com/tya34/Codex/main"
$codexHome = Join-Path $env:USERPROFILE ".codex"
$configPath = Join-Path $codexHome "config.toml"
$backup = Join-Path $codexHome ("config.toml.bak-" + (Get-Date -Format "yyyyMMdd-HHmmss"))

New-Item -ItemType Directory -Force -Path $codexHome | Out-Null
if (Test-Path $configPath) { Copy-Item $configPath $backup }
Invoke-WebRequest "$repo/config.toml" -OutFile $configPath
Write-Host "已完整覆盖 Codex 配置。备份文件：$backup"
Write-Host "请重启 Codex。"
```

完整覆盖会把本机路径也一起写入目标电脑，可能需要你之后手动调整项目路径、插件路径或 marketplace 路径。

## 当前清理规则摘要

当前配置要求：凡是任务涉及生成、下载、安装、解压、转换、导出、构建、运行脚本、日志、缓存或中间产物，Codex 在 final 前必须进行 Cleanup Audit。

Cleanup Audit 不能只检查某个固定目录，而要检查本次任务相关的所有位置，包括工作区临时目录、输出目录、下载/解压/构建/日志/缓存目录、工具实际使用过的系统临时目录，以及本轮任务创建或使用过的其它临时文件。

# Codex 全局配置同步

这个仓库保存我的 Codex 全局配置快照与可迁移行为规则，用于在其它设备上恢复一致的 Codex 工作方式，并记录 GitHub 插件连接器优先的远端操作策略。

## 当前文件

- `AGENTS.override.md`：全局强制指导文件。当前用于写入 Cleanup Audit 硬规则，优先级高于普通 `AGENTS.md`。
- `config.toml`：Codex 当前配置快照。包含 `developer_instructions`、沙盒设置、marketplace、插件开关、MCP、桌面行为和本机项目 trust 路径。

`config.toml` 里可能包含本机路径，例如用户名、插件缓存目录、marketplace 路径、MCP 路径和项目 trust 路径。不同电脑上这些路径通常不一样，所以跨设备同步时优先使用 `AGENTS.override.md`，不要盲目整份覆盖 `config.toml`。

## 本次更新

本次更新删除了 `config.toml` 中固定的 `model` 和 `model_reasoning_effort` 设置。模型与推理强度改为由用户在 Codex 界面中手动选择，仓库同步不再覆盖这两个选择；其余 marketplace、插件、MCP 和桌面配置保持记录。

GitHub 操作规则已调整为“插件连接器优先”：

- GitHub 仓库读取、状态核验、文件操作、Issue、Pull Request 和其他远端操作，默认先使用已安装的 Codex GitHub 插件连接器。
- 只有连接器不可用、授权失败、能力不足，或任务明确需要本地工作区时，才使用本地 Git、GitHub Desktop 或 `gh`。
- 多文件修改、构建和测试可以在本地完成；远端读取、核验和 API 操作仍优先使用 GitHub 插件。
- 用户明确指定工具时，以用户指定的方式为准。

Cleanup Audit 硬规则保持不变：凡是任务涉及生成、下载、安装、构建、日志、缓存或中间产物，Codex 在发送 final 前必须检查并清理当前任务产生且不再需要的辅助文件，并在 final 中包含一行以 `清理检查：` 开头的说明。

安全说明：仓库是公开的。上传前已检查当前配置，未发现 token、密码、私钥或带认证信息的 URL；但 `config.toml` 仍包含本机路径和运行时标识，跨设备使用时应优先采用下方的安全合并方式。

## 推荐：更新强制指令层

在其它 Windows 设备上打开 PowerShell，运行：

```powershell
$repo = "https://raw.githubusercontent.com/tya34/Codex/main"
$codexHome = Join-Path $env:USERPROFILE ".codex"
$overridePath = Join-Path $codexHome "AGENTS.override.md"
$backup = Join-Path $codexHome ("AGENTS.override.md.bak-" + (Get-Date -Format "yyyyMMdd-HHmmss"))

New-Item -ItemType Directory -Force -Path $codexHome | Out-Null
if (Test-Path $overridePath) {
    Copy-Item $overridePath $backup
}
Invoke-WebRequest "$repo/AGENTS.override.md" -OutFile $overridePath

Write-Host "已更新 AGENTS.override.md：$overridePath"
if (Test-Path $backup) { Write-Host "原文件备份：$backup" }
Write-Host "请完全重启 Codex 桌面端，并新建线程验证规则是否生效。"
```

重启 Codex 后，可以在新线程里测试：

```text
请复述当前生效的全局 Cleanup Audit 要求
```

如果模型能说出 final 前必须做 Cleanup Audit，并且 final 要包含以 `清理检查：` 开头的一行，说明强制指导文件已经被读取。

## 可选：安全合并 config.toml

如果还想同步 `config.toml` 里的通用配置，请使用安全合并，只同步可迁移字段，避免覆盖另一台电脑自己的本机路径。

```powershell
$repo = "https://raw.githubusercontent.com/tya34/Codex/main"
$codexHome = Join-Path $env:USERPROFILE ".codex"
$configPath = Join-Path $codexHome "config.toml"
$remoteConfig = Join-Path $env:TEMP ("codex-config-" + [guid]::NewGuid().ToString() + ".toml")
$backup = Join-Path $codexHome ("config.toml.bak-" + (Get-Date -Format "yyyyMMdd-HHmmss"))

function Get-DeveloperInstructionsBlock {
    param([string]$Text)

    $literalStart = "developer_instructions = '''"
    $tripleStart = 'developer_instructions = """'

    $start = $Text.IndexOf($literalStart)
    if ($start -ge 0) {
        $bodyStart = $start + $literalStart.Length
        $end = $Text.IndexOf("'''", $bodyStart)
        if ($end -lt 0) { throw "developer_instructions literal string is not closed" }
        return $Text.Substring($start, $end + 3 - $start)
    }

    $start = $Text.IndexOf($tripleStart)
    if ($start -ge 0) {
        $bodyStart = $start + $tripleStart.Length
        $end = $Text.IndexOf('"""', $bodyStart)
        if ($end -lt 0) { throw "developer_instructions triple string is not closed" }
        return $Text.Substring($start, $end + 3 - $start)
    }

    return $null
}

function Set-DeveloperInstructionsBlock {
    param([string]$Text, [string]$Block)
    $old = Get-DeveloperInstructionsBlock -Text $Text
    if ($old) { return $Text.Replace($old, $Block) }
    return $Block + "`n`n" + $Text
}

function Ensure-PluginEnabled {
    param([string]$Text, [string]$PluginName)
    $section = '[plugins."' + $PluginName + '"]'
    if ($Text.Contains($section)) { return $Text }
    return $Text.TrimEnd() + "`n`n$section`nenabled = true`n"
}

New-Item -ItemType Directory -Force -Path $codexHome | Out-Null
Invoke-WebRequest "$repo/config.toml" -OutFile $remoteConfig

if (Test-Path $configPath) {
    Copy-Item $configPath $backup
    $local = Get-Content $configPath -Raw -Encoding UTF8
} else {
    $local = ""
}

$remote = Get-Content $remoteConfig -Raw -Encoding UTF8
$instruction = Get-DeveloperInstructionsBlock -Text $remote
if (-not $instruction) { throw "未能从远端 config.toml 中读取 developer_instructions" }

$local = Set-DeveloperInstructionsBlock -Text $local -Block $instruction
$local = Ensure-PluginEnabled -Text $local -PluginName 'github@openai-curated'
$local = Ensure-PluginEnabled -Text $local -PluginName 'zotero@openai-curated'
$local = Ensure-PluginEnabled -Text $local -PluginName 'browser@openai-bundled'

Set-Content -Path $configPath -Value $local -Encoding UTF8
Remove-Item $remoteConfig -Force

Write-Host "已安全合并 Codex 配置。备份文件：$backup"
Write-Host "请完全重启 Codex。"
```

这段脚本会保留目标电脑原有的模型选择、推理强度、本机路径、MCP 配置、marketplace 路径和项目 trust 配置，只合并通用规则与常用插件开关。

## 可选：完整覆盖 config.toml

只有在明确希望目标电脑完全使用本仓库配置快照时，才使用完整覆盖：

```powershell
$repo = "https://raw.githubusercontent.com/tya34/Codex/main"
$codexHome = Join-Path $env:USERPROFILE ".codex"
$configPath = Join-Path $codexHome "config.toml"
$backup = Join-Path $codexHome ("config.toml.bak-" + (Get-Date -Format "yyyyMMdd-HHmmss"))

New-Item -ItemType Directory -Force -Path $codexHome | Out-Null
if (Test-Path $configPath) { Copy-Item $configPath $backup }
Invoke-WebRequest "$repo/config.toml" -OutFile $configPath

Write-Host "已完整覆盖 Codex 配置。备份文件：$backup"
Write-Host "请完全重启 Codex。"
```

完整覆盖可能把本仓库里的本机路径写入目标电脑，之后可能需要手动调整插件路径、marketplace 路径、MCP 路径和项目 trust 路径。

## 插件提醒

同步配置后，请确认至少安装并启用这些插件：

- GitHub：`github@openai-curated`
- Zotero：`zotero@openai-curated`
- Documents：`documents@openai-primary-runtime`
- PDF：`pdf@openai-primary-runtime`
- Spreadsheets：`spreadsheets@openai-primary-runtime`
- Presentations：`presentations@openai-primary-runtime`
- Template Creator：`template-creator@openai-primary-runtime`
- Sites：`sites@openai-bundled`
- Visualize：`visualize@openai-bundled`
- Browser：`browser@openai-bundled`

只在 `config.toml` 里写入 `enabled = true` 不等于插件已经安装或授权完成。新设备上仍需要在 Codex 里安装对应插件，并完成 GitHub 授权或 Zotero 本地应用配置。

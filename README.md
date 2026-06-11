# Codex 全局配置同步

这个仓库保存当前 Codex 全局配置快照，用于在其他电脑上恢复或更新相同的行为规则。

## 当前文件

- `config.toml`：当前电脑的 Codex 配置快照。里面包含清理检查硬规则、GitHub 操作方式选择规则、模型设置、沙盒设置、已启用插件和本机项目 trust 路径。

仓库现在只保留最终配置和说明文档；旧的临时记忆文件、缓存说明或中间辅助文件不再保留。

## 本次配置更新

这次更新加入了两类内容：

- GitHub 操作方式选择规则：小型文档或配置更新优先使用 Codex GitHub 连接器；多文件代码修改、需要测试、需要完整 diff 或复杂冲突处理时优先使用 GitHub Desktop 或本地 Git；如果用户想自己掌控提交前检查，优先建议 GitHub Desktop。
- 临时备份清理规则：如果任务为了安全创建了临时备份、回滚文件或导出备份，在操作验证成功后也必须删除；只有用户明确要求保留，或该备份本身就是最终交付物时才保留。

另外，README 记录了如何把 GitHub Desktop 自带的 Git 加入 Windows 用户 PATH，方便在其他电脑上直接使用 `git` 命令。

## 重要提醒

`config.toml` 里可能包含本机路径，例如用户名、插件缓存目录、项目路径和 marketplace 路径。不同电脑上的这些路径通常不一样。

因此，在其他电脑上更新配置时，推荐使用“安全合并”：只同步可迁移的全局指令和通用偏好，不直接覆盖整份本地配置。这样可以避免破坏那台电脑自己的沙盒、插件、本地路径和 MCP 配置。

## 插件提醒

在其他电脑上同步配置后，请确认至少安装并启用这两个插件：

- GitHub：`github@openai-curated`
- Zotero：`zotero@openai-curated`

当前配置还会启用 Browser 插件：`browser@openai-bundled`。如果目标电脑没有安装 GitHub 或 Zotero 插件，只写入 `enabled = true` 还不够，需要先在 Codex 里安装对应插件并完成授权或本地应用配置。

## GitHub 操作方式选择

当前全局配置采用下面的默认判断：

- 小型文档或配置更新：优先使用 Codex GitHub 连接器。它不需要本地 clone，也不依赖本机 `git` / `gh` 是否可用，适合快速同步确定改动。
- 多文件代码修改、需要本地测试、需要完整 diff、需要构建验证或复杂冲突处理：优先使用 GitHub Desktop 或本地 Git 工作流。
- 如果用户想自己掌控提交前检查、查看图形化 diff 或手动确认提交内容：优先建议使用 GitHub Desktop。
- 如果用户只想让 Codex 快速同步一个明确的小改动：优先使用 GitHub 连接器；如果需要把多个文件合成单个提交，需要明确说明连接器默认可能产生逐文件提交，并按需要改用本地 Git 或更完整的 GitHub API 提交流程。

## 启用 GitHub Desktop 内置 Git

GitHub Desktop 自带 Git，但它通常不会自动把 `git.exe` 加进普通 PowerShell 的 PATH。可以在另一台 Windows 电脑上运行下面的 PowerShell，把 GitHub Desktop 内置 Git 加入“用户 PATH”：

```powershell
$githubDesktop = Join-Path $env:LOCALAPPDATA "GitHubDesktop"
if (-not (Test-Path $githubDesktop)) {
    throw "未找到 GitHub Desktop 目录：$githubDesktop"
}

$gitCmd = Get-ChildItem -Path $githubDesktop -Directory -Filter "app-*" |
    Sort-Object Name -Descending |
    ForEach-Object { Join-Path $_.FullName "resources\app\git\cmd" } |
    Where-Object { Test-Path (Join-Path $_ "git.exe") } |
    Select-Object -First 1

if (-not $gitCmd) {
    throw "未找到 GitHub Desktop 内置 git.exe。请确认 GitHub Desktop 已安装并至少启动过一次。"
}

$userPath = [Environment]::GetEnvironmentVariable("Path", "User")
$parts = @()
if ($userPath) {
    $parts = $userPath -split ";" | Where-Object { $_ -ne "" }
}

$exists = $parts | Where-Object { $_.TrimEnd("\") -ieq $gitCmd.TrimEnd("\") }
if (-not $exists) {
    [Environment]::SetEnvironmentVariable("Path", (($parts + $gitCmd) -join ";"), "User")
}

if (($env:Path -split ";" | Where-Object { $_.TrimEnd("\") -ieq $gitCmd.TrimEnd("\") }).Count -eq 0) {
    $env:Path = $env:Path.TrimEnd(";") + ";" + $gitCmd
}

try {
    Add-Type -Namespace Win32 -Name NativeMethods -MemberDefinition @'
[System.Runtime.InteropServices.DllImport("user32.dll", SetLastError=true, CharSet=System.Runtime.InteropServices.CharSet.Auto)]
public static extern System.IntPtr SendMessageTimeout(System.IntPtr hWnd, uint Msg, System.IntPtr wParam, string lParam, uint fuFlags, uint uTimeout, out System.IntPtr lpdwResult);
'@
    $result = [IntPtr]::Zero
    [void][Win32.NativeMethods]::SendMessageTimeout([IntPtr]0xffff, 0x001A, [IntPtr]::Zero, "Environment", 2, 5000, [ref]$result)
} catch {
    Write-Warning "已更新用户 PATH，但发送环境变量刷新通知失败。重开终端后仍会生效。"
}

git --version
Write-Host "已加入用户 PATH：$gitCmd"
Write-Host "请重开 PowerShell、GitHub Desktop 或 Codex，让新 PATH 被新进程继承。"
```

推荐加到用户 PATH，而不是机器级系统 PATH。用户 PATH 不需要管理员权限，也足够让当前用户的 PowerShell、Codex 和大多数开发工具找到 `git`。

## 推荐：安全合并配置

在另一台 Windows 电脑上打开 PowerShell，运行：

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
    if ($old) {
        return $Text.Replace($old, $Block)
    }
    return $Block + "`n`n" + $Text
}

function Set-TopLevelLine {
    param([string]$Text, [string]$Line)

    $key = $Line.Split('=')[0].Trim()
    $pattern = "(?m)^" + [regex]::Escape($key) + "\s*=.*$"
    if ($Text -match $pattern) {
        return [regex]::Replace($Text, $pattern, $Line, 1)
    }
    return $Line + "`n" + $Text
}

function Set-WindowsSandbox {
    param([string]$Text)

    $sectionPattern = '(?ms)^\[windows\].*?(?=^\[|\z)'
    $section = "[windows]`nsandbox = `"elevated`"`n"
    if ($Text -match $sectionPattern) {
        return [regex]::Replace($Text, $sectionPattern, $section, 1)
    }
    return $Text.TrimEnd() + "`n`n" + $section
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
$local = Set-TopLevelLine -Text $local -Line 'model = "gpt-5.5"'
$local = Set-TopLevelLine -Text $local -Line 'model_reasoning_effort = "high"'
$local = Set-WindowsSandbox -Text $local
$local = Ensure-PluginEnabled -Text $local -PluginName 'github@openai-curated'
$local = Ensure-PluginEnabled -Text $local -PluginName 'zotero@openai-curated'
$local = Ensure-PluginEnabled -Text $local -PluginName 'browser@openai-bundled'

Set-Content -Path $configPath -Value $local -Encoding UTF8
Remove-Item $remoteConfig -Force
Write-Host "已安全合并 Codex 配置。备份文件：$backup"
Write-Host "请重启 Codex，并确认 GitHub 与 Zotero 插件已安装且可用。"
```

这段脚本会：

- 备份原本的本地 `config.toml`。
- 从 GitHub 下载当前仓库里的 `config.toml`。
- 合并 `developer_instructions`、模型设置、Windows 沙盒设置，以及 GitHub/Zotero/Browser 插件开关。
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
Write-Host "请重启 Codex，并确认 GitHub 与 Zotero 插件已安装且可用。"
```

完整覆盖会把本机路径也一起写入目标电脑，可能需要你之后手动调整项目路径、插件路径或 marketplace 路径。

## 当前清理规则摘要

当前配置要求：凡是任务涉及生成、下载、安装、解压、转换、导出、构建、运行脚本、日志、缓存或中间产物，Codex 在 final 前必须进行 Cleanup Audit。

Cleanup Audit 不能只检查某个固定目录，而要检查本次任务相关的所有位置，包括工作区临时目录、输出目录、下载/解压/构建/日志/缓存目录、工具实际使用过的系统临时目录，以及本轮任务创建或使用过的其它临时文件。

如果本轮任务为了安全创建了临时备份、回滚文件或导出备份，在操作已验证成功后也必须删除；只有用户明确要求保留，或该备份本身就是用户要求的最终交付物时才可保留，并需要在 final 中说明。

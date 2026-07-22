# Windows Codex logs_2.sqlite 高频写入规避方案

> 状态：这是针对上游尚未修复问题的本地、可逆规避方案，不是 Codex 官方修复。
>
> 实测环境：Windows，Codex Desktop 26.715.9757.0，2026-07-22。

## 问题概述

Codex Desktop 的 app-server 会把大量 TRACE/DEBUG 等内部事件写入：

- %USERPROFILE%\.codex\logs_2.sqlite
- %USERPROFILE%\.codex\logs_2.sqlite-wal

数据库使用 SQLite WAL 模式。即使主数据库大小保持稳定，WAL 仍可能持续循环写入、checkpoint 和复用页面，因此“文件没有继续变大”不代表没有持续写盘。

相关上游问题：

- [openai/codex#17320：TRACE 日志绕过 RUST_LOG，导致 SQLite WAL 过量写入](https://github.com/openai/codex/issues/17320)
- [openai/codex#24275：Codex Desktop 正常使用时 logs_2.sqlite/WAL 快速增长](https://github.com/openai/codex/issues/24275)
- [openai/codex#20213：共享日志库、TRACE 和 SQLite 压力](https://github.com/openai/codex/issues/20213)

## 本机诊断结果

应用规避方案前的一次活跃采样：

- 20 秒新增 133 条日志。
- 其中 89 条为 TRACE，占新增记录约 67%。
- WAL 在 20 个一秒采样点中有 9 次修改时间变化。
- 现存日志中，TRACE 占约 64% 的行数、75.35% 的估算数据量。
- Codex app-server 进程在另一段 15 秒采样中产生 955 次逻辑写操作，约 3.49 MB。
- 数据库 PRAGMA quick_check 结果为 ok。

主要 TRACE 来源包括：

- codex_app_server::outgoing_message
- codex_api::sse::responses
- codex_mcp::connection_manager
- codex_core::session::turn

## 方案原理

给 logs 表创建 BEFORE INSERT 触发器：

~~~sql
CREATE TRIGGER IF NOT EXISTS block_log_inserts
BEFORE INSERT ON logs
BEGIN
  SELECT RAISE(IGNORE);
END;
~~~

RAISE(IGNORE) 会静默忽略插入，不向调用者返回 SQLite 错误。现有日志不会被删除，会话历史也不会受到影响。

此方案只阻止 logs_2.sqlite 的新日志行写入，不会修改：

- 会话和任务历史
- 工作区文件
- state_5.sqlite
- 登录状态
- Codex 配置
- Git 仓库或构建产物

## Windows 应用方法

要求系统可以运行 Python 3。无需单独安装 sqlite3 命令行工具。

在 PowerShell 中执行：

~~~powershell
@'
import os
import sqlite3

db = os.path.join(os.environ["USERPROFILE"], ".codex", "logs_2.sqlite")
trigger_name = "block_log_inserts"

con = sqlite3.connect(db, timeout=30)
con.execute("PRAGMA busy_timeout=30000")

try:
    con.execute("BEGIN IMMEDIATE")
    con.execute(f"""
        CREATE TRIGGER IF NOT EXISTS {trigger_name}
        BEFORE INSERT ON logs
        BEGIN
          SELECT RAISE(IGNORE);
        END
    """)
    con.commit()
except Exception:
    con.rollback()
    raise

row = con.execute(
    "SELECT name, tbl_name FROM sqlite_master "
    "WHERE type='trigger' AND name=?",
    (trigger_name,),
).fetchone()

print("trigger:", row)
print("quick_check:", con.execute("PRAGMA quick_check").fetchone()[0])
con.close()
'@ | python -
~~~

预期输出应包含：

~~~text
trigger: ('block_log_inserts', 'logs')
quick_check: ok
~~~

脚本是幂等的；重复执行不会创建多个同名触发器。

## 验证触发器

下面的自检会尝试写入一条测试日志。触发器正常时，插入会被静默忽略，行号和行数保持不变：

~~~powershell
@'
import os
import sqlite3

db = os.path.join(os.environ["USERPROFILE"], ".codex", "logs_2.sqlite")
con = sqlite3.connect(db, timeout=30)

before = con.execute("SELECT MAX(id), COUNT(*) FROM logs").fetchone()

con.execute(
    "INSERT INTO logs(ts, ts_nanos, level, target, estimated_bytes) "
    "VALUES (strftime('%s','now'), 0, 'TRACE', 'workaround.self_test', 1)"
)
changes = con.execute("SELECT changes()").fetchone()[0]
con.commit()

after = con.execute("SELECT MAX(id), COUNT(*) FROM logs").fetchone()
trigger = con.execute(
    "SELECT COUNT(*) FROM sqlite_master "
    "WHERE type='trigger' AND name='block_log_inserts'"
).fetchone()[0]
integrity = con.execute("PRAGMA quick_check").fetchone()[0]
con.close()

print("before:", before)
print("after:", after)
print("changes:", changes)
print("trigger:", trigger)
print("quick_check:", integrity)

assert before == after
assert changes == 0
assert trigger == 1
assert integrity == "ok"
'@ | python -
~~~

实际应用后的 15 秒活跃验证结果：

- MAX(id) 变化：0
- 总行数变化：0
- TRACE 行数变化：0
- WAL 大小变化：0
- WAL 修改时间变化：0
- Codex 工具调用继续正常工作

## 撤销方案

需要恢复本地诊断日志时，在 PowerShell 中执行：

~~~powershell
@'
import os
import sqlite3

db = os.path.join(os.environ["USERPROFILE"], ".codex", "logs_2.sqlite")
con = sqlite3.connect(db, timeout=30)
con.execute("PRAGMA busy_timeout=30000")
con.execute("DROP TRIGGER IF EXISTS block_log_inserts")
con.commit()
print("block_log_inserts removed")
con.close()
'@ | python -
~~~

撤销后，新日志会再次写入 logs_2.sqlite。

## 对日常任务的影响

正常任务功能预计不受影响：

- 提问、模型响应和流式输出照常工作。
- 文件读取、修改、构建和终端命令照常工作。
- 工具、插件和 MCP 调用照常工作。
- 任务历史、项目状态和登录状态照常保存。
- 原有日志不会被删除。

主要代价：

- 触发器生效期间，logs_2.sqlite 不再保存新的 TRACE、DEBUG、INFO、WARN 和 ERROR 诊断副本。
- 如果之后发生崩溃或异常，OpenAI 支持人员可获得的本地诊断上下文会减少。
- 日志事件可能仍会在内存中生成和格式化，因此该方案主要解决 SQLite 写盘，不保证消除全部日志 CPU 开销。
- Codex 更新若重建 logs_2.sqlite 或 logs 表，触发器可能消失，需要重新验证或应用。

推荐做法：

1. 日常使用时保持触发器启用，降低日志 WAL 写入。
2. 需要向 OpenAI 提交故障材料时，先撤销触发器。
3. 重新复现问题并收集诊断日志。
4. 排查完成后重新应用触发器。

## 不建议的操作

- 不要在 Codex 运行时直接删除 logs_2.sqlite、WAL 或 SHM 文件。
- 不要把整个 .codex 目录重定向到临时目录；其中还包含会话、配置和状态数据。
- 不要把 logs_2.sqlite 当作会话历史数据库。
- 不要因为 WAL 保持固定大小就认为问题已经消失；应同时检查 MAX(id) 和 WAL 修改时间。

## 结论

在上游正式修复 TRACE 日志过滤和 SQLite 日志层之前，block_log_inserts 触发器是一种小范围、持久、可验证且可撤销的 Windows 规避方案。它能停止 logs_2.sqlite 的新增日志和 WAL 循环写入，同时保留 Codex 的正常任务功能。
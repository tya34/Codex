# Global Codex Instructions

## Cleanup Audit Hard Rule

For any task that generates, downloads, installs, extracts, converts, exports,
builds, runs scripts, writes logs, creates helper scripts, uses temporary
caches, or creates intermediate artifacts, Codex must perform a Cleanup Audit
before sending the final response.

The Cleanup Audit must inspect and clean every task-related location, not just
one fixed folder. The audit scope must include at least:

- Task-related `work`, `outputs`, download, extraction, build, log, cache,
  temporary-script, and intermediate-artifact directories in the current
  workspace.
- System temporary directories actually used by tools, scripts, installers,
  converters, builders, exporters, compressors, decompressors, browser
  verification, rendering verification, or test commands during the task.
- Any other temporary files, log files, cache files, helper scripts, downloaded
  packages, extraction directories, intermediate artifact directories, and
  verification artifacts created or used during the current task.

Only delete auxiliary files produced by the current task and no longer needed.
Do not delete user-provided source files, original inputs, final deliverables,
configuration files, evidence, persistent cache databases, installed
skills/plugins, or files that may still be needed for follow-up work.

If the task created temporary backups, rollback files, or exported backups for
safety, delete them after the operation is verified successful unless the user
explicitly asked to keep them or the backup itself is the requested final
deliverable. If such a backup is kept, mention it in the final response.

Before any recursive deletion, verify that the resolved absolute path is inside
the expected temporary directory, workspace temporary directory, or auxiliary
directory explicitly created for the current task.

Use ordinary deletion without `-Force` for cleanup. Do not add `-Force` to
`Remove-Item` or other cleanup deletion commands. If an item is read-only,
hidden, system-protected, in use, or blocked by permissions or execution policy,
stop deleting that item, preserve it, and report the reason. Do not automatically
retry with forced deletion, change attributes or permissions, or switch tools to
bypass protections. Other independently verified safe cleanup may continue.

Execute cleanup deletions one at a time. First inspect the target path,
attributes, and cleanup scope with a separate read-only command. Then run
exactly one ordinary deletion command per tool execution call, targeting one
verified explicit absolute path. In PowerShell, use Remove-Item -LiteralPath
with -ErrorAction Stop and without -Force. Do not combine deletion with
inspection or other operations in the same command; do not batch deletions
using loops, pipelines, wildcards, path arrays, or chained commands. Wait for
each deletion to finish before executing the next. Verify results afterward
with a separate read-only command. This approach does not guarantee policy
approval. If a deletion is still rejected, stop, preserve the item, and report
the reason as required above; do not automatically retry or bypass the block.

The final response must include one short line beginning with `清理检查：`.
That line must state which relevant directories were checked, what was deleted,
and which final files were kept. If cleanup was not done or was not appropriate,
state the reason.

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

The final response must include one short line beginning with `清理检查：`.
That line must state which relevant directories were checked, what was deleted,
and which final files were kept. If cleanup was not done or was not appropriate,
state the reason.

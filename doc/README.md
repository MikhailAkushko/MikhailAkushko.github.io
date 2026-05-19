# Json Data Importer Documentation

This is the entry point for the Json Data Importer documentation. If you are wiring a new import target for the first time, read the documents in the order below: from the basic scenario to extensions, CI, and diagnostics.

## Terminology

- Type names are written exactly as they appear in code: `ImportSourceConfig`, `ImportJobResult`, `ImportJobRunner`.
- Regular concepts use lowercase: config, config asset, target asset, job.
- Menu item names and exact UI labels stay as they appear in Unity when shown in a code block or when they explicitly describe a menu item.

## Recommended Reading Order

1. [Getting Started](GettingStarted.md) - the minimal path: entry type, target asset, `ImportSourceConfig`, preview/validate/apply.
2. [Import Attributes](ImportAttributes.md) - target attributes, identity, required fields, references, normalization, and discovery metadata.
3. [ImportSourceConfig Reference](ImportSourceConfigReference.md) - config asset fields, pipeline component selection, `ImportExecutionProfile`, and internal incremental state.
4. [Import Execution Flow](ImportExecutionFlow.md) - the runtime flow and the main invariant: editor, batch, and tests run through `ImportJobRunner`.
5. [Pipeline Stages And Extension Points](PipelineStagesAndExtensionPoints.md) - preview/apply stages and the extension point map.
6. [Custom Import Extensions](CustomImportExtensions.md) - practical templates for custom source providers, parsers, adapters/profiles, validators, diff engines, and fingerprint builders.
7. [Backup / Restore / Incremental State](BackupRestoreIncrementalState.md) - safe apply, backup history, restore, and incremental skip conditions.
8. [CI Commands](CICommands.md) - batchmode commands, reports, execution configs, and CLI overrides.
9. [ImportJob Status Troubleshooting](ImportJobStatusTroubleshooting.md) - `ImportJobResult` statuses, common causes, and what to check in messages.
10. [Editor Display Names And Badges](EditorDisplayNamesAndBadges.md) - how the inspector turns display names and suffixes into UI labels/badges.

## Authoritative Sources

- Execution policy order: [ImportSourceConfig Reference](ImportSourceConfigReference.md#executionprofile).
- Pipeline stages and extension points: [Pipeline Stages And Extension Points](PipelineStagesAndExtensionPoints.md).
- Practical custom implementations: [Custom Import Extensions](CustomImportExtensions.md).
- Incremental skip and restore conditions: [Backup / Restore / Incremental State](BackupRestoreIncrementalState.md).
- Creating a backup history asset and when backup is required: [Backup / Restore / Incremental State](BackupRestoreIncrementalState.md#how-to-create-a-backup-history-asset).
- `ImportJobResult` statuses and failure actions: [ImportJob Status Troubleshooting](ImportJobStatusTroubleshooting.md).
- Inspector names and badges: [EditorDisplayNamesAndBadges](EditorDisplayNamesAndBadges.md).

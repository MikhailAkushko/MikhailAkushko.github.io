# Import Execution Flow

This document describes the runtime execution path: how editor, batch, tests, and tools run an import job.

## Public Entry Point

Public entry point for running an import job:

```csharp
ImportJobResult result = await importJobRunner.RunAsync(definition, cancellationToken);
```

Editor, batch, and tests build `ImportJobDefinition`, show diagnostics, or save assets differently, but execution goes through `ImportJobRunner.RunAsync(...)` or the sync wrapper `Run(...)`.

## Main Roles

`ImportJobRunner`

- The only public executor for one import job.
- Owns locks.
- Selects mode: preview, validate, apply.
- Builds `ImportJobResult`.
- Coordinates incremental skip.
- Coordinates backup capture.
- Updates last applied state in `ImportSourceConfig`.

`ImportJobCorePipeline`

- Internal helper, not public API.
- Builds preview from source entries and target entries.
- Parses source only through the resolved parser.
- Stores prepared source entries for apply inside one runner session.
- Runs validation.
- Builds diff.
- Applies prepared source through the selected apply strategy.
- Fills reference lookup context.

`IImportSourceProvider`

- Responsible only for getting raw source data.
- Called from `ImportJobRunner.RunAsync(...)` through `ImportConfigResolver.ReadSourceDataAsync(...)`.
- `ImportSourceConfig` stores the selected provider and its serialized settings as a managed reference.
- Built-in providers cover `TextAsset`, file path, and HTTP sources.
- Google Sheets, custom auth, generated content, and other project-specific sources use a custom `IImportSourceProvider` selected in `ImportSourceConfig.sourceProvider`.

`IImportProfile`

- Describes the target execution contract: target id, display name, entry type, target adapter, backup codec, and default apply strategy.
- Auto profiles can be created for targets described by `[ImportTargetCollection]`, `[ImportTargetDocument]`, or a supported direct-fields shape.
- Custom profiles can be found by discovery when they are concrete `IImportProfile` classes with a public parameterless constructor.
- Custom profile becomes active only when config explicitly selects its `TargetTypeId`.

`IImportBackupCodec`

- Responsible only for serializing and reading backup payload.
- Receives entries already read by `TargetAdapter`.
- Does not parse import source and does not participate in preview/diff/validation.
- Used during backup capture and restore.

`ImportBatchRunner`

- Orchestrator for multiple configs.
- Builds execution order with dependencies.
- Calls `ImportJobRunner.RunAsync(...)` for each config.
- Preview/apply run inside `ImportJobRunner`.

## Unified Chain

```text
ImportSourceConfig inspector
Batch Command
Tests / Tools
     |
     v
ImportJobDefinition
     |
     v
ImportJobRunner.RunAsync(definition, cancellationToken)
     |
     v
ExecuteWithLocksAsync(definition, RunCoreAsync)
     |
     v
RunCoreAsync(definition)
     |
     v
BuildPreviewCoreAsync(definition)
     |
     v
switch ExecutionMode
     |
     +-- PreviewOnly  -> return preview result
     |
     +-- ValidateOnly -> validate result status
     |
     +-- Apply        -> apply strategy -> incremental check -> backup -> apply -> conditional fingerprints -> last applied state
```

## In Detail: ImportJobRunner.RunAsync

### 1. Entry

Caller creates `ImportJobDefinition`:

```csharp
var definition = new ImportJobDefinition
{
    Config = config,
    ExecutionMode = ImportExecutionMode.Apply
};
```

For preview:

```csharp
var definition = new ImportJobDefinition
{
    Config = config,
    ExecutionMode = ImportExecutionMode.PreviewOnly,
    FailOnValidationErrors = false
};
```

Then calls:

```csharp
ImportJobResult result = await runner.RunAsync(definition, cancellationToken);
```

### 2. ExecuteWithLocks

`RunAsync(...)` immediately enters `ExecuteWithLocksAsync(...)`.

This layer:

- takes the config lock;
- takes the target lock;
- prevents two import jobs from working with the same config at the same time;
- prevents two import jobs from working with the same target asset at the same time;
- always releases locks in `finally`.

If a lock cannot be taken, the returned `ImportJobResult` has status:

```text
ImportJobStatus.LockBlocked
```

If `definition.Config` or `targetAsset` is missing, execution continues without locks so core validation can return the normal `MissingConfig` or `MissingTarget` status.

### 3. RunCoreAsync Always Starts With Preview

`RunCoreAsync(...)` first calls:

```csharp
ImportJobResult result = await BuildPreviewCoreAsync(definition, cancellationToken);
```

This is important: `PreviewOnly`, `ValidateOnly`, and `Apply` start with the same preview/build chain.

If preview cannot be built, `RunCoreAsync(...)` immediately returns the failure result.

### 4. BuildPreviewCoreAsync

`BuildPreviewCoreAsync(...)` prepares the job:

```text
Clear prepared source
Validate definition
Validate config
Validate target asset
Resolve profile
Read source data through ImportConfigResolver.ReadSourceDataAsync
Resolve parser from config or compatible auto parser
Build preview through ImportJobCorePipeline
Fill preview stats
Compute canonical fingerprints when fingerprinting is enabled and a compatible builder is resolved
Return result
```

Parser precedence:

```text
if config.selectedParserId resolves:
    use parser explicitly selected in config
else if a compatible auto parser can be created for the resolved target/profile:
    use the auto JSON or CSV parser
else:
    fail job before preview
```

Editor and batch use this order through the shared runner.

### 5. ImportJobCorePipeline.BuildPreview

When a parser is resolved:

```text
IImportParser.ParseEntries(raw source)
prepared source entries = parsed entries
```

If no explicitly selected parser resolves and no compatible auto parser can be created for the resolved target/profile, the job fails before preview.

After that, pipeline builds preview:

```text
Get target entries
Build ImportTypeDescriptor
Resolve identity strategy
Build reference lookup dictionaries
Run ValidationPipeline
Run the resolved diff engine, if configured
Return ImportPreview
```

Prepared source entries remain inside `ImportJobCorePipeline` for the current `ImportJobRunner`. They are needed for apply and, when fingerprinting is enabled, canonical source fingerprint.

### 6. PreviewOnly

Mode-specific behavior:

```text
return preview result
no apply
no backup
no lastApplied* update
```

### 7. ValidateOnly

Mode-specific behavior:

```text
if preview has validation errors and FailOnValidationErrors == true:
    Status = ValidationFailed
    Success = false
else:
    Status = Success
    Success = true
no apply
no backup
no lastApplied* update
```

### 8. Apply

Mode-specific behavior:

```text
Resolve execution profile
Resolve enableIncrementalSkip
Resolve apply strategy
CanSkipIncrementalApply
    if unchanged:
        Status = Skipped
        Success = true
        Applied = false
        AssetsChanged = false
    else:
        ApplyPreparedCore(...)
```

### 9. ApplyPreparedCore

`ApplyPreparedCore(...)` checks:

```text
definition exists
config exists
target exists
profile exists
preview exists
source exists
prepared source exists
validation errors do not block apply
apply strategy exists
```

Then:

```text
Capture backup if enabled
    TargetAdapter.GetTargetEntries(...)
    BackupCodec.CreateBackupPayloadJson(...)
ImportJobCorePipeline.ApplyPreparedSource(...)
Compute final canonical fingerprints when fingerprinting is enabled and a compatible builder is resolved
config.MarkLastAppliedState(...) when final fingerprints were computed
Return success
```

If final fingerprints cannot be computed, config clears last applied state:

```csharp
config.ClearLastAppliedState();
```

## Editor Chain

### Build Preview Button

```text
ImportSourceConfigEditor preview action
     |
     v
ImportJobDefinition { ExecutionMode = PreviewOnly }
     |
     v
ImportJobRunner.RunAsync(definition, cancellationToken)
```

After the result, editor only displays:

- preview counts;
- raw/canonical fingerprints, when available;
- resolved parser/profile/source provider;
- compatible candidates;
- stale state;
- job messages.

### Validate Button

```text
ImportSourceConfigEditor validate action
     |
     v
ImportJobDefinition { ExecutionMode = ValidateOnly }
     |
     v
ImportJobRunner.RunAsync(definition, cancellationToken)
```

### Apply Button

```text
ImportSourceConfigEditor apply action
     |
     v
ImportJobDefinition { ExecutionMode = Apply }
     |
     v
ImportJobRunner.RunAsync(definition, cancellationToken)
     |
     v
if result.Success and mode == Apply:
    EditorUtility.SetDirty(...)
    AssetDatabase.SaveAssets()
```

Editor save is not core import logic. It is the Unity editor persistence layer.

Current editor persistence does not save each modified asset independently at the
moment it changes. After a successful `Apply` result, `ImportSourceConfigEditor`
marks the `ImportSourceConfig`, its `targetAsset`, and its assigned
`ImportBackupHistoryAsset` dirty, then calls `AssetDatabase.SaveAssets()`.
Because the inspector checks `result.Success`, this save path also runs for a
successful `Skipped` apply result, although no core apply mutation is expected
in that case.

This means a backup snapshot captured before `ApplyPreparedSource(...)` is
persisted by the inspector only if the apply job later returns success. If a
custom apply strategy or target adapter throws after backup capture,
`ImportJobRunner` catches the exception and returns `Status = Failed`,
`Success = false`, and leaves `AssetsChanged = false`; the inspector does not
mark or save the backup history in that path. The snapshot may exist in memory
until the asset is reloaded, but current editor persistence does not guarantee
that it is written to disk.

`lastApplied*` fields are also mutated only inside the successful apply path:
the runner calls `MarkLastAppliedState(...)` when final fingerprints are
computed, or `ClearLastAppliedState()` when apply succeeded but final
fingerprints could not be computed. The config is marked dirty by the editor
success-save path above, not by the runner itself.

## Batch Chain

```text
JsonDataImporterBatchCommands
     |
     v
ImportBatchRunner.RunAsync(configs, mode)
     |
     v
ImportExecutionPlanBuilder.Build(configs)
     |
     v
foreach config in ordered configs:
    check dependency result
    build ImportJobDefinition
    ImportJobRunner.RunAsync(definition)
```

Batch differs from editor because it runs multiple configs and respects dependency ordering. Preview/apply run inside `ImportJobRunner`.

Batch persistence uses `ImportJobResult.AssetsChanged`: for each job with
`AssetsChanged == true`, it marks the `ImportSourceConfig`, `targetAsset`, and
assigned `ImportBackupHistoryAsset` dirty. It then calls
`AssetDatabase.SaveAssets()` when the batch result has any changed jobs. A failed
apply after backup capture currently has `AssetsChanged == false`, so batch mode
also does not guarantee that the captured backup snapshot is persisted.

`ImportBatchRunner.Run(...)` and `ImportJobRunner.Run(...)` remain compatibility wrappers over the async chain. They do not contain a separate sync implementation of import.

## Restore Backup

Backup restore is currently a separate operation, not import execution.

It goes through:

```text
ImportSourceConfig.RestoreBackupToTarget(profile)
profile.BackupCodec.ReadBackupPayloadEntries(...)
profile.TargetAdapter.SetTargetEntries(...)
```

Restore is not import execution: it does not go through `ImportJobRunner`, and it does not run parser, validation, diff, source provider, or incremental skip.

## Related Documentation

- [Import Attributes](ImportAttributes.md) - import attributes from the basic auto target shape to stage-specific validation, diff, and fingerprint settings.
- [Pipeline Stages And Extension Points](PipelineStagesAndExtensionPoints.md) - pipeline stage order and where custom source providers, parsers, validators, diff engines, fingerprint builders, and apply strategies participate.
- [Custom Import Extensions](CustomImportExtensions.md) - practical code templates for cases where the auto path does not fit.
- [Backup / Restore / Incremental State](BackupRestoreIncrementalState.md) - safe apply, backup snapshots, restore latest backup, and internal `lastApplied*` state.
- [CI Commands](CICommands.md) - Unity batch mode commands for CI, including `Jdi.Cli.Run`, execution configs, reports, overrides, and exit codes.
- [ImportJob Status Troubleshooting](ImportJobStatusTroubleshooting.md) - result statuses, common failure causes, and what to check in `Messages`.

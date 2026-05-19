# ImportJob Status Troubleshooting

This document helps you read the `ImportJobRunner` result and quickly understand what to fix in an `ImportSourceConfig`, target asset, source provider, or custom integration code.

`Preview`, `Validate`, and `Apply` always start with the same step: the runner builds a preview. Because of that, target, source provider, parser, validator, and diff engine errors can appear before `Apply` tries to change the asset.

## How To Read ImportJobResult

Start with these fields:

| Field | Meaning |
| --- | --- |
| `Status` | Main job result: succeeded, skipped, blocked, or failed at a specific stage. |
| `Success` | `true` when the selected mode completed successfully. It is also `true` for `Skipped`, because the import was intentionally skipped. |
| `Messages` | The first place to look for `Failed`, `InvalidTarget`, `MissingSource`, or `LockBlocked`. |
| `Applied` | `true` if apply actually reached the apply strategy and wrote data to the target. |
| `Skipped` | `true` if apply was skipped by incremental logic. |
| `AssetsChanged` | `true` if the job changed the target asset. Usually `false` for `Preview`, `Validate`, and `Skipped`. |
| `BlockedByLock` | `true` if the job did not start because a config or target asset lock was held. |
| `ValidationErrorCount`, `ValidationWarningCount` | Number of validation diagnostics from preview. |
| `DiffErrorCount`, `DiffWarningCount`, `DiffInfoCount` | Number of diff diagnostics from preview. |
| `AddedCount`, `UpdatedCount`, `RemovedCount`, `UnchangedCount` | Final diff statistics. Useful for checking that the importer saw the expected changes. |
| `CanonicalSourceFingerprint`, `CanonicalTargetFingerprint` | Fingerprints used for incremental skip and state diagnostics. |
| `Preview` | Full preview: prepared source, validation messages, diff result. |
| `Profile`, `ProfileDisplayName`, `ProfileTargetTypeId` | The profile selected for the target. |
| `SourceData`, `SourceDescriptor` | What the source provider returned and how it described the source. |

Common combinations:

| Mode | Successful result |
| --- | --- |
| `PreviewOnly` | `Status = Success`, `Success = true`, `Applied = false`, `AssetsChanged = false`. |
| `ValidateOnly` without blocking validation errors | `Status = Success`, `Success = true`, `Applied = false`, `AssetsChanged = false`. |
| `ValidateOnly` with validation errors and fail policy enabled | `Status = ValidationFailed`, `Success = false`, target is not changed. |
| `Apply` with changes | `Status = Success`, `Success = true`, `Applied = true`, `AssetsChanged = true`. |
| `Apply` without changes by fingerprints | `Status = Skipped`, `Success = true`, `Skipped = true`, `Applied = false`, `AssetsChanged = false`. |

## ImportJobStatus

| Status | Summary |
| --- | --- |
| `Success` | The requested mode completed without blocking errors. |
| `Skipped` | `Apply` was skipped because source/target/strategy matched the last applied state. |
| `MissingDefinition` | The runner received `null` instead of an `ImportJobDefinition`. |
| `MissingConfig` | `ImportJobDefinition.Config` has no `ImportSourceConfig`. |
| `MissingTarget` | The config has no assigned `targetAsset`. |
| `InvalidTarget` | The runner could not find a compatible import profile for the target. |
| `InvalidSourceProvider` | The source provider is not configured, or the config cannot create the provider. |
| `MissingProfile` | Reserved status for old/custom orchestration code. |
| `MissingSource` | A provider was selected, but source data is missing. |
| `MissingPreview` | Apply path received a `null` preview. |
| `MissingPreparedSource` | Preview exists, but prepared source was not stored for apply. |
| `ValidationFailed` | `ValidateOnly` found validation errors while fail policy was enabled. |
| `ApplyBlocked` | `Apply` was stopped by validation errors before target changes. |
| `DependencyBlocked` | A batch dependency did not successfully run before the current config. |
| `LockBlocked` | Another import job holds a lock on the config or target asset. |
| `Canceled` | The job was canceled through `CancellationToken`. |
| `Failed` | Runtime failure without a more specific status; check `Messages`. |

## Common Situations

### Success

`Success` means the requested mode completed without blocking errors. For `Preview` and `Validate`, this does not mean the target was changed. Check `Applied` and `AssetsChanged`.

### Skipped

`Skipped` is not an error. It is the result of incremental skip: the runner compared the current source/target fingerprints with the saved `lastApplied*` state and decided apply was not needed.

Check `CanonicalSourceFingerprint`, `CanonicalTargetFingerprint`, and the config `lastApplied*` fields if the skip looks unexpected. To force data to be applied, disable incremental skip in the execution profile/job definition, change source/target, or clear the config `lastApplied*` state.

### MissingDefinition And MissingConfig

Both statuses usually mean an integration code, batch command, or test issue.

`MissingDefinition` appears when `ImportJobRunner` receives `null` instead of an `ImportJobDefinition`. Create an `ImportJobDefinition` before calling `RunAsync`.

`MissingConfig` means `ImportJobDefinition.Config` is empty. Pass the config asset into the job definition.

### MissingTarget

`MissingTarget` means the importer does not know which asset to change. The fix always starts with `ImportSourceConfig.targetAsset`: assign the database, document, or collection asset to `targetAsset`.

### InvalidTarget And MissingProfile

`InvalidTarget` means a target asset is assigned, but the system could not understand how to work with it.

For the auto path, these conditions are checked:

- target class is marked with `[ImportTargetMetadata(...)]`;
- metadata provides a stable `targetTypeId`, a readable `displayName`, and the correct `entryType`;
- target has `[ImportTargetCollection]`, `[ImportTargetDocument]`, or direct fields shape;
- selected `preferredTargetTypeId` does not point to another incompatible profile.

If the target shape is custom, use a custom `IImportProfile` and custom `IImportTargetAdapter`.

`MissingProfile` is reserved in the enum. The standard `ImportJobRunner` returns `InvalidTarget` instead. If you see `MissingProfile`, the result came from a non-standard runner path or old wrapper code. Check the same things as for `InvalidTarget`.

### InvalidSourceProvider And MissingSource

`InvalidSourceProvider` is a provider selection problem: the provider is not configured or cannot be created.

`MissingSource` means the provider is already selected, but the actual data is missing. For the standard `TextAssetImportSourceProvider`, this most often means the `TextAsset` field is empty.

For a custom provider, check that it returns a non-empty `ImportSourceData`.

### MissingPreview And MissingPreparedSource

Both statuses mean apply received an incomplete prepared session. In the standard `ImportJobRunner.RunAsync`, preview is built first, then apply runs in the same runner/pipeline session.

`MissingPreview` means the apply path received a `null` preview.

`MissingPreparedSource` means preview exists, but the pipeline did not store source entries for apply.

For normal import execution, the public entry point is `ImportJobRunner.RunAsync`.

### ValidationFailed And ApplyBlocked

Both statuses are related to validation errors and fail policy.

`ValidationFailed` appears in `ValidateOnly` mode. It means the importer checked the data, found errors, and returned a failed validation result.

`ApplyBlocked` appears in `Apply` mode. It means preview was built and errors were found, but the target asset was not changed because apply was blocked by policy.

Open validation messages in the preview/report, then fix the source data or validators. For a diagnostic run without failure, temporarily disable fail policy in the execution profile/job definition.

### DependencyBlocked

`DependencyBlocked` happens in a batch run when a config depends on another config that did not run earlier in the current run or did not finish successfully.

Check the batch run order and dependency config results. Fix the dependency job and rerun the batch.

### LockBlocked

`LockBlocked` protects against concurrent writes to the same config or target asset. Usually it is enough to wait for the other import to finish. In CI, this often means the same config was included in several parallel jobs.

### Canceled

`Canceled` means the job was canceled through `CancellationToken`. Rerun the import. If cancellation is unexpected, check the editor tool, CI timeout, or custom progress/cancellation integration.

### Failed

`Failed` is a general fallback for runtime failures: parser not selected/found, validator/diff/fingerprint component not found or incompatible, backup not written, apply strategy missing, or an exception from parser/apply/custom code.

Read `Messages` first. Usually they identify which component id was not resolved, which parser is missing, why backup failed, or which exception message came from custom code.

## What To Check In Messages

`Messages` usually contains the exact reason for diagnostic statuses. Example meanings:

- `No source provider is configured.` - an empty or invalid source provider is selected.
- `Configured Text Asset source provider has no TextAsset assigned.` - provider exists, but source asset is not assigned.
- `Parser is not selected and no auto parser could be resolved for this target.` - no auto parser exists and no custom parser is selected.
- `Selected parser '...' could not be resolved.` - the config stores a parser id that discovery no longer finds.
- `Selected validators could not be resolved: ...` - one of the validator ids was removed, renamed, or is incompatible with `entryType`.
- `Selected diff engine could not be resolved: ...` - the selected diff engine is no longer available.
- `Apply strategy could not be resolved.` - the profile has no default apply strategy and no override is set.
- `Import job is blocked because the config/target asset is already locked...` - a parallel import holds the lock.

If `Status = Failed`, read `Messages` first, then check the selected component ids in `ImportSourceConfig`.

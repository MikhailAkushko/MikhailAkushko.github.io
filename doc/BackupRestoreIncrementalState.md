# Backup / Restore / Incremental State

This document describes safe `Apply`: when backup is created, why apply can stop before changing the target, how restore works, and why `lastApplied*` fields are considered internal state.

## Safe Apply

`Apply` does not start by writing to the target asset. It goes through several preflight stages:

```text
Build preview
-> validation policy check
-> apply strategy resolution
-> backup capture
-> ApplyPreparedSource
-> final fingerprints
-> last applied state update
```

Before `ApplyPreparedSource`, the target asset has not been changed yet. This is important: many errors intentionally finish the job before target mutation.

## When Backup Is Required

Backup is required when `CaptureBackupOnApply` is enabled for the job.

The full execution policy priority order is described in [ImportSourceConfig Reference](ImportSourceConfigReference.md#executionprofile).

Backup is enabled by default. When it is enabled, `ImportSourceConfig.backupHistory` must have an assigned `ImportBackupHistoryAsset`. If `backupHistory` is not assigned, apply finishes with `Failed` before changing the target asset.

For production/CI imports, keep backup enabled. Disabling it only makes sense for temporary test runs, generated targets, or cases where the target is easy to recreate from source.

## How To Create A Backup History Asset

Create the asset through the Unity menu:

```text
Create > Json Data Importer > Import Backup History
```

Assign the created `ImportBackupHistory.asset` to `ImportSourceConfig.backupHistory`. If one config writes to one target asset, usually use a separate backup history asset for that config. For several independent configs, do not reuse one history without a clear reason: snapshots store `targetTypeId`, target guid/path, and metadata for a specific apply.

## Why Apply Can Fail Before Target Changes

Apply can stop before target asset mutation for several reasons:

| Reason | Status | What happens |
| --- | --- | --- |
| `ImportJobDefinition` was not passed. | `MissingDefinition` | The runner cannot start the job. |
| `ImportSourceConfig` was not passed. | `MissingConfig` | The runner does not know which config to run. |
| `targetAsset` is not assigned. | `MissingTarget` | There is no target asset for preview/apply. |
| Target is incompatible with profile. | `InvalidTarget` | Adapter/profile cannot be resolved. |
| Source provider is not configured. | `InvalidSourceProvider` | There is nothing to read. |
| Source is empty or `TextAsset` is not assigned. | `MissingSource` | A provider is selected, but source data is missing. |
| Parser/validator/diff/fingerprint/apply component is not resolved. | `Failed` | The config references an unavailable or incompatible component. |
| Preview was not created. | `MissingPreview` | Apply path was called without a preview result. |
| Prepared source is missing. | `MissingPreparedSource` | Apply path was called outside the runner session, or preview did not prepare source entries. |
| Validation errors block apply. | `ApplyBlocked` | Target is not changed because `Fail On Validation Errors` is enabled. |
| Apply strategy was not found. | `Failed` | There is no component to build final entries. |
| Backup capture is enabled, but backup cannot be created. | `Failed` | Target is not changed: backup capture runs before apply. |
| Another import holds the config or target. | `LockBlocked` | The runner does not enter the execution path. |

After a successful backup capture, `ApplyPreparedSource` can still fail if a custom apply strategy or target adapter throws an exception. In that case, backup is already saved and can be used for restore.

## What Is Stored In A Backup Snapshot

The snapshot is stored in `ImportBackupHistoryAsset`. A new snapshot is inserted at the beginning of the list, so the latest snapshot always has index `0`.

The snapshot stores:

| Field | Stored value |
| --- | --- |
| `targetTypeId` | `profile.TargetTypeId`, so restore does not apply the snapshot through another profile. |
| `targetDisplayName` | Human-readable target name from the adapter. |
| `targetAssetGuid` | Unity asset GUID of the target, if it could be obtained. |
| `targetAssetPath` | Asset path of the target, if it could be obtained. |
| `createdAtUtc` | UTC timestamp when the snapshot was created. |
| `backupPayloadJson` | Serialized target entries before import. Created through `profile.BackupCodec`. |
| `importMetadata.sourceDescriptor` | Source provider description for diagnostics. |
| `importMetadata.recordCountBeforeImport` | Number of target entries before import. |
| `importMetadata.updatedCount`, `addedCount`, `removedCount`, `unchangedCount` | Diff summary from the preview built before apply. |
| `importMetadata.summaryLine` | Short summary line for UI. |
| `importMetadata.sourceFingerprintSha256` | SHA-256 of raw source content at apply time. |
| `importMetadata.sourceFingerprintShort` | Short source fingerprint for UI. |
| `importMetadata.sourceContentLength` | Raw source content length. |

The snapshot does not store the whole target asset as a Unity asset copy. It stores the entries returned by `TargetAdapter.GetTargetEntries(...)` before import, serialized through `BackupCodec`.

Raw source content is also not stored as a separate snapshot field; only its fingerprint and length are saved.

## maxBackupHistory

`ImportSourceConfig.maxBackupHistory` limits the number of snapshots in the assigned `backupHistory`.

Rules:

- a new snapshot is added to the beginning;
- if `maxBackupHistory < 1`, `1` is used effectively;
- old snapshots are removed from the end of the list;
- the field affects capture only, not restore.

## Restore Latest Backup

Restore latest backup is a separate editor operation, not import apply.

The editor flow:

```text
Check targetAsset
-> check backupHistory.HasSnapshot
-> take backupHistory.Snapshots[0]
-> resolve profile by snapshot.targetTypeId
-> show confirmation dialog
-> Undo.RegisterCompleteObjectUndo(targetAsset)
-> config.RestoreBackupToTarget(profile)
-> SetDirty + SaveAssets
```

The core restore flow:

```text
backupHistory.RestoreLatestTo(targetAsset, profile)
-> RestoreTo(targetAsset, profile, snapshotIndex: 0)
-> check snapshot.targetTypeId == profile.TargetTypeId
-> validate target asset guid/path
-> read entries through profile.BackupCodec
-> build final entries through ReplaceAllApplyStrategy
-> profile.TargetAdapter.SetTargetEntries(targetAsset, finalEntries)
```

Restore replaces current target entries with data from the backup snapshot. It does not run parser, validation, diff, source provider, or incremental skip.

Restore also does not update `lastApplied*` state. After restore, target state usually differs from the state recorded after the last successful apply, so the next apply runs as changed until fingerprints match again after a successful apply.

## lastApplied* State

The `lastApplied*` fields in `ImportSourceConfig` are internal incremental state:

| Field | Stored value |
| --- | --- |
| `lastAppliedSourceFingerprint` | Prepared source fingerprint after the last successful apply. |
| `lastAppliedTargetFingerprint` | Target fingerprint after the last successful apply. |
| `lastAppliedProfileTargetTypeId` | `profile.TargetTypeId` used by the last successful apply. |
| `lastAppliedApplyStrategyId` | `applyStrategy.StrategyId` used by the last successful apply. |
| `lastAppliedAtUtc` | UTC timestamp of the last successful apply. |

The runner updates these fields only after a successful apply when both the source fingerprint and final target fingerprint were computed.

If apply succeeded but final fingerprints cannot be computed, the runner calls `ClearLastAppliedState()`. This is safer than keeping the old state: the next apply will not be skipped incorrectly.

## Why lastApplied* Should Not Be Edited Manually

These fields participate in the `CanSkipIncremental(...)` decision. Manual editing can cause two bad scenarios:

- false skip: the importer decides source/target did not change and does not apply real changes;
- lost skip: the importer reapplies unchanged source because state became incomplete or mismatched.

To force the next apply, do not hand-pick values. Clear the `lastApplied*` state or temporarily disable incremental skip in the execution profile/job override.

## When Incremental Skip Works

`Skipped` is possible only in `Apply` mode.

Conditions:

- `enableIncrementalSkip` is enabled;
- apply strategy resolved;
- preview was built without validation errors;
- source fingerprint was computed;
- current target fingerprint was computed;
- all `lastApplied*` fields are filled;
- current source fingerprint matches `lastAppliedSourceFingerprint`;
- current target fingerprint matches `lastAppliedTargetFingerprint`;
- current `profile.TargetTypeId` matches `lastAppliedProfileTargetTypeId`;
- current `applyStrategy.StrategyId` matches `lastAppliedApplyStrategyId`.

If all conditions are met, the runner returns:

```text
Status = Skipped
Success = true
Applied = false
AssetsChanged = false
```

Backup is not created for `Skipped`, because apply does not run and the target asset does not change.

## When Incremental Skip Does Not Work

Incremental skip does not work when:

- this is the first apply and `lastApplied*` state is still empty;
- `enableIncrementalSkip` is disabled;
- `enableFingerprinting` is disabled;
- fingerprint builder is not resolved;
- source fingerprint cannot be computed;
- target fingerprint cannot be computed;
- preview contains validation errors;
- source content changed;
- target asset was changed manually after the last apply;
- target was restored from backup;
- another profile is selected;
- another apply strategy is selected;
- custom identity strategy, fingerprint builder, or field policy changed the canonical fingerprint;
- the previous apply failed;
- the previous successful apply could not save final fingerprints and cleared state.

This is normal behavior. Incremental skip is conservative: if the pipeline cannot prove that source and target match the last successful apply, it runs apply.

## Practical Check

Before enabling safe apply in a production config, check the settings first, then the expected behavior.

### Settings Before The First Apply

1. `executionProfile.captureBackupOnApply = true`.
2. `backupHistory` is assigned.
3. `maxBackupHistory` is greater than `0`.
4. `executionProfile.failOnValidationErrors = true`.
5. `executionProfile.enableFingerprinting = true`.
6. `executionProfile.enableIncrementalSkip = true`.
7. The selected profile has a `BackupCodec`.
8. The selected target adapter correctly implements `GetTargetEntries` and `SetTargetEntries`.

### Check After The First Apply

1. After the first successful apply, the config has `lastApplied*` values.
2. Reapplying without changes returns `Skipped`.
3. Restore latest backup returns the target to its previous state.

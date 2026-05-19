# Pipeline Stages And Extension Points

This document describes what happens inside the import pipeline and which supported extension points participate in each stage.

The public execution path goes through `ImportJobRunner.RunAsync(...)`. `ImportJobCorePipeline` is an internal layer that builds preview and applies already prepared source inside one runner session.

Practical code templates for custom providers/parsers/adapters/profiles/validators/diff/fingerprint are in [Custom Import Extensions](CustomImportExtensions.md). Safe apply, backup/restore, and incremental state are described in [Backup / Restore / Incremental State](BackupRestoreIncrementalState.md).

## Overall Chain

```text
ImportSourceConfig
-> ImportJobDefinition
-> ImportJobRunner.RunAsync(...)
-> Source Provider
-> Parser
-> Parser Diagnostics
-> TargetAdapter.GetTargetEntries
-> Metadata Descriptor
-> Identity Strategy
-> Reference Lookup
-> Validation
-> Diff
-> Fingerprint
-> Preview / Validate / Apply decision
-> Apply Strategy
-> TargetAdapter.SetTargetEntries
-> Final Fingerprint
-> Last Applied State
```

`Preview`, `Validate`, and `Apply` start with the same preview path. The difference appears after preview: `Preview` returns the result immediately, `Validate` checks validation policy, and `Apply` runs incremental skip check, backup, and only then applies prepared source.

## Preview Stages

| Stage | What it does | Extension point | Important details |
| --- | --- | --- | --- |
| Source Provider | Reads raw source data and returns `ImportSourceData`. | `IImportSourceProvider` | Provider does not parse entries and does not change the target. Regular JSON usually uses TextAsset provider. |
| Parser | Turns `sourceData.RawContent` into a list of source entries. | `IImportParser`, `IImportParser<TEntry>` | `parser.EntryType` matches `profile.EntryType`. Result is stored as prepared source inside the pipeline session. |
| Parser Diagnostics | Reads diagnostics from the parser after parse. | `IImportParserDiagnosticsProvider` | These diagnostics are added to the shared `ValidationMessages` list, so parser can show warnings/errors next to validation. |
| Target Entries | Reads current entries from the target asset. | `IImportTargetAdapter.GetTargetEntries` | These entries are used for validation context, diff, target fingerprint, and backup. |
| Metadata Descriptor | Builds a normalized description of entry type. | Attributes and field policies | `ImportTypeDescriptor` is used by validation, diff, fingerprint, and apply. Usually users affect it through attributes, not custom code. |
| Identity Strategy | Defines stable identity/key for entries. | `IIdentityStrategy` | Identity is used for reference checks, diff, fingerprint, and apply strategy. |
| Reference Lookup | Collects external reference entries before validation. | `IImportReferenceLookupProvider` | If entry type contains reference members to other entry types, lookup provider injects entries/descriptors/identity strategies into validation context. |
| Validation | Checks prepared source with target entries and reference lookup. | `IImportValidator` | Parser diagnostics and validation diagnostics are combined in `ImportPreview.ValidationMessages`. |
| Diff | Compares source entries and target entries. | `IImportDiffEngine`, `IImportDiffEngineWithResult` | If diff engine throws an exception, pipeline isolates the error as a warning diagnostic and continues preview without changes. |
| Fingerprint | Builds canonical fingerprints for prepared source and current target. | `IImportFingerprintBuilder` | Fingerprint is computed by `ImportJobRunner`, not `ImportJobCorePipeline`. It uses prepared source from pipeline and target entries from adapter. |

## Apply Stages

Apply does not parse source again. It works only with prepared source created on the preview stage in the same `ImportJobRunner` session.

```text
Preview already built
-> validation errors policy check
-> apply strategy resolution
-> optional backup capture
-> ImportJobCorePipeline.ApplyPreparedSource(...)
-> TargetAdapter.GetTargetEntries
-> Metadata Descriptor
-> Identity Strategy
-> Apply Strategy.BuildFinalEntries
-> TargetAdapter.SetTargetEntries
-> final source/target fingerprints
-> config.MarkLastAppliedState(...)
```

| Stage | What it does | Extension point | Important details |
| --- | --- | --- | --- |
| Validation policy check | Blocks apply when validation errors exist and fail policy is enabled. | `ImportExecutionProfile.failOnValidationErrors`, `ImportJobDefinition.FailOnValidationErrors` | When blocked, returns `ApplyBlocked`; target does not change. |
| Apply Strategy Resolution | Selects strategy from override or default profile strategy. | `IImportApplyStrategy`, `IImportProfile.GetDefaultApplyStrategy()` | If strategy is not found, job finishes as `Failed`. |
| Backup Capture | Saves current target entries before apply when enabled. | `IImportBackupCodec`, `ImportBackupHistory` | Backup reads entries through `TargetAdapter.GetTargetEntries`, not through parser. |
| Prepared Source Read | Takes entries saved after parser. | No public extension point | If prepared source is missing, apply returns `MissingPreparedSource`. |
| Current Target Read | Reads target entries before final entries are built. | `IImportTargetAdapter.GetTargetEntries` | Apply strategy receives both source entries and current target entries. |
| Build Final Entries | Builds the final entries list. | `IImportApplyStrategy.BuildFinalEntries` | Replace/merge/update behavior is decided here. Writing to the Unity asset happens at the `Target Write` stage. |
| Target Write | Writes final entries into the target asset. | `IImportTargetAdapter.SetTargetEntries` | This is the actual target asset mutation point. |
| Final Fingerprint | Computes fingerprints after apply. | `IImportFingerprintBuilder` | If fingerprints are computed, config updates `lastApplied*` state. If not, state is cleared. |

## Important Invariants

### Parser Diagnostics Become Validation Messages

Parser can implement `IImportParserDiagnosticsProvider`. After `ParseEntries(...)`, pipeline calls `GetDiagnostics()` and adds those messages to `ImportPreview.ValidationMessages`.

This means:

- parser can report unmapped fields, unknown enum values, fallback conversion, or incomplete data;
- these messages are visible to the user together with regular validation;
- validation policy can block validate/apply when a parser diagnostic has error severity.

### Reference Lookup Runs Before Validation

Before validators run, pipeline builds reference dictionaries:

- entries of the current `profile.EntryType`;
- descriptor of the current `profile.EntryType`;
- identity strategy of the current `profile.EntryType`;
- additional reference entries found through `IImportReferenceLookupProvider`.

Validators receive a ready `ImportValidationContext`, where references can be checked not only inside the current source but also against data from other import targets.

Minimal shape of a project-specific lookup provider:

```csharp
#if UNITY_EDITOR
using System;
using System.Collections.Generic;
using JsonDataImporter.Runtime.Core;
using JsonDataImporter.Runtime.Core.Identity;
using JsonDataImporter.Runtime.Core.Validation;
using UnityEditor;
using UnityEngine;
using Object = UnityEngine.Object;

[InitializeOnLoad]
public static class ProjectImportReferenceBootstrap
{
    static ProjectImportReferenceBootstrap()
    {
        ImportReferenceLookupProviderRegistry.Provider =
            new ProjectImportReferenceLookupProvider();
    }
}

public sealed class ProjectImportReferenceLookupProvider : IImportReferenceLookupProvider
{
    public bool TryResolve(
        Type entryType,
        IImportProfile activeProfile,
        Object activeTargetAsset,
        out ImportReferenceLookupResult result)
    {
        result = null;

        if (entryType != typeof(ItemRecord))
            return false;

        ItemDatabase database = LoadItemDatabase(activeTargetAsset);
        if (database == null)
            return false;

        result = new ImportReferenceLookupResult
        {
            TargetTypeId = "items_database",
            EntryType = typeof(ItemRecord),
            Entries = ToObjects(database.items),
            IdentityStrategy = new DefaultIdentityStrategy()
        };

        return true;
    }

    private static ItemDatabase LoadItemDatabase(Object activeTargetAsset)
    {
        string[] guids = AssetDatabase.FindAssets("t:ItemDatabase");
        for (int i = 0; i < guids.Length; i++)
        {
            string path = AssetDatabase.GUIDToAssetPath(guids[i]);
            var database = AssetDatabase.LoadAssetAtPath<ItemDatabase>(path);
            if (database != null && !ReferenceEquals(database, activeTargetAsset))
                return database;
        }

        return null;
    }

    private static IReadOnlyList<object> ToObjects(IReadOnlyList<ItemRecord> items)
    {
        var result = new List<object>();
        if (items == null)
            return result;

        for (int i = 0; i < items.Count; i++)
            result.Add(items[i]);

        return result;
    }
}
#endif
```

This provider is responsible only for supplying reference entries. The actual `[ImportReference]` check is performed by the built-in `default_reference` validator.

### Diff Failures Are Isolated

If a diff engine fails, `ImportJobCorePipeline` catches the exception and returns an `ImportDiffDiagnostic` with `Warning` severity.

Result:

- preview remains available;
- validation messages are not lost;
- diff changes can be empty;
- the user sees a warning that the diff engine was isolated from the pipeline.

Parser, source provider, target adapter, and apply strategy are not isolated this way. Their errors can finish the job with `Failed`.

### Apply Uses Only Prepared Source

Prepared source means source entries produced by the parser during preview. Apply takes exactly those entries through internal pipeline state.

Apply does not reread the source provider and does not run parser again. Parsing behavior is defined by the preview stage, and apply strategy receives already prepared source entries.

## Non-Obvious Data Dependencies

- Validation uses reference lookup dictionaries; diff, fingerprint, and apply strategy do not use references.
- Diff uses prepared source and current target entries, but its diagnostics do not block preview.
- Fingerprint computes source fingerprint from prepared source and target fingerprint from entries returned by target adapter.
- Apply strategy receives prepared source and current target entries; it builds final entries but does not write to the target asset.
- `TargetAdapter.SetTargetEntries` receives only final entries; source provider, parser, and reference lookup are no longer involved at that point.

## Quick Pick

| Task | Extension point | Where it is selected |
| --- | --- | --- |
| Read data not stored in `TextAsset` | `IImportSourceProvider` | Source provider settings in `ImportSourceConfig`. |
| Parse a custom source shape | `IImportParser` | `selectedParserId` or auto parser resolution. |
| Show parser warnings/errors | `IImportParserDiagnosticsProvider` | Automatically, when parser implements the interface. |
| Describe a new target shape | `IImportProfile` + `IImportTargetAdapter` | Profile resolution by target asset and `preferredTargetTypeId`. |
| Change target read/write behavior | `IImportTargetAdapter` | Through the selected profile. |
| Change entry comparison keys | `IIdentityStrategy` | Identity strategy resolution for profile/entry type. |
| Check data before apply | `IImportValidator` | `selectedValidatorIds`. |
| Attach external reference data | `IImportReferenceLookupProvider` | Global provider registry, used before validation. |
| Change how changes are displayed | `IImportDiffEngine` or `IImportDiffEngineWithResult` | `selectedDiffEngineId`. |
| Change incremental fingerprints | `IImportFingerprintBuilder` | `selectedFingerprintBuilderId`, execution profile fingerprint settings. |
| Change replace/merge behavior | `IImportApplyStrategy` | `selectedApplyStrategyId`, job override, or default profile strategy. |
| Change backup payload | `IImportBackupCodec` | Through the selected profile. |

Class templates are in [Custom Import Extensions](CustomImportExtensions.md). This table states which stage participates in each task without duplicating implementation details. Parser is responsible for source shape, validator for project rules, diff engine for displaying changes, apply strategy for replace/merge behavior, and fingerprint builder for incremental skip.

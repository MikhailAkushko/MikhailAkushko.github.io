# Custom Import Extensions

This reference shows the supported extension points used when the auto path does not fit.

The auto path covers simple cases: a built-in text source, JSON/CSV shape matching the entry type, and a target asset described through `[ImportTargetCollection]`, `[ImportTargetDocument]`, or direct fields. If one stage needs project-specific behavior, extend that stage only. The full stage map is in [Pipeline Stages And Extension Points](PipelineStagesAndExtensionPoints.md).

## Quick Pick

| Problem | Extension point | Where it is selected |
| --- | --- | --- |
| Source needs project-specific access or generated text. | `IImportSourceProvider` | `Source Provider` in `ImportSourceConfig`. |
| Source has a custom format or shape. | `IImportParser` | `selectedParserId` / Parser dropdown. |
| Target asset cannot be read/written by an auto adapter. | `IImportTargetAdapter` | Through custom `IImportProfile`. |
| Target type, entry type, adapter, backup, and default apply strategy are defined together. | `IImportProfile` | Profile resolution / `preferredTargetTypeId`. |
| Project-specific checks are needed. | `IImportValidator` | `selectedValidatorIds`. |
| Preview changes are displayed by domain rules. | `IImportDiffEngine` or `IImportDiffEngineWithResult` | `selectedDiffEngineId`. |
| Incremental skip uses a custom data set. | `IImportFingerprintBuilder` | `selectedFingerprintBuilderId` and execution profile. |
| Replace/merge behavior must follow project rules. | `IImportApplyStrategy` | `selectedApplyStrategyId`, job override, or profile default. |

The exact stage call order and data transition between preview/apply are described in [Pipeline Stages And Extension Points](PipelineStagesAndExtensionPoints.md).

Examples below use sample types:

```csharp
using System;
using System.Collections.Generic;
using UnityEngine;

[Serializable]
public sealed class QuestRow
{
    public string id;
    public string title;
    public int reward;
    public string editorNote;
}

[Serializable]
public sealed class QuestRuntimeConfig
{
    public List<QuestRow> quests = new();
}

public sealed class QuestDatabase : ScriptableObject
{
    public QuestRuntimeConfig runtime = new();
}
```

`QuestDatabase.runtime.quests` is intentionally not a direct `[ImportTargetCollection]` field. This is an example of a target shape that can need a custom adapter/profile.

## Custom IImportSourceProvider

Custom source provider is used when built-in `Text Asset`, `File Path`, or `HTTP` providers do not cover how raw text is obtained: Google Sheets, custom authentication, generated content, decrypted content, or an internal tool API.

Source provider reads raw content. Parsing, validation, and target asset changes are performed by later pipeline stages.

```csharp
using System;
using System.Net.Http;
using System.Threading;
using System.Threading.Tasks;
using JsonDataImporter.Runtime.Core;
using JsonDataImporter.Runtime.Core.Metadata;
using JsonDataImporter.Runtime.Core.Source;
using JsonDataImporter.Runtime.Utils;
using UnityEngine;

[Serializable]
[ImportSourceProviderMetadata("quest_http_json", "Quest HTTP JSON [Custom]", "Custom")]
public sealed class QuestHttpSourceProvider : IImportSourceProvider
{
    [SerializeField]
    private string _url;

    [SerializeField]
    private int _timeoutSeconds = 30;

    public async Task<ImportSourceData> ReadAsync(CancellationToken cancellationToken)
    {
        if (string.IsNullOrWhiteSpace(_url))
            throw new InvalidOperationException("Quest source URL is not assigned.");

        using var client = new HttpClient
        {
            Timeout = TimeSpan.FromSeconds(Math.Max(1, _timeoutSeconds))
        };

        using HttpResponseMessage response = await client.GetAsync(_url, cancellationToken)
            .ConfigureAwait(false);
        response.EnsureSuccessStatusCode();

        string rawContent = await response.Content.ReadAsStringAsync()
            .ConfigureAwait(false);

        return new ImportSourceData
        {
            SourceDescriptor = $"Quest HTTP JSON: {_url}",
            RawContent = rawContent,
            FingerprintSha256 = SourceFingerprintUtility.ComputeSha256(rawContent)
        };
    }
}
```

Requirements for discovery and inspector:

- concrete class implements `IImportSourceProvider`;
- add `[Serializable]` if provider stores settings in `ImportSourceConfig`;
- add `[ImportSourceProviderMetadata(...)]`;
- public parameterless constructor is required unless this is a built-in special case;
- keep settings in `[SerializeField]` fields because provider lives in config as a managed reference.

Serialized fields are saved into the asset, so long-lived secrets do not belong there.

## Custom IImportParser

`IImportParser` converts raw source text from `ImportSourceData.RawContent` into normalized source entries. It runs after `IImportSourceProvider` has read the source and before validation, diff, fingerprinting, backup, and apply. A parser does not read files or URLs, select the target asset, validate references, build diffs, or write to the target asset; it only turns the already-read source string into entries.

The selected parser must produce entries whose `EntryType` matches the resolved import profile `EntryType`. `IImportParser<TEntry>` is the typed convenience interface, but the runtime pipeline still calls `ParseEntries(...)`.

Custom parser is used when source cannot be parsed directly by the auto JSON/CSV parser:

- CSV dialect, TSV, YAML, or custom text format;
- JSON root does not match `entryType`;
- field names, enum values, dates, or object-map-to-list conversion is required;
- the difference between a missing field and explicit `null` must be preserved;
- user-facing parser diagnostics are needed.

This is a deliberately simplified parser example. It does not support quoted commas, escaped quotes, or multiline CSV records. Use the built-in CSV parser or a proper CSV implementation for production data.

```csharp
using System;
using System.Collections.Generic;
using JsonDataImporter.Runtime.Core;
using JsonDataImporter.Runtime.Core.Metadata;
using JsonDataImporter.Runtime.Core.Source;

[ImportParserMetadata(ParserId, "Quest CSV Parser [Custom]", typeof(QuestRow), "Custom")]
public sealed class QuestCsvParser :
    IImportParser<QuestRow>,
    IImportParserDiagnosticsProvider
{
    public const string ParserId = "quest_csv_parser";

    private readonly List<ImportValidationMessage> _diagnostics = new();

    public Type EntryType => typeof(QuestRow);

    public IReadOnlyList<QuestRow> ParseTypedEntries(string rawContent)
    {
        _diagnostics.Clear();

        if (string.IsNullOrWhiteSpace(rawContent))
            throw new ArgumentException("Quest CSV content is empty.", nameof(rawContent));

        var result = new List<QuestRow>();
        string[] lines = rawContent.Split(new[] { "\r\n", "\n" }, StringSplitOptions.None);

        for (int i = 1; i < lines.Length; i++)
        {
            string line = lines[i];
            if (string.IsNullOrWhiteSpace(line))
                continue;

            string[] columns = line.Split(',');
            if (columns.Length < 3)
            {
                _diagnostics.Add(new ImportValidationMessage
                {
                    EntryKey = $"line:{i + 1}",
                    Message = "Quest CSV row has fewer than 3 columns.",
                    IsError = true
                });
                continue;
            }

            if (!int.TryParse(columns[2], out int reward))
            {
                _diagnostics.Add(new ImportValidationMessage
                {
                    EntryKey = columns[0],
                    Message = "Quest reward is not a valid integer.",
                    IsError = true
                });
                reward = 0;
            }

            result.Add(new QuestRow
            {
                id = columns[0].Trim(),
                title = columns[1].Trim(),
                reward = reward
            });
        }

        return result;
    }

    public IReadOnlyList<object> ParseEntries(string rawContent)
    {
        IReadOnlyList<QuestRow> typedEntries = ParseTypedEntries(rawContent);
        var result = new List<object>(typedEntries.Count);

        for (int i = 0; i < typedEntries.Count; i++)
            result.Add(typedEntries[i]);

        return result;
    }

    public IReadOnlyList<ImportValidationMessage> GetDiagnostics()
    {
        return _diagnostics;
    }
}
```

Parser diagnostics are added to `ImportPreview.ValidationMessages`. If a diagnostic has `IsError = true`, `ValidateOnly` can return `ValidationFailed`, and `Apply` can return `ApplyBlocked` when fail policy is enabled.

Requirements:

- concrete class implements `IImportParser` or `IImportParser<TEntry>`;
- public parameterless constructor, so discovery can create it;
- `[ImportParserMetadata(...)]` with stable parser id, display name, and entry type;
- `EntryType` and metadata `entryType` match the selected profile `EntryType`;
- `ParseEntries(...)` returns normalized entries or throws for unrecoverable parse errors;
- optional `IImportParserDiagnosticsProvider` returns warnings/errors from the last parse.

Related parser documentation:

- Parser field and auto JSON/CSV selection: [`selectedParserId`](ImportSourceConfigReference.md#selectedparserid).
- Beginner scenario and simplified parser example: [Getting Started - When a Custom Parser Is Needed](GettingStarted.md#5-when-a-custom-parser-is-needed).
- Parser discovery metadata: [Import Attributes - Extension Discovery Attributes](ImportAttributes.md#extension-discovery-attributes).
- Runtime parser resolution and parse call: [Import Execution Flow - BuildPreviewCoreAsync](ImportExecutionFlow.md#4-buildpreviewcoreasync) and [ImportJobCorePipeline.BuildPreview](ImportExecutionFlow.md#5-importjobcorepipelinebuildpreview).
- Pipeline position, diagnostics, and prepared source: [Pipeline Stages And Extension Points](PipelineStagesAndExtensionPoints.md#preview-stages).
- Parser display names in the inspector: [Editor Display Names And Badges](EditorDisplayNamesAndBadges.md).
- Missing parser and parser exceptions: [ImportJob Status Troubleshooting - Failed](ImportJobStatusTroubleshooting.md#failed).

## Custom IImportTargetAdapter

Custom adapter is used when the target asset has a custom shape:

- data is stored deep inside a nested object;
- target stores a different type than the parser returns;
- deep clone/convert of source entry is required before writing;
- apply writes one document into a custom field;
- target requires special clearing, merge, or normalization logic.

Adapter is responsible for reading and writing target entries. It does not choose parser, validate source, or build diff.

```csharp
using System;
using System.Collections.Generic;
using JsonDataImporter.Runtime.Core;
using Object = UnityEngine.Object;

public sealed class QuestTargetAdapter : IImportTargetAdapter
{
    public bool CanHandleTarget(Object targetAsset)
    {
        return targetAsset is QuestDatabase;
    }

    public string GetTargetDisplayName(Object targetAsset)
    {
        return targetAsset != null ? targetAsset.name : "<null target>";
    }

    public int GetTargetEntryCount(Object targetAsset)
    {
        return targetAsset is QuestDatabase database &&
               database.runtime?.quests != null
            ? database.runtime.quests.Count
            : 0;
    }

    public IReadOnlyList<object> GetTargetEntries(Object targetAsset)
    {
        if (targetAsset is not QuestDatabase database ||
            database.runtime?.quests == null)
        {
            return Array.Empty<object>();
        }

        var result = new List<object>(database.runtime.quests.Count);
        for (int i = 0; i < database.runtime.quests.Count; i++)
            result.Add(database.runtime.quests[i]);

        return result;
    }

    public object CreateTargetEntryFromSource(object sourceEntry)
    {
        return CloneQuest(sourceEntry);
    }

    public void SetTargetEntries(Object targetAsset, IReadOnlyList<object> finalEntries)
    {
        if (targetAsset is not QuestDatabase database)
            throw new InvalidOperationException("Quest adapter received invalid target.");

        database.runtime ??= new QuestRuntimeConfig();
        database.runtime.quests ??= new List<QuestRow>();
        database.runtime.quests.Clear();

        if (finalEntries == null)
            return;

        for (int i = 0; i < finalEntries.Count; i++)
            database.runtime.quests.Add(CloneQuest(finalEntries[i]));
    }

    private static QuestRow CloneQuest(object sourceEntry)
    {
        if (sourceEntry is not QuestRow quest)
            throw new InvalidOperationException("Quest adapter expected QuestRow.");

        return new QuestRow
        {
            id = quest.id,
            title = quest.title,
            reward = quest.reward,
            editorNote = quest.editorNote
        };
    }
}
```

`SetTargetEntries` is the only supported write point for the target asset. Parser, validator, diff engine, and apply strategy work with prepared data and diagnostics, but do not write to the Unity asset.

## Custom IImportProfile

Custom profile connects:

- stable `TargetTypeId`;
- user-facing `DisplayName`;
- `EntryType` produced by parser;
- target adapter;
- backup codec;
- default apply strategy.

```csharp
using System;
using JsonDataImporter.Runtime.Core;
using JsonDataImporter.Runtime.Core.Apply;
using JsonDataImporter.Runtime.Core.Metadata;

[ImportProfileMetadata("Quest Database [Custom Profile]", "Custom")]
public sealed class QuestImportProfile : IImportProfile
{
    public const string ProfileId = "quest_database_custom";

    private static readonly IImportApplyStrategy DefaultApplyStrategy =
        new ReplaceAllApplyStrategy();

    private static readonly IImportBackupCodec DefaultBackupCodec =
        new JsonEntryListBackupCodec();

    private readonly QuestTargetAdapter _targetAdapter = new();

    public string TargetTypeId => ProfileId;
    public string DisplayName => "Quest Database [Custom Profile]";
    public Type EntryType => typeof(QuestRow);
    public IImportTargetAdapter TargetAdapter => _targetAdapter;
    public IImportBackupCodec BackupCodec => DefaultBackupCodec;

    public IImportApplyStrategy GetDefaultApplyStrategy()
    {
        return DefaultApplyStrategy;
    }
}
```

Requirements:

- concrete `IImportProfile` with a public parameterless constructor;
- stable `TargetTypeId`, because config and incremental state store this id;
- `EntryType` matching parser `EntryType`;
- `TargetAdapter.CanHandleTarget(targetAsset)` returns `true` for the asset in config;
- collection-like target usually uses `ReplaceAllApplyStrategy`;
- one-document target usually uses `OneClassReplaceAllApplyStrategy`.

After compilation, select this profile in config or set `preferredTargetTypeId = QuestImportProfile.ProfileId`.

## Custom Validator

`IImportValidator` checks already parsed source entries and returns `ImportValidationMessage` diagnostics. It runs after parser diagnostics and reference lookup have been prepared, and before diff/apply decisions. A validator should not read source files, parse raw text, or write to the target asset; it only inspects `ImportValidationContext` and reports warnings or errors.

Custom validator is used when data is already parsed but project-specific rules are needed:

- required fields;
- value ranges;
- id uniqueness;
- references to other import targets;
- blocking errors before apply.

```csharp
using System.Collections.Generic;
using JsonDataImporter.Runtime.Core;
using JsonDataImporter.Runtime.Core.Metadata;
using JsonDataImporter.Runtime.Core.Validation;

[ImportValidatorMetadata(
    "quest_reward_validator",
    "Quest Reward Validator [Custom]",
    typeof(QuestRow))]
public sealed class QuestRewardValidator : IImportValidator
{
    public IReadOnlyList<ImportValidationMessage> Validate(
        ImportValidationContext context)
    {
        var result = new List<ImportValidationMessage>();
        IReadOnlyList<object> entries = context?.SourceEntries;
        if (entries == null)
            return result;

        for (int i = 0; i < entries.Count; i++)
        {
            if (entries[i] is not QuestRow quest)
                continue;

            if (string.IsNullOrWhiteSpace(quest.id))
            {
                result.Add(new ImportValidationMessage
                {
                    EntryKey = $"index:{i}",
                    Message = "Quest id is required.",
                    IsError = true
                });
            }

            if (quest.reward < 0)
            {
                result.Add(new ImportValidationMessage
                {
                    EntryKey = quest.id,
                    Message = "Quest reward cannot be negative.",
                    IsError = true
                });
            }
        }

        return result;
    }
}
```

Validator is selected in `selectedValidatorIds`. If error diagnostics exist and fail policy is enabled, `ValidateOnly` returns `ValidationFailed`, and `Apply` returns `ApplyBlocked`.

Requirements:

- concrete class implements `IImportValidator`;
- `[ImportValidatorMetadata(...)]` with stable validator id, display name, and entry type;
- metadata `entryType` matches the selected profile `EntryType`, or is `typeof(object)` for a general-purpose validator;
- `Validate(...)` reads from `ImportValidationContext.SourceEntries`, `TargetEntries`, `TypeDescriptor`, `IdentityStrategy`, and reference lookup dictionaries as needed;
- returned `ImportValidationMessage.IsError = true` marks a blocking error when fail policy is enabled.

Related validation documentation:

- Validator selection and default validators: [`selectedValidatorIds`](ImportSourceConfigReference.md#selectedvalidatorids).
- Built-in validation attributes: [Required Fields](ImportAttributes.md#required-fields), [References](ImportAttributes.md#references), and [`[ImportValidationIgnore]`](ImportAttributes.md#importvalidationignore).
- Validator discovery metadata: [Import Attributes - Extension Discovery Attributes](ImportAttributes.md#extension-discovery-attributes).
- Validation pipeline position and context: [Pipeline Stages And Extension Points](PipelineStagesAndExtensionPoints.md#preview-stages) and [Reference Lookup Runs Before Validation](PipelineStagesAndExtensionPoints.md#reference-lookup-runs-before-validation).
- Validation failure policy: [`failOnValidationErrors`](ImportSourceConfigReference.md#failonvalidationerrors).
- Statuses caused by validation errors: [ValidationFailed And ApplyBlocked](ImportJobStatusTroubleshooting.md#validationfailed-and-applyblocked).

## Custom Diff Engine

`IImportDiffEngine` builds the preview change list from prepared source entries and current target entries. It receives `ImportDiffContext` with source entries, target entries, target asset, profile, entry type, type descriptor, and identity strategy. A diff engine should not read source data, validate source correctness, write to the target asset, or decide apply behavior; it only describes what changed for preview/reporting.

Custom diff engine is used when standard comparison produces an inconvenient preview:

- changes are grouped by domain rules;
- noisy fields are hidden;
- special details are shown;
- diff diagnostics are returned.

```csharp
using System.Collections.Generic;
using JsonDataImporter.Runtime.Core;
using JsonDataImporter.Runtime.Core.Diff;
using JsonDataImporter.Runtime.Core.Identity;
using JsonDataImporter.Runtime.Core.Metadata;

[ImportDiffEngineMetadata(
    "quest_title_diff",
    "Quest Title Diff [Custom]",
    typeof(QuestRow),
    "Custom")]
public sealed class QuestTitleDiffEngine : IImportDiffEngine
{
    public List<ImportEntryChange> BuildChanges(ImportDiffContext context)
    {
        var result = new List<ImportEntryChange>();
        var targetById = new Dictionary<string, QuestRow>();

        AddTargets(context, targetById);

        IReadOnlyList<object> sourceEntries = context?.SourceEntries;
        if (sourceEntries == null)
            return result;

        for (int i = 0; i < sourceEntries.Count; i++)
        {
            if (sourceEntries[i] is not QuestRow source)
                continue;

            string key = GetKey(source, context.TypeDescriptor, context.IdentityStrategy, i);
            if (!targetById.TryGetValue(key, out QuestRow target))
            {
                result.Add(new ImportEntryChange
                {
                    EntryKey = key,
                    ChangeType = ImportEntryChangeType.Added,
                    Details = "Quest added."
                });
                continue;
            }

            if (source.title != target.title)
            {
                result.Add(new ImportEntryChange
                {
                    EntryKey = key,
                    ChangeType = ImportEntryChangeType.Updated,
                    Details = "Quest title changed.",
                    FieldChanges = new List<ImportFieldChange>
                    {
                        new()
                        {
                            FieldName = "title",
                            OldValue = target.title,
                            NewValue = source.title
                        }
                    }
                });
            }
            else
            {
                result.Add(new ImportEntryChange
                {
                    EntryKey = key,
                    ChangeType = ImportEntryChangeType.Unchanged,
                    Details = "Quest title unchanged."
                });
            }
        }

        return result;
    }

    private static void AddTargets(
        ImportDiffContext context,
        Dictionary<string, QuestRow> targetById)
    {
        IReadOnlyList<object> targetEntries = context?.TargetEntries;
        if (targetEntries == null)
            return;

        for (int i = 0; i < targetEntries.Count; i++)
        {
            if (targetEntries[i] is not QuestRow target)
                continue;

            string key = GetKey(target, context.TypeDescriptor, context.IdentityStrategy, i);
            if (!string.IsNullOrWhiteSpace(key))
                targetById[key] = target;
        }
    }

    private static string GetKey(
        QuestRow quest,
        ImportTypeDescriptor descriptor,
        IIdentityStrategy identityStrategy,
        int index)
    {
        return identityStrategy?.GetIdentity(
                   quest,
                   descriptor,
                   new ImportIdentityContext { Index = index })
               ?? quest.id
               ?? string.Empty;
    }
}
```

Diff failures are isolated by the pipeline: if diff engine throws an exception, preview does not fail completely, but gets a warning diagnostic and empty/partial diff result.

For controlled diagnostics, use `IImportDiffEngineWithResult`, which returns `ImportDiffResult`.

Requirements:

- concrete class implements `IImportDiffEngine`;
- optional `IImportDiffEngineWithResult` when the engine needs to return `ImportDiffDiagnostic` entries with the changes;
- `[ImportDiffEngineMetadata(...)]` with stable diff engine id, display name, entry type, and optional category;
- metadata `entryType` matches the selected profile `EntryType`, or is `typeof(object)` for a general-purpose engine;
- discovery must be able to create the engine through its supported constructor path;
- returned `ImportEntryChange.ChangeType` should use `Added`, `Updated`, `Removed`, or `Unchanged` consistently with preview/report counts.

Related diff engine documentation:

- Diff engine selection and default engine: [`selectedDiffEngineId`](ImportSourceConfigReference.md#selecteddiffengineid).
- Field-level diff controls: [Normalization And Comparison](ImportAttributes.md#normalization-and-comparison) and [Excluding Members From Pipeline Stages](ImportAttributes.md#excluding-members-from-pipeline-stages).
- Diff engine discovery metadata: [Import Attributes - Extension Discovery Attributes](ImportAttributes.md#extension-discovery-attributes).
- Pipeline position and isolated failures: [Pipeline Stages And Extension Points](PipelineStagesAndExtensionPoints.md#preview-stages) and [Diff Failures Are Isolated](PipelineStagesAndExtensionPoints.md#diff-failures-are-isolated).
- Runtime preview flow: [ImportJobCorePipeline.BuildPreview](ImportExecutionFlow.md#5-importjobcorepipelinebuildpreview).
- Diff counters and diagnostics in results: [ImportJob Status Troubleshooting](ImportJobStatusTroubleshooting.md#how-to-read-importjobresult).

## Custom Fingerprint Builder

`IImportFingerprintBuilder` computes a deterministic fingerprint for an entry list. `ImportJobRunner` calls it for prepared source entries during preview/apply and for target entries read through the selected target adapter. The resulting source and target fingerprints are stored in result diagnostics and, after successful apply, in `ImportSourceConfig.lastApplied*` state for incremental skip.

The builder receives an `ImportTypeDescriptor`, entries, and an optional `IIdentityStrategy`. It should not read raw source data, write to the target asset, build diff changes, or decide whether apply should run; it only defines stable change-detection input.

Custom fingerprint builder is used when incremental skip needs a custom data set:

- editor-only fields are ignored;
- entries are sorted differently;
- numbers, dates, or whitespace are normalized;
- fingerprint is stabilized for data where order does not matter.

```csharp
using System.Collections.Generic;
using JsonDataImporter.Runtime.Core.Fingerprinting;
using JsonDataImporter.Runtime.Core.Identity;
using JsonDataImporter.Runtime.Core.Metadata;
using JsonDataImporter.Runtime.Utils;

[ImportFingerprintBuilderMetadata(
    "quest_stable_fingerprint",
    "Quest Stable Fingerprint [Custom]",
    typeof(QuestRow),
    "Custom")]
public sealed class QuestStableFingerprintBuilder : IImportFingerprintBuilder
{
    public bool TryBuild(
        ImportTypeDescriptor descriptor,
        IReadOnlyList<object> entries,
        out string fingerprint,
        IIdentityStrategy identityStrategy = null)
    {
        fingerprint = string.Empty;
        if (entries == null)
            return false;

        var parts = new List<string>();
        for (int i = 0; i < entries.Count; i++)
        {
            if (entries[i] is not QuestRow quest)
                continue;

            string key = identityStrategy?.GetIdentity(
                             quest,
                             descriptor,
                             new ImportIdentityContext { Index = i })
                         ?? quest.id
                         ?? string.Empty;

            parts.Add($"{key}|{quest.title?.Trim()}|{quest.reward}");
        }

        parts.Sort();
        fingerprint = SourceFingerprintUtility.ComputeSha256(string.Join("\n", parts));
        return true;
    }
}
```

This example intentionally does not include `editorNote`, so changing the editor-only note will not trigger a new incremental apply.

`IImportFingerprintBuilder.TryBuild(...)` is deterministic: identical entries produce identical fingerprints on different machines and in different Unity runs.

Requirements:

- concrete class implements `IImportFingerprintBuilder`;
- `[ImportFingerprintBuilderMetadata(...)]` with stable fingerprint builder id, display name, entry type, and optional category;
- metadata `entryType` matches the selected profile `EntryType`, or is `typeof(object)` for a general-purpose builder;
- discovery must be able to create the builder through its supported constructor path;
- `TryBuild(...)` returns `true` only when a reliable fingerprint was computed;
- the fingerprint string is deterministic and stable across editor sessions, machines, item ordering rules, and culture settings;
- fields intentionally excluded from the fingerprint must be safe to ignore for incremental skip.

Related fingerprint documentation:

- Fingerprint builder selection and default builder: [`selectedFingerprintBuilderId`](ImportSourceConfigReference.md#selectedfingerprintbuilderid).
- Fingerprinting policy switches: [`enableFingerprinting`](ImportSourceConfigReference.md#enablefingerprinting) and [`enableIncrementalSkip`](ImportSourceConfigReference.md#enableincrementalskip).
- Incremental state and skip conditions: [Backup / Restore / Incremental State](BackupRestoreIncrementalState.md#lastapplied-state), [When Incremental Skip Works](BackupRestoreIncrementalState.md#when-incremental-skip-works), and [When Incremental Skip Does Not Work](BackupRestoreIncrementalState.md#when-incremental-skip-does-not-work).
- Field-level fingerprint controls: [Normalization And Comparison](ImportAttributes.md#normalization-and-comparison) and [`[ImportFingerprintIgnore]`](ImportAttributes.md#importfingerprintignore).
- Fingerprint builder discovery metadata: [Import Attributes - Extension Discovery Attributes](ImportAttributes.md#extension-discovery-attributes).
- Pipeline position and runtime flow: [Pipeline Stages And Extension Points](PipelineStagesAndExtensionPoints.md#preview-stages), [BuildPreviewCoreAsync](ImportExecutionFlow.md#4-buildpreviewcoreasync), and [ApplyPreparedCore](ImportExecutionFlow.md#9-applypreparedcore).
- CI overrides: [CI Commands - Incremental Skip And Fingerprinting](CICommands.md#incremental-skip-and-fingerprinting).

## Custom Apply Strategy

`IImportApplyStrategy` builds the final entries that will be written to the target asset. It receives prepared source entries, current target entries, target adapter, type descriptor, and identity strategy through `ImportApplyContext`. The strategy does not write to the asset directly; after `BuildFinalEntries(...)` returns, the selected target adapter writes the final list through `SetTargetEntries(...)`.

Custom apply strategy is used when built-in replace/merge/patch/append behavior does not match project rules:

- target entries need domain-specific merge rules;
- some editor-only values must be preserved while imported values are refreshed;
- source entries should be filtered or grouped before writing;
- collection order or duplicate handling differs from the default strategies.

Place the registration class in an Editor folder or an Editor-only assembly. Do not include UnityEditor references in a runtime assembly.

```csharp
using System.Collections.Generic;
using JsonDataImporter.Runtime.Core.Apply;
using JsonDataImporter.Runtime.Core.Identity;
using UnityEditor;

public sealed class QuestMergeKeepNotesApplyStrategy : IImportApplyStrategy
{
    public const string Id = "quest_merge_keep_notes";

    public string StrategyId => Id;
    public string DisplayName => "Quest Merge Keep Notes [Custom]";

    public List<object> BuildFinalEntries(ImportApplyContext context)
    {
        var result = new List<object>();
        if (context?.Adapter == null ||
            context.TypeDescriptor == null ||
            context.IdentityStrategy == null ||
            context.SourceEntries == null)
        {
            return result;
        }

        Dictionary<string, QuestRow> targetById = BuildTargetIndex(context);

        for (int i = 0; i < context.SourceEntries.Count; i++)
        {
            object sourceEntry = context.SourceEntries[i];
            if (sourceEntry == null)
                continue;

            object finalEntry = context.Adapter.CreateTargetEntryFromSource(sourceEntry);
            if (finalEntry is QuestRow quest)
            {
                string key = GetKey(quest, context, i);
                if (targetById.TryGetValue(key, out QuestRow previous))
                    quest.editorNote = previous.editorNote;
            }

            result.Add(finalEntry);
        }

        return result;
    }

    private static Dictionary<string, QuestRow> BuildTargetIndex(ImportApplyContext context)
    {
        var result = new Dictionary<string, QuestRow>();
        IReadOnlyList<object> entries = context.CurrentTargetEntries;
        if (entries == null)
            return result;

        for (int i = 0; i < entries.Count; i++)
        {
            if (entries[i] is not QuestRow quest)
                continue;

            string key = GetKey(quest, context, i);
            if (!string.IsNullOrWhiteSpace(key))
                result[key] = quest;
        }

        return result;
    }

    private static string GetKey(QuestRow quest, ImportApplyContext context, int index)
    {
        return context.IdentityStrategy.GetIdentity(
                   quest,
                   context.TypeDescriptor,
                   new ImportIdentityContext { Index = index })
               ?? quest.id
               ?? string.Empty;
    }
}

[InitializeOnLoad]
public static class QuestApplyStrategyRegistration
{
    static QuestApplyStrategyRegistration()
    {
        ImportApplyStrategyRegistry.Register(new QuestMergeKeepNotesApplyStrategy());
    }
}
```

Requirements:

- concrete class implements `IImportApplyStrategy`;
- `StrategyId` is stable because configs and `lastAppliedApplyStrategyId` store it;
- strategy is registered in `ImportApplyStrategyRegistry`;
- `BuildFinalEntries(...)` returns the complete final entry list for the target adapter;
- use `context.Adapter.CreateTargetEntryFromSource(...)` when source entries must be converted/cloned into target entries;
- errors thrown by an apply strategy are not isolated like diff errors and can finish apply as `Failed`.

Related apply strategy documentation:

- Apply strategy selection and built-in ids: [`selectedApplyStrategyId`](ImportSourceConfigReference.md#selectedapplystrategyid).
- Profile default strategy: [Custom `IImportProfile`](CustomImportExtensions.md#custom-iimportprofile).
- Apply stage order and target write boundary: [Pipeline Stages And Extension Points - Apply Stages](PipelineStagesAndExtensionPoints.md#apply-stages).
- Prepared source invariant: [Apply Uses Only Prepared Source](PipelineStagesAndExtensionPoints.md#apply-uses-only-prepared-source).
- Runtime apply flow: [Apply](ImportExecutionFlow.md#8-apply) and [ApplyPreparedCore](ImportExecutionFlow.md#9-applypreparedcore).
- Incremental skip uses strategy id: [Backup / Restore / Incremental State - lastApplied* State](BackupRestoreIncrementalState.md#lastapplied-state).
- Missing strategy and runtime failures: [ImportJob Status Troubleshooting - Failed](ImportJobStatusTroubleshooting.md#failed).

## How To Connect A Custom Chain

After compilation:

1. In `ImportSourceConfig`, assign `targetAsset`.
2. In source provider, select the custom provider if source is external.
3. In import profile, select the custom profile or set `preferredTargetTypeId`.
4. In `Parser`, select the custom parser if auto parser does not fit.
5. In `Validators`, add custom validators.
6. In `Diff Engine`, select custom diff engine if custom preview changes are needed.
7. In `Fingerprint Builder`, select custom fingerprint builder if incremental behavior differs from the canonical default.
8. In `Apply Strategy`, select custom apply strategy if final entry behavior differs from built-in replace/merge/patch/append.
9. Press `Preview` and check `Messages`, validation diagnostics, diff diagnostics, and fingerprints.

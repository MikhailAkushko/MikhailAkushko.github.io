# Custom Import Extensions

This reference describes supported extension points used when the auto path does not fit.

The auto path covers simple cases: `TextAsset` with JSON, entry type matches JSON shape, and target asset can be described through `[ImportTargetCollection]`, `[ImportTargetDocument]`, or direct fields. If one of these does not fit, extension points for specific pipeline stages are used. The full stage map is in [Pipeline Stages And Extension Points](PipelineStagesAndExtensionPoints.md).

## Quick Pick

| Problem | Extension point | Where it is selected |
| --- | --- | --- |
| Data does not come from Unity `TextAsset`. | `IImportSourceProvider` | `Source Provider` in `ImportSourceConfig`. |
| Source has a custom format or shape. | `IImportParser` | `selectedParserId` / Parser dropdown. |
| target asset cannot be read/written by an auto adapter. | `IImportTargetAdapter` | Through custom `IImportProfile`. |
| Target type, entry type, adapter, backup, and default apply strategy are defined together. | `IImportProfile` | Profile resolution / `preferredTargetTypeId`. |
| Project-specific checks are needed. | `IImportValidator` | `selectedValidatorIds`. |
| Preview changes are displayed by domain rules. | `IImportDiffEngine` or `IImportDiffEngineWithResult` | `selectedDiffEngineId`. |
| Incremental skip uses a custom data set. | `IImportFingerprintBuilder` | `selectedFingerprintBuilderId` and execution profile. |

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

Custom source provider is used when data lives in an external system: HTTP endpoint, Google Sheets, local file outside Unity asset database, generated content, internal tool API.

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

Custom parser is used when source cannot be read directly by the auto JSON parser:

- CSV, TSV, YAML, or custom text format;
- JSON root does not match `entryType`;
- field names, enum values, dates, or object-map-to-list conversion is required;
- the difference between a missing field and explicit `null` must be preserved;
- user-facing parser diagnostics are needed.

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

## Custom Diff Engine

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

## Custom Fingerprint Builder

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

## How To Connect A Custom Chain

After compilation:

1. In `ImportSourceConfig`, assign `targetAsset`.
2. In source provider, select the custom provider if source is external.
3. In import profile, select the custom profile or set `preferredTargetTypeId`.
4. In `Parser`, select the custom parser if auto parser does not fit.
5. In `Validators`, add custom validators.
6. In `Diff Engine`, select custom diff engine if custom preview changes are needed.
7. In `Fingerprint Builder`, select custom fingerprint builder if incremental behavior differs from the canonical default.
8. Press `Preview` and check `Messages`, validation diagnostics, diff diagnostics, and fingerprints.

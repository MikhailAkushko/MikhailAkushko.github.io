# Getting Started

This document describes the basic path for wiring a new import target using `AutoDocumentSettingsDatabase` as an example. It also explains where the system works automatically and when custom parsers/profiles/adapters are used.

## 1. Describe The Data

First, you need a serializable entry type. This is the type of one imported record or one document.

```csharp
using System;

namespace Game.Import
{
    [Serializable]
    public sealed class AutoDocumentSettings
    {
        public string title;
        public int maxLevel;
        public bool isEnabled;
        public float gravity;
        public AutoDocumentLimits limits;
    }

    [Serializable]
    public sealed class AutoDocumentLimits
    {
        public int minAmount;
        public int maxAmount;
    }
}
```

For the auto JSON path, public field names match JSON field names. For example:

```json
{
  "title": "Auto Document Settings",
  "maxLevel": 12,
  "isEnabled": true,
  "gravity": 9.81,
  "limits": {
    "minAmount": 1,
    "maxAmount": 99
  }
}
```

If JSON does not match the entry type shape, see [When A Custom Parser Is Needed](#5-when-a-custom-parser-is-needed).

## 2. Create The Target Asset Class

This is the target asset: a `ScriptableObject` that the importer applies data to.

For a single-document scenario, use [`[ImportTargetMetadata]`](ImportAttributes.md) on the class and [`[ImportTargetDocument]`](ImportAttributes.md) on the field or property where the document is stored:

```csharp
using JsonDataImporter.Runtime.Core.Metadata;
using UnityEngine;

namespace Game.Import
{
    [ImportTargetMetadata(
        "auto_document_settings_database",
        "Auto Document Settings Database",
        typeof(AutoDocumentSettings))]
    [CreateAssetMenu(
        fileName = "AutoDocumentSettingsDatabase",
        menuName = "Game/Import/Auto Document Settings Database")]
    public sealed class AutoDocumentSettingsDatabase : ScriptableObject
    {
        [ImportTargetDocument]
        public AutoDocumentSettings settings;
    }
}
```

Important details:

- `targetTypeId` (`"auto_document_settings_database"`) is stable: it is stored in the config and used for auto profile/parser ids.
- `displayName` (`"Auto Document Settings Database"`) is shown in the editor UI.
- `entryType` (`typeof(AutoDocumentSettings)`) tells the pipeline which type to parse, validate, diff, and apply.
- `[ImportTargetDocument]` tells the auto adapter that the target stores one document, not a list of records.

After compilation, create the asset through the menu specified in `CreateAssetMenu`.

## 3. Create ImportSourceConfig

Create the config asset:

```text
Create > Json Data Importer > Import Source Config
```

The minimal happy path looks like this:

```text
targetAsset: AutoDocumentSettingsDatabase.asset
source provider: Text Asset
Text Asset: auto_document_settings.json
import profile: Auto Document Settings Database [Auto Document]
Parser: Auto Document Settings Database [Auto JSON Document]
apply strategy: Replace All [One Class]
```

To get this state, fill only the basic fields:

- `targetAsset` - the created `AutoDocumentSettingsDatabase.asset`.
- source provider - usually `Text Asset` when JSON is stored in a Unity `TextAsset`.
- `Text Asset` inside the source provider - the JSON file with data.

Then press `Auto` for pipeline selection. For a document target, the editor usually selects the auto document profile, auto JSON parser, compatible validators, diff/fingerprint builder, and `Replace All (One Class)`.

For the first run, `Backup History` and `Execution Profile` can be skipped; they are needed when setting up safe apply and project policy. The full field list and policy override order are described in [ImportSourceConfig Reference](ImportSourceConfigReference.md).

After that, you can press:

- `Preview` - read source, parse entries, build validation/diff.
- `Validate` - run the same preview path and return validation status.
- `Apply` - run the preview path again, then apply prepared source to the target asset.

## 4. What Happens Automatically

The auto path works when the target asset has a supported shape.

The system can automatically:

- find the target asset class by `[ImportTargetMetadata]`;
- create an auto profile and target adapter for `[ImportTargetCollection]`;
- create an auto profile and target adapter for `[ImportTargetDocument]`;
- create an auto direct-fields adapter when target public fields/properties match the entry type;
- create an auto JSON parser with id `auto_json::<targetTypeId>`;
- choose a compatible parser/profile in the editor through the `Auto` button;
- build preview, validation, diff, fingerprints, and incremental skip through `ImportJobRunner`.

Supported auto target shapes:

```csharp
// Collection: JSON root array or object with an array field named items.
[ImportTargetCollection]
public List<ItemRecord> items = new();

// Document: JSON root object.
[ImportTargetDocument]
public AutoDocumentSettings settings;

// Direct fields: no [ImportTargetDocument]/[ImportTargetCollection],
// when target public fields/properties match entry type by name/type.
public string title;
public int maxLevel;
```

The auto JSON document parser expects a JSON root object and parses it into `entryType` through `JsonUtility`. The auto collection parser expects a root array or an object with an array field whose name matches the `[ImportTargetCollection]` member name.

## 5. When A Custom Parser Is Needed

A custom parser is needed when source data cannot be parsed directly by the auto JSON parser into your entry type.

Common reasons:

- JSON has a different structure than the entry type.
- Field names, types, enum/string values, or dates need conversion.
- JSON contains a dictionary/object-map that needs to become a list.
- The difference between a missing field and explicit `null` needs to be preserved.
- Source is not JSON at all or comes in a custom format.

Custom parser shape:

```csharp
using System;
using System.Collections.Generic;
using JsonDataImporter.Runtime.Core.Metadata;
using JsonDataImporter.Runtime.Core.Source;

[ImportParserMetadata(
    "hard_json_station_parser",
    "Hard JSON Station Parser [Custom]",
    typeof(SpaceStation),
    "Custom")]
public sealed class HardJsonStationParser : IImportParser<SpaceStation>
{
    public Type EntryType => typeof(SpaceStation);

    public IReadOnlyList<SpaceStation> ParseTypedEntries(string rawContent)
    {
        // Parse any source shape here and return normalized entries.
        return new[] { ParseStation(rawContent) };
    }

    public IReadOnlyList<object> ParseEntries(string rawContent)
    {
        IReadOnlyList<SpaceStation> typed = ParseTypedEntries(rawContent);
        var result = new List<object>(typed.Count);
        for (int i = 0; i < typed.Count; i++)
            result.Add(typed[i]);
        return result;
    }

    private static SpaceStation ParseStation(string rawContent)
    {
        // Put custom JSON/file conversion here.
        throw new NotImplementedException();
    }
}
```

After compilation, the parser appears in the Parser dropdown when its `entryType` matches the entry type of the selected target/profile. In the config, select this parser instead of the auto JSON parser.

## 6. When A Custom Target Adapter/Profile Is Needed

A custom adapter is needed when the target asset cannot be described by one of the auto shapes:

- there is no [`[ImportTargetCollection]`](ImportAttributes.md) or [`[ImportTargetDocument]`](ImportAttributes.md);
- data is stored deep inside the asset internal structure;
- apply uses special clone/merge/normalization logic;
- the target asset stores a different type than the parser returns;
- custom identity/apply/backup logic is required.

In this case, the usual pieces are:

- `IImportTargetAdapter` - how to read current entries from the target and how to write final entries back;
- `IImportProfile` - connects `TargetTypeId`, `DisplayName`, `EntryType`, adapter, backup codec, and default apply strategy;
- custom parser, if source shape is also custom.

Simplified adapter shape:

```csharp
using System;
using System.Collections.Generic;
using JsonDataImporter.Runtime.Core;
using UnityEngine;
using Object = UnityEngine.Object;

public sealed class HardJsonStationTargetAdapter : IImportTargetAdapter
{
    public bool CanHandleTarget(Object targetAsset)
    {
        return targetAsset is HardJsonStationDatabase;
    }

    public string GetTargetDisplayName(Object targetAsset)
    {
        return targetAsset != null ? targetAsset.name : "<null target>";
    }

    public int GetTargetEntryCount(Object targetAsset)
    {
        return targetAsset is HardJsonStationDatabase db && db.station != null ? 1 : 0;
    }

    public IReadOnlyList<object> GetTargetEntries(Object targetAsset)
    {
        if (targetAsset is not HardJsonStationDatabase db || db.station == null)
            return Array.Empty<object>();

        return new object[] { db.station };
    }

    public object CreateTargetEntryFromSource(object sourceEntry)
    {
        return CloneStation(sourceEntry);
    }

    public void SetTargetEntries(Object targetAsset, IReadOnlyList<object> finalEntries)
    {
        if (targetAsset is not HardJsonStationDatabase db)
            throw new InvalidOperationException("Invalid target.");

        db.station = finalEntries != null && finalEntries.Count > 0
            ? CloneStation(finalEntries[0])
            : null;
    }

    private static SpaceStation CloneStation(object sourceEntry)
    {
        if (sourceEntry is not SpaceStation station)
            throw new InvalidOperationException("Expected SpaceStation entry.");

        // Deep clone fields that should not share references with source entries.
        return station;
    }
}
```

Simplified profile shape:

```csharp
using System;
using JsonDataImporter.Runtime.Core;
using JsonDataImporter.Runtime.Core.Apply;
using JsonDataImporter.Runtime.Core.Metadata;

[ImportProfileMetadata("Hard JSON Station Profile [Custom]", "Custom")]
public sealed class HardJsonStationImportProfile : IImportProfile
{
    private readonly HardJsonStationTargetAdapter _targetAdapter = new();

    public string TargetTypeId => "hard_json_station_profile";
    public string DisplayName => "Hard JSON Station Profile [Custom]";
    public Type EntryType => typeof(SpaceStation);
    public IImportTargetAdapter TargetAdapter => _targetAdapter;
    public IImportBackupCodec BackupCodec { get; } = new JsonEntryListBackupCodec();

    public IImportApplyStrategy GetDefaultApplyStrategy()
    {
        return new OneClassReplaceAllApplyStrategy();
    }
}
```

After compilation, the profile is found by the discovery system if it is a concrete `IImportProfile` with a public parameterless constructor. In `ImportSourceConfig`, select this profile in the import profile field.

## 7. Choosing Auto Or Custom

Use the auto path when:

- target can be described by `[ImportTargetCollection]`, `[ImportTargetDocument]`, or direct fields;
- JSON shape matches the public fields of the entry type;
- standard validation/diff/fingerprint/apply steps are enough.

A custom parser is needed when:

- source shape is more complex than entry type;
- conversion is needed before data enters the pipeline;
- auto JSON parser reports unmapped field diagnostics or cannot find the required root object/array.

A custom adapter/profile is needed when:

- target asset cannot be safely read/written by the standard auto adapter;
- clone, apply shape, or backup entries need control;
- one source entry must be written into a custom location in the target asset.

A custom source provider is needed when:

- data is not stored in a `TextAsset`;
- the source is HTTP, Google Sheets, a local file outside the Unity asset database, generated content, or another external service.

Execution path details are described in [Import Execution Flow](ImportExecutionFlow.md).

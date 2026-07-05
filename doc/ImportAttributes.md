# Import Attributes

This document describes import attributes from `JsonDataImporter.Runtime.Core.Metadata`.
The order goes from the most important attributes that define the auto import shape to narrower settings for individual pipeline stages.

## Target Shape Attributes

### `[ImportTargetMetadata(targetTypeId, displayName, entryType)]`

Applied to a target asset class, usually a `ScriptableObject`.

- `targetTypeId` - stable target type id. It is used by the discovery layer, configs, auto profile, and auto parser id.
- `displayName` - name for inspector/debug UI.
- `entryType` - type of one imported record.

This attribute makes the asset visible to auto import discovery. By itself it does not select a data source, parser, diff engine, or apply strategy. For auto list/document/direct-fields import, a supported target shape must also exist.

Practical requirements:

- `targetTypeId` must be non-empty and without accidental leading/trailing whitespace;
- `entryType` must be a concrete record type and usually needs `[Serializable]` for auto JSON;
- if a target has no `[ImportTargetMetadata]`, auto profile/parser discovery will not pick it up, but a custom `IImportProfile` can still handle the asset manually.

### `[ImportTargetCollection]`

Applied to a target asset field or property that stores a list of records.

```csharp
[ImportTargetCollection]
public List<PlainItemRecord> items = new();
```

Used by the auto list import path:

- auto adapter reads current entries from this collection;
- auto JSON parser accepts either a root array or an object with an array field whose name matches the collection member name;
- apply writes the final list back into this same collection.

Supported shape: a non-array `IList`/`IList<T>`-compatible collection where element type exactly matches `entryType`. Initialize the collection ahead of time because the auto adapter works with the existing list instance.

Multiple target-shape attributes on one asset are not considered a supported configuration yet. Do not rely on a target that has several `[ImportTargetCollection]` members, several `[ImportTargetDocument]` members, or a combination of `[ImportTargetCollection]` and `[ImportTargetDocument]`. Current discovery logic can select one member and show a warning, but that is a compatibility fallback, not a stable contract.

### `[ImportTargetDocument]`

Applied to a target asset field or property that stores one document.

```csharp
[ImportTargetDocument]
public AutoDocumentSettings settings;
```

Used by the auto document import path:

- auto JSON parser expects a root object;
- pipeline treats the document as one entry;
- apply replaces the document member with the first final entry, or `null` if the final list is empty.

Supported shape: a field/property whose type exactly matches `entryType`. Static/readonly fields are not supported; properties must be readable and writable.

For the auto path, use only one target shape per asset: collection, document, or direct fields. A scenario where one `ScriptableObject` declares both collection and document members is not currently supported as a normal import target. For several independent views, the supported options are a custom `IImportProfile`/`IImportTargetAdapter` or several target assets.

## Identity And Matching

### `[ImportIdentity(order = 0)]`

Applied to an entry type field or property. Defines the record identity key for validation, diff, and apply matching.

```csharp
[Serializable]
public sealed class ItemRecord
{
    [ImportIdentity]
    public string itemId;
}
```

If identity has multiple parts, the pipeline sorts parts by `order` and joins values with `|`.

```csharp
[ImportIdentity(0)] public string table;
[ImportIdentity(1)] public string key;
```

For an entry with `table = "items"` and `key = "sword"`, the default identity is `items|sword`. If several identity members have the same `order`, their relative order should not be treated as a stable contract. If at least one identity part is `null` or an empty/whitespace string, default identity is not built.

If the entry type has no `[ImportIdentity]`, the default identity strategy tries to find a member named `id` case-insensitively. For some auto scenarios, the selector can choose a separate strategy: single-document identity for document/direct-fields import, or index identity when records have no data identity.

Practical rule: for list imports, explicitly set `[ImportIdentity]` or have a stable `id` field. Empty identity produces a validation error and prevents safe diff/apply matching.

## Required Fields

### `[ImportRequired]`

Applied to an entry type field or property. Built-in structure validator treats the field as required.

```csharp
[ImportRequired]
public string title;
```

A field is considered missing when, after reading the value, it is `null` or an empty/whitespace string. Required validation runs before apply and can block apply when the job is configured to fail on validation errors.

Non-nullable value types such as `int`, `float`, and `bool` always have a value. When using `JsonUtility`, `[ImportRequired]` cannot distinguish a missing JSON field from the type's default value.

### `[ImportRequiredAfterNormalization]`

Applied to an entry type field or property. Marks the field as required with the explicit meaning "check after normalizers".

```csharp
[ImportNormalizer(ImportNormalizerIds.Trim, ImportNormalizerIds.EmptyStringToNull)]
[ImportRequiredAfterNormalization]
public string code;
```

Used when a raw value can look non-empty but become missing after normalization. For example, `"   "` becomes `null` after `trim` and `empty_to_null`.

## References

### `[ImportReference(targetEntryType = null, AllowEmpty = true)]`

Applied to a string field/property that stores the identity of another record.

```csharp
[ImportReference(typeof(ItemRecord), AllowEmpty = false)]
public string itemId;
```

Example of a reference from an imported quest to an item stored in another import target:

```csharp
using System;
using JsonDataImporter.Runtime.Core.Metadata;

[Serializable]
public sealed class ItemRecord
{
    [ImportIdentity]
    public string id;

    public string title;
}

[Serializable]
public sealed class QuestRecord
{
    [ImportIdentity]
    public string questId;

    [ImportReference(typeof(ItemRecord), AllowEmpty = false)]
    public string rewardItemId;
}
```

In this example, `rewardItemId` must match the identity of an existing `ItemRecord`. Editor reference lookup mixes entries from another target into validation context when it can uniquely find a profile/asset for `ItemRecord`.

- `targetEntryType` points to the entry type referenced by the field. If it is not set, the reference is checked against the current entry type.
- `AllowEmpty` allows `null`, empty string, and whitespace. Default is `true`.

After normalizers, the built-in reference validator checks that the value is a string and that a non-empty reference exists in the identity set of the target type. For cross-target references, validation context must contain entries of the target `targetEntryType`; otherwise the reference is considered unsupported. How those entries are injected through `IImportReferenceLookupProvider` is shown in [Pipeline Stages And Extension Points](PipelineStagesAndExtensionPoints.md#reference-lookup-runs-before-validation).

## Normalization And Comparison

### `[ImportNormalizer(params string[] normalizerIds)]`

Applied to a field or property. Applies named normalizers before validation, diff, and canonical fingerprint.

```csharp
[ImportNormalizer(ImportNormalizerIds.Trim, ImportNormalizerIds.Lowercase)]
public string tag;
```

The attribute can be applied multiple times; all non-empty ids are added to the policy.

> Warning: unknown normalizer ids are currently skipped. A typo in a normalizer id may leave the value unnormalized without failing the import.

Built-in ids:

- `ImportNormalizerIds.Trim` / `"trim"` - `string.Trim()`;
- `ImportNormalizerIds.EmptyStringToNull` / `"empty_to_null"` - whitespace string becomes `null`;
- `ImportNormalizerIds.Lowercase` / `"lowercase"` - `ToLowerInvariant()`.

### `[ImportComparer(comparerId)]`

Applied to a field or property. Selects a named comparer for diff on this field.

```csharp
[ImportComparer(ImportComparerIds.FloatTolerance)]
public float weight;
```

Comparer affects only diff comparison. Validation, apply, and fingerprint do not change because of it. If the id is not present in `ImportComparerRegistry`, diff emits a diagnostic for the problematic comparison.

Useful built-in ids:

- `case_insensitive_string` / `string_ignore_case`;
- `float_tolerance`;
- `dictionary_structural`;
- `json_structural`;
- `datetime_same_instant_utc`;
- `datetime_exact_ticks`;
- `datetime_date_only`;
- `unordered_string_collection`;
- `unordered_string_set`;
- `unordered_string_collection_ignore_case`;
- `unordered_string_set_ignore_case`.

## Excluding Members From Pipeline Stages

### `[ImportIgnore]`

Applied to a field or property. Excludes the member from auto member mapping and default field-level diff.

```csharp
[ImportIgnore]
public string editorOnlyNote;
```

In the current implementation, the attribute definitely excludes the member from:

- auto direct-fields mapping;
- auto JSON unmapped-field diagnostics;
- default diff members;
- enum validation pass.

To explicitly exclude a field from fingerprint or built-in validation, use the stage-specific attributes below.

### `[ImportDiffIgnore]`

Applied to a field or property. Excludes the member only from diff.

```csharp
[ImportDiffIgnore]
public string lastEditedBy;
```

The field remains available to validation, fingerprint, and apply. Use this for values that can be imported or included in fingerprint but should not be shown as field-level diff.

### `[ImportFingerprintIgnore]`

Applied to a field or property. Excludes the member from canonical fingerprint.

```csharp
[ImportFingerprintIgnore]
public string generatedAt;
```

Useful for volatile values that do not participate in incremental skip. The field remains in diff and validation unless other ignore attributes are added.

### `[ImportValidationIgnore]`

Applied to a field or property. Excludes the member from built-in validation.

```csharp
[ImportValidationIgnore]
public LegacyEnum legacyState;
```

The field remains in diff and fingerprint. For references, this also disables built-in reference validation on this member.

## Extension Discovery Attributes

These attributes are applied not to data fields, but to pipeline extension classes. They let discovery and the inspector show and select custom components.

### `[ImportSourceProviderMetadata(providerId, displayName, category = "Default")]`

Applied to a concrete `IImportSourceProvider`.

```csharp
[ImportSourceProviderMetadata("http", "HTTP", "Built-in")]
public sealed class HttpImportSourceProvider : IImportSourceProvider
{
}
```

The provider is selected in `ImportSourceConfig.sourceProvider` as a managed reference. Discovery picks up the provider if the class is concrete and can be created from config: built-in source providers or a public parameterless constructor. See [Custom `IImportSourceProvider`](CustomImportExtensions.md#custom-iimportsourceprovider).

### `[ImportParserMetadata(parserId, displayName, entryType, category = "Default")]`

Applied to a concrete `IImportParser` with a public parameterless constructor.

```csharp
[ImportParserMetadata("items_json", "Items JSON Parser", typeof(ItemRecord), "Custom")]
public sealed class ItemJsonSourceParser : IImportParser
{
}
```

`parserId` is selected in config. `entryType` must match the target entry type; otherwise the parser is not treated as a compatible candidate. See [Custom `IImportParser`](CustomImportExtensions.md#custom-iimportparser).

### `[ImportProfileMetadata(displayName = null, category = null)]`

Applied to a concrete `IImportProfile`.

```csharp
[ImportProfileMetadata("Cards Profile", "Custom")]
public sealed class CardsImportProfile : IImportProfile
{
}
```

The attribute affects how the profile option is displayed in the inspector. Custom profile discovery itself depends on `IImportProfile`, concrete class, and public parameterless constructor; metadata does not replace required profile properties: `TargetTypeId`, `EntryType`, `TargetAdapter`. See [Custom `IImportProfile`](CustomImportExtensions.md#custom-iimportprofile).

### `[ImportValidatorMetadata(validatorId, displayName, entryType)]`

Applied to a concrete `IImportValidator`.

```csharp
[ImportValidatorMetadata("card_rules", "Card Business Rules", typeof(CardRecord))]
public sealed class CardRulesValidator : IImportValidator
{
}
```

Used by discovery and tooling/compatibility lists. `entryType` can be a concrete type or `typeof(object)` for a general-purpose validator. See [Custom Validator](CustomImportExtensions.md#custom-validator).

### `[ImportDiffEngineMetadata(diffEngineId, displayName, entryType, category = null)]`

Applied to a concrete `IImportDiffEngine`.

```csharp
[ImportDiffEngineMetadata("card_diff", "Card Diff", typeof(CardRecord), "Custom")]
public sealed class CardDiffEngine : IImportDiffEngine
{
}
```

`diffEngineId` is selected by the config/profile layer. Discovery creates the engine if the type is concrete and its constructor is supported by `DiscoveredComponentActivator`. See [Custom Diff Engine](CustomImportExtensions.md#custom-diff-engine).

### `[ImportFingerprintBuilderMetadata(fingerprintBuilderId, displayName, entryType, category = null)]`

Applied to a concrete `IImportFingerprintBuilder`.

```csharp
[ImportFingerprintBuilderMetadata("card_fingerprint", "Card Fingerprint", typeof(CardRecord), "Custom")]
public sealed class CardFingerprintBuilder : IImportFingerprintBuilder
{
}
```

Used to select a custom fingerprint builder. `fingerprintBuilderId` must be stable because it is stored in config and used during resolution. See [Custom Fingerprint Builder](CustomImportExtensions.md#custom-fingerprint-builder).

## Attribute Quick Pick

- Make an asset an auto import target: `[ImportTargetMetadata]` + `[ImportTargetCollection]` or `[ImportTargetDocument]`.
- Stable matching behavior for list import: `[ImportIdentity]`.
- Block empty data: `[ImportRequired]` or `[ImportRequiredAfterNormalization]`.
- Check a reference to another record: `[ImportReference]`.
- Normalize values before validation and diff: `[ImportNormalizer]`.
- Compare a field with custom semantics: `[ImportComparer]`.
- Exclude a member from automatic mapping and default diff: `[ImportIgnore]`.
- Exclude a field from a specific stage: `[ImportDiffIgnore]`, `[ImportFingerprintIgnore]`, `[ImportValidationIgnore]`.
- Register a custom source/parser/validator/diff/fingerprint/profile: a metadata attribute from the discovery section.

# Editor Display Names And Badges

This note describes how `ImportSourceConfigEditor` shows selected profiles, parsers, validators, diff/fingerprint builders, and apply strategies in the inspector.

Main rule: the editor first receives a `DisplayName` string, then splits it into the main title and small badges. `TargetTypeId`, `ParserId`, `ValidatorId`, and other ids remain technical config values; they are used for resolution and are not used as the main user-facing name.

## Where DisplayName Comes From

For auto targets, the main name comes from the target attribute:

```csharp
[ImportTargetMetadata("items_database", "Items Database", typeof(ItemRecord))]
public sealed class ItemsDatabase : ScriptableObject
{
}
```

`"Items Database"` becomes `DiscoveredImportTarget.DisplayName`. The auto adapter then appends the adapter type:

```csharp
public string DisplayName => $"{_target.DisplayName} (Auto Direct Fields Adapter)";
```

For a direct-fields target, the final profile string usually looks like this:

```text
Items Database (Auto Direct Fields Adapter) (auto_direct_fields::items_database)
```

In the editor it is shown as:

```text
Items Database    [Auto Direct Fields]
```

The technical id `auto_direct_fields::items_database` is hidden from the user-facing text but remains the selected id in the config.

For a custom profile, display can be overridden with an attribute:

```csharp
[ImportProfileMetadata("Items Import [Custom]", "Gameplay")]
public sealed class ItemsImportProfile : IImportProfile
{
    public string TargetTypeId => "items_custom";
    public string DisplayName => "Items Import";
}
```

If `ImportProfileMetadata.DisplayName` is set, the inspector uses it. If it is empty, `IImportProfile.DisplayName` is used. `ImportProfileMetadata.Category` controls the dropdown menu group.

For parsers, diff engines, fingerprint builders, and source providers, names come from the corresponding metadata attributes:

```csharp
[ImportParserMetadata("items_json", "Items JSON Parser", typeof(ItemRecord), "Gameplay")]
[ImportDiffEngineMetadata("items_diff", "Items Diff Engine", typeof(ItemRecord), "Gameplay")]
[ImportFingerprintBuilderMetadata("items_fp", "Items Fingerprint Builder", typeof(ItemRecord), "Gameplay")]
[ImportSourceProviderMetadata("remote_items", "Remote Items", "Sources")]
```

For validators:

```csharp
[ImportValidatorMetadata("items_rules", "Items Rules", typeof(ItemRecord))]
```

The editor usually appends the id to the end of the menu item: `Items JSON Parser (items_json)`. When that id matches the selected `resolvedId`, it is treated as technical and is not turned into a badge.

## How Badges Are Built

The selected text is parsed by `BuildSelectionDisplayParts` in `ImportSourceConfigEditor`.

1. Text in square brackets `[...]` always becomes a badge and is removed from the title.
2. The last round-parenthesized suffixes `(...)` are parsed from right to left.
3. A round-parenthesized suffix is hidden as technical when it equals the selected id, contains `::`, or contains `_`.
4. Other round-parenthesized suffixes become badges.
5. Badge text loses trailing words `Adapter`, `Parser`, `Builder`, `Engine`.

Examples:

```text
Items Database (Auto Direct Fields Adapter)
=> title: Items Database
=> badge: Auto Direct Fields

Items JSON Parser (items_json)
=> title: Items JSON Parser
=> badge: none, because items_json is a technical id

Items Import [Custom] (items_custom)
=> title: Items Import
=> badge: Custom

Replace All (One Class) (one_class_replace_all)
=> title: Replace All
=> badge: One Class
```

## How To Control Display

For the main user-facing name, change `displayName` in the metadata attribute or `DisplayName` on the profile/adapter/strategy.

For short explicit statuses, use square brackets:

```csharp
[ImportProfileMetadata("Items Import [Beta]", "Gameplay")]
```

For the auto adapter type, use a human-readable suffix in round parentheses:

```csharp
public string DisplayName => $"{_target.DisplayName} (Auto Direct Fields Adapter)";
```

The editor shows it as the `Auto Direct Fields` badge. The words `Adapter`, `Parser`, `Builder`, and `Engine` are removed from the end of the badge automatically.

Technical ids are better kept technical: `items_json`, `auto_direct_fields::items_database`, `one_class_replace_all`. These suffixes do not clutter the UI because the editor hides them from titles and badges.

## Practical Rules

- `displayName` in an attribute is a user-facing name: `Items Database`, not `items_database`.
- Use `category` for the dropdown menu group, not as part of the name.
- For a status or purpose badge, add `[Custom]`, `[Beta]`, `[Experimental]`.
- For an adapter type badge, add a human-readable suffix such as `(Auto List Adapter)`, `(Auto Document Adapter)`, or `(Auto Direct Fields Adapter)`.
- The technical id is not added to the main name manually: the editor shows ids where they are needed for selection.

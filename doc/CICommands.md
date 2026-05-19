# CI Commands

This document describes command-line execution of JSON Data Importer from Unity batch mode: which commands to use in CI, what each command does, and which settings it overrides.

## Basic Template

On Windows CI, `Unity.com` is more convenient to run: it sends stdout/stderr to the console better. `Unity.exe` can also work, but it is usually less convenient for logs.

```powershell
$Unity = "C:\Program Files\Unity\Hub\Editor\6000.n\Editor\Unity.com"
$Project = "C:\projects\JsonDataImporter"

& $Unity -batchmode -nographics -quit `
    -projectPath $Project `
    -executeMethod Jdi.Cli.Run `
    -logFile -
```

Required Unity arguments:

- `-batchmode` - starts Unity without the interactive editor workflow.
- `-nographics` - disables graphics mode, useful for CI agents.
- `-quit` - closes Unity after the method finishes.
- `-projectPath` - path to the Unity project root.
- `-executeMethod Jdi.Cli.Run` - short namespaced entry point for the JDI CLI.
- `-logFile -` - writes Unity log to stdout; useful for CI logs.

`Jdi.Cli.Run` is a short wrapper over `JsonDataImporter.Editor.Batch.JsonDataImporterBatchCommands.RunFromCommandLine`. The short entry point is namespaced so the package does not add global types.

## Help

Show help:

```powershell
& $Unity -batchmode -nographics -quit -projectPath $Project -executeMethod Jdi.Cli.Help -logFile -
```

Alternatives:

```powershell
& $Unity -batchmode -nographics -quit -projectPath $Project -executeMethod Jdi.Cli.Run help -logFile -
& $Unity -batchmode -nographics -quit -projectPath $Project -executeMethod Jdi.Cli.Run -- --help -logFile -
```

Pass `--help` after `--`, because Unity can reserve `-help`/`--help` for itself. Simple aliases `help`, `?`, `/?`, `/help`, `-jdiHelp`, `--jdiHelp` are also supported.

## Recommended CI Path: Execution Config By Id

Run a batch execution config by stable id:

```powershell
& $Unity -batchmode -nographics -quit -projectPath $Project -executeMethod Jdi.Cli.Run -id DemoAllValid -logFile -
```

What the command does:

- finds an `ImportBatchExecutionConfig` asset with `id == DemoAllValid`;
- takes the `sources` list from that asset;
- takes `executionMode` from that asset;
- runs sources in the order defined in the asset;
- applies boolean overrides from `ImportBatchExecutionConfig`;
- returns a CI exit code based on job results.

This is the most stable CI option: the pipeline does not depend on a specific asset path when id is unique in the project.

For a set of CI scenarios, create several execution configs:

```powershell
& $Unity -batchmode -nographics -quit -projectPath $Project -executeMethod Jdi.Cli.Run -id AllValid -logFile -
& $Unity -batchmode -nographics -quit -projectPath $Project -executeMethod Jdi.Cli.Run -id MixedIndependent -logFile -
& $Unity -batchmode -nographics -quit -projectPath $Project -executeMethod Jdi.Cli.Run -id ValidationOverride -logFile -
```

These ids must exist in `ImportBatchExecutionConfig.id`. The current demo set has `DemoAllValid`.

## Execution Config By Asset Path

Run a specific `ImportBatchExecutionConfig` asset:

```powershell
& $Unity -batchmode -nographics -quit -projectPath $Project -executeMethod Jdi.Cli.Run `
    -ec "Assets/JsonDataImporter/Demo/Examples/BatchImport/Assets/AllValidBatchImportConfig.asset" `
    -logFile -
```

What the command does:

- loads the batch config by Unity asset path;
- runs the same rules as `-id`;
- useful when id is not stable yet or when a specific asset needs to be checked explicitly.

Asset path is passed as a Unity path like `Assets/...`, not as an absolute filesystem path.

## Single ImportSourceConfig

Validate-only run for one source config:

```powershell
& $Unity -batchmode -nographics -quit -projectPath $Project -executeMethod Jdi.Cli.Run `
    -c "Assets/JsonDataImporter/Demo/Examples/PlainItems/Assets/PlainItemsImportConfig.asset" `
    -logFile -
```

What the command does:

- loads one `ImportSourceConfig`;
- builds preview;
- runs validation;
- does not apply changes to the target asset;
- uses dependency ordering, although it usually does not matter for one config.

`-c` supports forms `-c <AssetPath>` and `-c=<AssetPath>`. Old alias `-importConfig <AssetPath>` is also supported.

### Preview

```powershell
& $Unity -batchmode -nographics -quit -projectPath $Project -executeMethod Jdi.Cli.Run `
    -c "Assets/JsonDataImporter/Demo/Examples/PlainItems/Assets/PlainItemsImportConfig.asset" `
    --preview `
    -logFile -
```

What the command does:

- builds preview;
- computes diff/statistics;
- does not turn validation errors into a failed validate result;
- does not run apply, backup, or last applied state update.

### Apply

```powershell
& $Unity -batchmode -nographics -quit -projectPath $Project -executeMethod Jdi.Cli.Run `
    -c "Assets/JsonDataImporter/Demo/Examples/PlainItems/Assets/PlainItemsImportConfig.asset" `
    --apply `
    -logFile -
```

What the command does:

- builds preview;
- runs validation;
- if validation does not block apply, applies changes to the target asset;
- when backup capture is enabled, creates backup before apply;
- updates fingerprints and last applied state;
- saves changed assets through `AssetDatabase.SaveAssets()`.

If both `--apply` and `--preview` are passed, `--apply` has priority.

## All ImportSourceConfig Assets

Validate-only run for all `ImportSourceConfig` assets in the project:

```powershell
& $Unity -batchmode -nographics -quit -projectPath $Project -executeMethod Jdi.Cli.Run -all -logFile -
```

Preview for all configs:

```powershell
& $Unity -batchmode -nographics -quit -projectPath $Project -executeMethod Jdi.Cli.Run -all --preview -logFile -
```

Apply for all configs:

```powershell
& $Unity -batchmode -nographics -quit -projectPath $Project -executeMethod Jdi.Cli.Run -all --apply -logFile -
```

What `-all` does:

- finds all assets of type `ImportSourceConfig`;
- sorts them by asset path;
- builds a dependency execution plan;
- runs each config through `ImportJobRunner`;
- blocks dependent configs when a dependency did not finish successfully.

If no `ImportSourceConfig` is found, the command exits with code `3`.

## Folder Filter

Run all source configs only under the specified folder:

```powershell
& $Unity -batchmode -nographics -quit -projectPath $Project -executeMethod Jdi.Cli.Run `
    -all `
    --folder="Assets/JsonDataImporter/Demo/Examples/PlainItems" `
    -logFile -
```

What `--folder` does:

- applies only to `-all`;
- filters found `ImportSourceConfig` assets by Unity asset path;
- expects a path like `Assets/...`.

`--folder` does not apply to `-id` and `-ec`, because execution config defines the sources list itself.

## Reports

Run with JSON and text reports:

```powershell
& $Unity -batchmode -nographics -quit -projectPath $Project -executeMethod Jdi.Cli.Run `
    -id DemoAllValid `
    --reportJson="Artifacts/import-report.json" `
    --reportText="Artifacts/import-report.txt" `
    -logFile -
```

What report flags do:

- `--reportJson` writes a structured report through `JsonUtility.ToJson(..., true)`;
- `--reportText` writes a human-readable summary;
- directory is created automatically if it does not exist;
- report contains batch summary and data for each job: status, config, profile, source, validation messages, diff diagnostics, counts, applied/skipped/assetsChanged.

For `-id`/`-ec`, CLI report paths override `reportJsonPath` and `reportTextPath` saved in `ImportBatchExecutionConfig`.

For `-c`/`-all`, report paths come only from CLI flags.

## Incremental Skip And Fingerprinting

Disable incremental skip for `-c` or `-all`:

```powershell
& $Unity -batchmode -nographics -quit -projectPath $Project -executeMethod Jdi.Cli.Run `
    -c "Assets/JsonDataImporter/Demo/Examples/PlainItems/Assets/PlainItemsImportConfig.asset" `
    --apply `
    --noIncremental `
    -logFile -
```

Disable fingerprinting for `-c` or `-all`:

```powershell
& $Unity -batchmode -nographics -quit -projectPath $Project -executeMethod Jdi.Cli.Run `
    -all `
    --noFingerprinting `
    -logFile -
```

By default, without `--noIncremental` or `--noFingerprinting`, CLI sets `true` instead of inheriting from profile; see the "What CLI Overrides" table for details.

For `-id` and `-ec`, these flags do not apply. To disable incremental skip or fingerprinting in an execution config run, configure `enableIncrementalSkip` or `enableFingerprinting` inside `ImportBatchExecutionConfig`.

## Execution Policy Priority

The general policy value order is described in [ImportSourceConfig Reference](ImportSourceConfigReference.md#executionprofile). This section records only CLI/batch-specific behavior.

`ImportBatchExecutionConfig` uses `ImportBoolOverride`:

- `Inherit` becomes `null` and allows the value to come from `ImportSourceConfig.executionProfile`;
- explicit override becomes `true` or `false` and has priority over the source execution profile.

There are no CLI flags for backup or fail-on-validation, so these settings are inherited from source execution profile and then default `true`.

## What CLI Overrides

For `-c` and `-all`:

| CLI | Effect |
| --- | --- |
| `--apply` | `ImportExecutionMode.Apply` |
| `--preview` | `ImportExecutionMode.PreviewOnly` |
| no mode flag | `ImportExecutionMode.ValidateOnly` |
| `--targetTypeId=<id>` | overrides preferred target type id |
| `--folder=<path>` | filters configs only for `-all` |
| `--reportJson=<path>` | writes JSON report |
| `--reportText=<path>` | writes text report |
| `--noIncremental` | `EnableIncrementalSkip = false` |
| no `--noIncremental` | `EnableIncrementalSkip = true` |
| `--noFingerprinting` | `EnableFingerprinting = false` |
| no `--noFingerprinting` | `EnableFingerprinting = true` |

For `-id` and `-ec`:

| CLI | Effect |
| --- | --- |
| `-id <id>` | selects `ImportBatchExecutionConfig` by id |
| `-ec <path>` | selects `ImportBatchExecutionConfig` by asset path |
| `--reportJson=<path>` | overrides `ImportBatchExecutionConfig.reportJsonPath` |
| `--reportText=<path>` | overrides `ImportBatchExecutionConfig.reportTextPath` |

Other flags do not apply to run options for `-id`/`-ec`. Mode, sources, order, and optional boolean overrides come from `ImportBatchExecutionConfig`.

## Exit Codes

| Code | Meaning |
| --- | --- |
| `0` | all jobs completed successfully |
| `2` | validation failed, or apply was blocked by validation/diff errors |
| `3` | batch configuration/runtime error, missing assets, dependency blocked, lock blocked, or other failure |

Batch command computes the highest exit code among jobs. If at least one job has a non-zero code, Unity process exits with that code.

## Legacy Command

The old full entry point still works:

```powershell
& $Unity -batchmode -nographics -quit -projectPath $Project `
    -executeMethod JsonDataImporter.Editor.Batch.JsonDataImporterBatchCommands.RunFromCommandLine `
    --executionConfigId=DemoAllValid `
    -logFile -
```

For new CI scripts, the short form is preferred:

```powershell
& $Unity -batchmode -nographics -quit -projectPath $Project -executeMethod Jdi.Cli.Run -id DemoAllValid -logFile -
```

## Command Quick Pick

- Stable CI scenario: `Jdi.Cli.Run -id <execution_config_id>`.
- Run a specific batch config asset: `Jdi.Cli.Run -ec <asset_path>`.
- Check one source config: `Jdi.Cli.Run -c <asset_path>`.
- Check all source configs: `Jdi.Cli.Run -all`.
- Preview only: add `--preview` to `-c` or `-all`.
- Apply changes: add `--apply` to `-c` or `-all`, or set `executionMode = Apply` in `ImportBatchExecutionConfig`.
- Save a machine-readable report: add `--reportJson=<path>`.
- Save a readable report: add `--reportText=<path>`.

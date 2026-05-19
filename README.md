# JSON Data Importer — Configurable Import Pipeline

A configurable, format-agnostic import pipeline for Unity Editor.

JSON is the built-in automatic path, but the core pipeline is not limited to JSON. Source providers return raw string content, and parsers convert that content into typed entries.

## Overview

JSON Data Importer helps you safely load, validate, preview, and apply structured data into Unity assets.

Instead of manually copying data or writing one-off importer scripts, you create an `ImportSourceConfig` asset that describes the full import workflow:

- where the data comes from;
- how raw content is parsed;
- which Unity asset receives the data;
- how entries are validated;
- how changes are previewed;
- how backups are created;
- how final data is applied.

The main workflow is:

```text
Preview → Validate → Apply
```

Before anything is written to the target asset, the importer can:

- read source data;
- parse raw content into typed entries;
- validate imported entries;
- show diff/preview results;
- create a backup snapshot;
- apply changes safely;
- skip unchanged data using fingerprints.

## Why use it?

JSON Data Importer is designed for Unity projects where structured data is updated frequently outside the editor.

Typical examples:

- game balance data;
- item databases;
- quest data;
- settings/config files;
- generated content;
- external tool exports;
- internal content pipelines;
- automated validation/import workflows.

The goal is to make data import predictable, safe, repeatable, and easy to automate.

## Key Features

### Config-driven import jobs

Each import job is stored as an `ImportSourceConfig` asset.

A config can define:

- source provider;
- target asset;
- import profile;
- parser;
- validators;
- diff engine;
- fingerprint builder;
- apply strategy;
- backup settings;
- execution policy.

### Built-in JSON workflow

For common Unity data workflows, the package provides a built-in JSON path:

- Unity `TextAsset` source;
- auto JSON parser;
- `ScriptableObject` target;
- collection targets;
- single document targets;
- direct fields targets;
- automatic pipeline component selection.

### Format-agnostic core pipeline

The core pipeline is not tied to JSON.

The source provider only reads raw string content.  
The parser decides how that content should be interpreted.

This allows custom workflows for formats such as:

- JSON;
- CSV;
- TSV;
- YAML;
- generated text;
- external tool output;
- any text-based format parsed by a custom parser.

### Validation before apply

Imported data can be checked before it modifies Unity assets.

The pipeline supports:

- required field validation;
- reference validation;
- parser diagnostics;
- custom validators;
- validation messages;
- apply blocking on validation errors.

### Built-in Diff Editor

Preview changes before applying imported data.

The Diff Editor can show:

- added entries;
- updated entries;
- removed entries;
- unchanged entries;
- field-level before/after values.

### Backup History

The importer can create backup snapshots before apply.

Backup History allows you to:

- inspect saved snapshots;
- view snapshot metadata;
- review stored target data;
- restore previous target data when needed.

### Incremental skip

The pipeline can skip unchanged imports.

Using fingerprints and last applied state, the importer can detect when source data, target data, profile, and apply strategy have not changed.

If nothing changed, apply can be skipped instead of rewriting the target asset.

### Batch execution

Multiple import configs can be grouped into a batch execution config.

Batch execution allows you to:

- run several import jobs together;
- apply shared run settings;
- see per-job status;
- review success, failed, skipped, and changed results.

### CI-friendly command-line workflow

The same import workflow can be executed from Unity batch mode.

Useful for:

- automated validation;
- content pipeline checks;
- CI import runs;
- batch reports;
- exit-code based build steps.

## Extension Points

The pipeline can be extended with custom components.

You can add your own:

- `IImportSourceProvider`;
- `IImportParser`;
- `IImportValidator`;
- `IImportTargetAdapter`;
- `IImportProfile`;
- `IImportDiffEngine`;
- `IImportFingerprintBuilder`;
- `IImportApplyStrategy`;
- `IImportBackupCodec`.

Use a custom source provider when data comes from a custom location.  
Use a custom parser when raw content has a custom format or shape.

You can keep the same safe workflow:

```text
Preview → Validate → Apply
```

while replacing only the stage that needs project-specific behavior.

## Basic Workflow

1. Create a serializable entry type.
2. Create a `ScriptableObject` target asset.
3. Create an `ImportSourceConfig`.
4. Select the source provider.
5. Assign the source data.
6. Assign the target asset.
7. Use automatic setup or choose pipeline components manually.
8. Run `Preview`.
9. Run `Validate`.
10. Run `Apply`.

## Documentation

Start here:

- [Getting Started](doc/GettingStarted.md)

Core references:

- [Import Attributes](doc/ImportAttributes.md)
- [ImportSourceConfig Reference](doc/ImportSourceConfigReference.md)
- [Pipeline Stages And Extension Points](doc/PipelineStagesAndExtensionPoints.md)
- [Custom Import Extensions](doc/CustomImportExtensions.md)
- [Backup / Restore / Incremental State](doc/BackupRestoreIncrementalState.md)
- [Import Job Status Troubleshooting](doc/ImportJobStatusTroubleshooting.md)

Automation and CI:

- [CI Commands](doc/CICommands.md)

Internal architecture:

- [Import Execution Flow](doc/ImportExecutionFlow.md)

Editor UI:

- [Editor Display Names And Badges](doc/EditorDisplayNamesAndBadges.md)

## Recommended Reading Order

If you are setting up your first import target:

1. [Getting Started](doc/GettingStarted.md)
2. [Import Attributes](doc/ImportAttributes.md)
3. [ImportSourceConfig Reference](doc/ImportSourceConfigReference.md)
4. [Backup / Restore / Incremental State](doc/BackupRestoreIncrementalState.md)
5. [Import Job Status Troubleshooting](doc/ImportJobStatusTroubleshooting.md)

If you want to extend the pipeline:

1. [Pipeline Stages And Extension Points](doc/PipelineStagesAndExtensionPoints.md)
2. [Custom Import Extensions](doc/CustomImportExtensions.md)
3. [Import Attributes](doc/ImportAttributes.md)
4. [ImportSourceConfig Reference](doc/ImportSourceConfigReference.md)

If you want to automate import in CI:

1. [CI Commands](doc/CICommands.md)
2. [Import Job Status Troubleshooting](doc/ImportJobStatusTroubleshooting.md)
3. [Import Execution Flow](doc/ImportExecutionFlow.md)

## Unity Asset Store

JSON Data Importer — Configurable Import Pipeline is designed for Unity Editor workflows where external or generated data needs to be imported safely, repeatedly, and with clear validation before modifying project assets.

## License

This repository contains public documentation for the Unity package.

The package itself is distributed separately through the Unity Asset Store.

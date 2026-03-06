# cdevents_v0.3_to_v0.4

Migrates CDEvents from spec **v0.3** to spec **v0.4**.

## What it does

- Updates `context.version` from `"0.3.0"` to `"0.4.1"`
- Maps `context.type` event type version strings to their v0.4 equivalents
- Passes all other fields through unchanged (`context.id` is preserved — this transformer migrates an existing event, not creates a new one)
- Unknown event types (not in the mapping table) pass through unchanged

## Usage

```toml
[transformers]
cdevents_v0_3_to_v0_4 = {type = "vrl", template_file = "./to_v0_4.vrl"}
```

Or using the remote repository:

```toml
[remote.transformers-community]
type = "github"
owner = "cdviz-dev"
repo = "transformers-community"

[transformers]
cdevents_v0_3_to_v0_4 = { type = "vrl", template_rfile = "transformers-community:///cdevents/cdevents_v0_3/to_v0_4.vrl"}
```

## Supported event types (39)

| v0.3 type                                  | v0.4 type                                  |
| ------------------------------------------ | ------------------------------------------ |
| `dev.cdevents.artifact.packaged.0.1.1`     | `dev.cdevents.artifact.packaged.0.2.0`     |
| `dev.cdevents.artifact.published.0.1.1`    | `dev.cdevents.artifact.published.0.2.0`    |
| `dev.cdevents.artifact.signed.0.1.0`       | `dev.cdevents.artifact.signed.0.2.0`       |
| `dev.cdevents.branch.created.0.1.2`        | `dev.cdevents.branch.created.0.2.0`        |
| `dev.cdevents.branch.deleted.0.1.2`        | `dev.cdevents.branch.deleted.0.2.0`        |
| `dev.cdevents.build.finished.0.1.1`        | `dev.cdevents.build.finished.0.2.0`        |
| `dev.cdevents.build.queued.0.1.1`          | `dev.cdevents.build.queued.0.2.0`          |
| `dev.cdevents.build.started.0.1.1`         | `dev.cdevents.build.started.0.2.0`         |
| `dev.cdevents.change.abandoned.0.1.2`      | `dev.cdevents.change.abandoned.0.2.0`      |
| `dev.cdevents.change.created.0.1.2`        | `dev.cdevents.change.created.0.3.0`        |
| `dev.cdevents.change.merged.0.1.2`         | `dev.cdevents.change.merged.0.2.0`         |
| `dev.cdevents.change.reviewed.0.1.2`       | `dev.cdevents.change.reviewed.0.2.0`       |
| `dev.cdevents.change.updated.0.1.2`        | `dev.cdevents.change.updated.0.2.0`        |
| `dev.cdevents.environment.created.0.1.1`   | `dev.cdevents.environment.created.0.2.0`   |
| `dev.cdevents.environment.deleted.0.1.1`   | `dev.cdevents.environment.deleted.0.2.0`   |
| `dev.cdevents.environment.modified.0.1.1`  | `dev.cdevents.environment.modified.0.2.0`  |
| `dev.cdevents.incident.detected.0.1.0`     | `dev.cdevents.incident.detected.0.2.0`     |
| `dev.cdevents.incident.reported.0.1.0`     | `dev.cdevents.incident.reported.0.2.0`     |
| `dev.cdevents.incident.resolved.0.1.0`     | `dev.cdevents.incident.resolved.0.2.0`     |
| `dev.cdevents.pipelinerun.finished.0.1.1`  | `dev.cdevents.pipelinerun.finished.0.2.0`  |
| `dev.cdevents.pipelinerun.queued.0.1.1`    | `dev.cdevents.pipelinerun.queued.0.2.0`    |
| `dev.cdevents.pipelinerun.started.0.1.1`   | `dev.cdevents.pipelinerun.started.0.2.0`   |
| `dev.cdevents.repository.created.0.1.1`    | `dev.cdevents.repository.created.0.2.0`    |
| `dev.cdevents.repository.deleted.0.1.1`    | `dev.cdevents.repository.deleted.0.2.0`    |
| `dev.cdevents.repository.modified.0.1.1`   | `dev.cdevents.repository.modified.0.2.0`   |
| `dev.cdevents.service.deployed.0.1.1`      | `dev.cdevents.service.deployed.0.2.0`      |
| `dev.cdevents.service.published.0.1.1`     | `dev.cdevents.service.published.0.2.0`     |
| `dev.cdevents.service.removed.0.1.1`       | `dev.cdevents.service.removed.0.2.0`       |
| `dev.cdevents.service.rolledback.0.1.1`    | `dev.cdevents.service.rolledback.0.2.0`    |
| `dev.cdevents.service.upgraded.0.1.1`      | `dev.cdevents.service.upgraded.0.2.0`      |
| `dev.cdevents.taskrun.finished.0.1.1`      | `dev.cdevents.taskrun.finished.0.2.0`      |
| `dev.cdevents.taskrun.started.0.1.1`       | `dev.cdevents.taskrun.started.0.2.0`       |
| `dev.cdevents.testcaserun.finished.0.1.0`  | `dev.cdevents.testcaserun.finished.0.2.0`  |
| `dev.cdevents.testcaserun.queued.0.1.0`    | `dev.cdevents.testcaserun.queued.0.2.0`    |
| `dev.cdevents.testcaserun.started.0.1.0`   | `dev.cdevents.testcaserun.started.0.2.0`   |
| `dev.cdevents.testoutput.published.0.1.0`  | `dev.cdevents.testoutput.published.0.2.0`  |
| `dev.cdevents.testsuiterun.finished.0.1.0` | `dev.cdevents.testsuiterun.finished.0.2.0` |
| `dev.cdevents.testsuiterun.queued.0.1.0`   | `dev.cdevents.testsuiterun.queued.0.2.0`   |
| `dev.cdevents.testsuiterun.started.0.1.0`  | `dev.cdevents.testsuiterun.started.0.2.0`  |

## Notes

- New v0.4 fields (`chainId`, `links`, `schemaUri`) are not added by this transformer (they are optional in v0.4)
- Optional fields present in the input are passed through unchanged

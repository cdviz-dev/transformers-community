# cdevents_v0.4_to_v0.5

Migrates CDEvents from spec **v0.4** to spec **v0.5**.

## What it does

- Renames `context.version` to `context.specversion` with value `"0.5.0"`
- Maps `context.type` event type version strings to their v0.5 equivalents
- Renames `subject.content.url` to `subject.content.uri` for event types where this field was renamed (environment, pipelineRun, repository, taskRun)
- Maps renamed outcome values for finished test events:
  - `testCaseRun`: `"pass"` → `"success"`
  - `testSuiteRun`: `"fail"` → `"failure"`
- `context.id` is preserved — this transformer migrates an existing event, not creates a new one
- Unknown event types (not in the mapping table) pass through unchanged

## Usage

```toml
[transformers]
cdevents_v0_4_to_v0_5 = { type = "vrl", template_file = "./to_v0_5.vrl"}
```

Or using the remote repository:

```toml
[remote.transformers-community]
type = "github"
owner = "cdviz-dev"
repo = "transformers-community"

[transformers]
cdevents_v0_4_to_v0_5 = { type = "vrl", template_rfile = "transformers-community:///cdevents/cdevents_v0_4/to_v0_5.vrl"}
```

## Supported event types (45)

| v0.4 type                                  | v0.5 type                                  |
| ------------------------------------------ | ------------------------------------------ |
| `dev.cdevents.artifact.deleted.0.1.0`      | `dev.cdevents.artifact.deleted.0.2.0`      |
| `dev.cdevents.artifact.downloaded.0.1.0`   | `dev.cdevents.artifact.downloaded.0.2.0`   |
| `dev.cdevents.artifact.packaged.0.2.0`     | `dev.cdevents.artifact.packaged.0.3.0`     |
| `dev.cdevents.artifact.published.0.2.0`    | `dev.cdevents.artifact.published.0.3.0`    |
| `dev.cdevents.artifact.signed.0.2.0`       | `dev.cdevents.artifact.signed.0.3.0`       |
| `dev.cdevents.branch.created.0.2.0`        | `dev.cdevents.branch.created.0.3.0`        |
| `dev.cdevents.branch.deleted.0.2.0`        | `dev.cdevents.branch.deleted.0.3.0`        |
| `dev.cdevents.build.finished.0.2.0`        | `dev.cdevents.build.finished.0.3.0`        |
| `dev.cdevents.build.queued.0.2.0`          | `dev.cdevents.build.queued.0.3.0`          |
| `dev.cdevents.build.started.0.2.0`         | `dev.cdevents.build.started.0.3.0`         |
| `dev.cdevents.change.abandoned.0.2.0`      | `dev.cdevents.change.abandoned.0.3.0`      |
| `dev.cdevents.change.created.0.3.0`        | `dev.cdevents.change.created.0.4.0`        |
| `dev.cdevents.change.merged.0.2.0`         | `dev.cdevents.change.merged.0.3.0`         |
| `dev.cdevents.change.reviewed.0.2.0`       | `dev.cdevents.change.reviewed.0.3.0`       |
| `dev.cdevents.change.updated.0.2.0`        | `dev.cdevents.change.updated.0.3.0`        |
| `dev.cdevents.environment.created.0.2.0`   | `dev.cdevents.environment.created.0.3.0`   |
| `dev.cdevents.environment.deleted.0.2.0`   | `dev.cdevents.environment.deleted.0.3.0`   |
| `dev.cdevents.environment.modified.0.2.0`  | `dev.cdevents.environment.modified.0.3.0`  |
| `dev.cdevents.incident.detected.0.2.0`     | `dev.cdevents.incident.detected.0.3.0`     |
| `dev.cdevents.incident.reported.0.2.0`     | `dev.cdevents.incident.reported.0.3.0`     |
| `dev.cdevents.incident.resolved.0.2.0`     | `dev.cdevents.incident.resolved.0.3.0`     |
| `dev.cdevents.pipelinerun.finished.0.2.0`  | `dev.cdevents.pipelinerun.finished.0.3.0`  |
| `dev.cdevents.pipelinerun.queued.0.2.0`    | `dev.cdevents.pipelinerun.queued.0.3.0`    |
| `dev.cdevents.pipelinerun.started.0.2.0`   | `dev.cdevents.pipelinerun.started.0.3.0`   |
| `dev.cdevents.repository.created.0.2.0`    | `dev.cdevents.repository.created.0.3.0`    |
| `dev.cdevents.repository.deleted.0.2.0`    | `dev.cdevents.repository.deleted.0.3.0`    |
| `dev.cdevents.repository.modified.0.2.0`   | `dev.cdevents.repository.modified.0.3.0`   |
| `dev.cdevents.service.deployed.0.2.0`      | `dev.cdevents.service.deployed.0.3.0`      |
| `dev.cdevents.service.published.0.2.0`     | `dev.cdevents.service.published.0.3.0`     |
| `dev.cdevents.service.removed.0.2.0`       | `dev.cdevents.service.removed.0.3.0`       |
| `dev.cdevents.service.rolledback.0.2.0`    | `dev.cdevents.service.rolledback.0.3.0`    |
| `dev.cdevents.service.upgraded.0.2.0`      | `dev.cdevents.service.upgraded.0.3.0`      |
| `dev.cdevents.taskrun.finished.0.2.0`      | `dev.cdevents.taskrun.finished.0.3.0`      |
| `dev.cdevents.taskrun.started.0.2.0`       | `dev.cdevents.taskrun.started.0.3.0`       |
| `dev.cdevents.testcaserun.finished.0.2.0`  | `dev.cdevents.testcaserun.finished.0.3.0`  |
| `dev.cdevents.testcaserun.queued.0.2.0`    | `dev.cdevents.testcaserun.queued.0.3.0`    |
| `dev.cdevents.testcaserun.skipped.0.1.0`   | `dev.cdevents.testcaserun.skipped.0.2.0`   |
| `dev.cdevents.testcaserun.started.0.2.0`   | `dev.cdevents.testcaserun.started.0.3.0`   |
| `dev.cdevents.testoutput.published.0.2.0`  | `dev.cdevents.testoutput.published.0.3.0`  |
| `dev.cdevents.testsuiterun.finished.0.2.0` | `dev.cdevents.testsuiterun.finished.0.3.0` |
| `dev.cdevents.testsuiterun.queued.0.2.0`   | `dev.cdevents.testsuiterun.queued.0.3.0`   |
| `dev.cdevents.testsuiterun.started.0.2.0`  | `dev.cdevents.testsuiterun.started.0.3.0`  |
| `dev.cdevents.ticket.closed.0.1.0`         | `dev.cdevents.ticket.closed.0.2.0`         |
| `dev.cdevents.ticket.created.0.1.0`        | `dev.cdevents.ticket.created.0.2.0`        |
| `dev.cdevents.ticket.updated.0.1.0`        | `dev.cdevents.ticket.updated.0.2.0`        |

## Notes

- `context.version` is renamed to `context.specversion` (key rename introduced in v0.5)
- Optional fields present in the input (`chainId`, `links`, `schemaUri`) are passed through unchanged

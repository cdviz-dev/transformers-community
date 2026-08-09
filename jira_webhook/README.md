# JIRA Events Transformer

Transform JIRA webhook events into CDEvents.

## Overview

This transformer converts JIRA webhook events (issues, versions) into standardized CDEvents following the [CDEvents specification](https://cdevents.dev).

Supports both **JIRA Cloud** (uses `accountId` for user identity) and **JIRA Server / Data Center** (uses `name` for user identity).

The transformer uses VRL (Vector Remap Language) to detect event types from webhook payloads and map them to appropriate CDEvents.

### Event Mapping

Event type detection is performed in VRL based on the `webhookEvent` field:

| JIRA Event                          | CDEvent Type               | Detection Logic                                                                |
| ----------------------------------- | -------------------------- | ------------------------------------------------------------------------------ |
| `jira:issue_created`                | `ticket.created.0.2.0`     | `webhookEvent = "jira:issue_created"`                                          |
| `jira:issue_updated` (status=done)  | `ticket.closed.0.2.0`      | `webhookEvent = "jira:issue_updated"` AND `status.statusCategory.key = "done"` |
| `jira:issue_updated` (other status) | `ticket.updated.0.2.0`     | `webhookEvent = "jira:issue_updated"` AND other status category                |
| `jira:issue_deleted` (no resolution) | `ticket.closed.0.2.0`     | `webhookEvent = "jira:issue_deleted"` AND `issue.fields.resolution` is null   |
| `jira:issue_deleted` (with resolution) | _(skipped)_             | `webhookEvent = "jira:issue_deleted"` AND `issue.fields.resolution` is set    |
| `jira:version_released`             | `artifact.published.0.3.0` | `webhookEvent = "jira:version_released"`                                       |

### Subject Fields

For **issue events** (`ticket.*`):

| CDEvent Field        | JIRA Source                                  | Notes                              |
| -------------------- | -------------------------------------------- | ---------------------------------- |
| `subject.id`         | `<base_url>/browse/<issue.key>`              | Browse URL derived from issue.self |
| `content.summary`    | `issue.fields.summary`                       |                                    |
| `content.ticketType` | `issue.fields.issuetype.name` (lowercased)   | e.g. "bug", "story", "task"        |
| `content.uri`        | `<base_url>/browse/<issue.key>`              |                                    |
| `content.priority`   | `issue.fields.priority.name` (lowercased)    | e.g. "high", "medium", "low"       |
| `content.creator`    | `issue.fields.reporter.accountId` or `.name` | Cloud: accountId; Server: name     |
| `content.assignees`  | `issue.fields.assignee.accountId` or `.name` | Cloud: accountId; Server: name     |
| `content.labels`     | `issue.fields.labels[]`                      |                                    |
| `content.milestone`  | `issue.fields.fixVersions[0].name`           | First fix version, or null         |
| `content.resolution` | `issue.fields.resolution.name` mapped to CDEvents enum | `ticket.closed` only; `"withdrawn"` for `issue_deleted` with no prior resolution |

For **version events** (`artifact.published`):

| CDEvent Field       | JIRA Source                                                          | Notes                              |
| ------------------- | -------------------------------------------------------------------- | ---------------------------------- |
| `subject.id`        | PURL: `pkg:generic/<projectKey>@<version>?repository_url=<base_url>` | base_url derived from version.self |
| `context.timestamp` | `version.releaseDate` (date-only `%Y-%m-%d`, parsed to midnight UTC) | Falls back to `now()` if absent    |

### ArtifactId Generation

For `artifact.published` events, the artifactId is a PURL:

```
pkg:generic/<projectKey>@<version_name>?repository_url=<base_url>
```

Where `<base_url>` is extracted from the `version.self` REST URL by stripping the `/rest/api/<version>/version/<id>` suffix, and `<projectKey>` comes from `version.projectKey` (Cloud) or falls back to `version.projectId` (Server).

## Inputs

`inputs/examples` are based on [JIRA Cloud webhook documentation](https://developer.atlassian.com/server/jira/platform/webhooks/).

### Capturing Real JIRA Webhook Payloads

To capture authentic JIRA webhook payloads for testing:

1. Visit [webhook.site](https://webhook.site) and copy your unique URL.
2. In JIRA: **Settings → System → WebHooks → Create a WebHook**.
3. Set the URL to your webhook.site URL.
4. Select the events: **Issue: created, updated** and **Version: released**.
5. Trigger events in JIRA and save the payloads from webhook.site.

## Testing

```bash
# Check transformations against expected outputs
mise run test

# Review and update expected outputs after input changes
mise run test --mode review
```

## Configuration

### Remote (via cdviz-collector remote config)

```toml
[remote.transformers-pro]
type = "github"
owner = "cdviz-dev"
repo = "transformers-pro"
token = "gh...."

[transformers]
jira_events = { type = "vrl", template_rfile = "transformers-pro:///jira_events/transformer.vrl" }
```

### Local

```toml
[transformers]
jira_events = { type = "vrl", template_file = "./path/to/transformers-pro/jira_events/transformer.vrl" }
```

### Source Example

```toml
[sources.jira_webhook]
enabled = true
transformer_refs = ["jira_events"]

[sources.jira_webhook.extractor]
type = "webhook"
id = "000-jira"
headers_to_keep = []
# secure the endpoint with a secret & signature see https://developer.atlassian.com/cloud/jira/platform/webhooks/#secure-admin-webhooks
```

## Design Decisions

- **Cloud/Server compatibility**: User identity falls back from `accountId` (Cloud) to `name` (Server/DC) using `field.accountId || field.name || ""`.
- **Browse URL derivation**: The base URL for browse links is extracted from the REST API `self` URL by stripping the `/rest/api/<version>/issue/<id>` suffix, avoiding the need to configure the JIRA base URL separately.
- **Status category for close detection**: JIRA's `statusCategory.key = "done"` is used rather than specific status names, making the transformer work regardless of custom workflow configurations.
- **Date-only releaseDate**: Version release dates are date-only strings (`%Y-%m-%d`), parsed to midnight UTC. Falls back to `now()` if absent.
- **`jira:issue_deleted` mapped to `ticket.closed`**: Deletion is treated as closure since no dedicated CDEvent type exists for deletion. If the issue already had a resolution set, the event is skipped (a `ticket.closed` was already emitted by the prior `issue_updated`). Otherwise, `resolution` is set to `"withdrawn"`. The timestamp uses the issue's last `updated` field.
- **Resolution mapping**: Jira resolution names are mapped to the CDEvents enum (`completed`, `withdrawn`, `duplicate`) defined in the [ticketclosed schema](https://github.com/cdevents/spec/blob/main/schemas/ticketclosed.json). Unknown values fall back to `downcase(name)`.

## References

- [JIRA Cloud Webhook Documentation](https://developer.atlassian.com/cloud/jira/platform/webhooks/)
- [JIRA Server Webhook Documentation](https://developer.atlassian.com/server/jira/platform/webhooks/)
- [CDEvents Ticket Events](https://cdevents.dev/docs/sdk/go/reference/ticket/)
- [CDEvents Artifact Events](https://cdevents.dev/docs/sdk/go/reference/artifact/)

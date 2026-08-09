# Bitbucket Events Transformer

Transform Bitbucket Cloud webhook events into CDEvents (0.5).

## Overview

This transformer converts Bitbucket Cloud webhook events (pushes, pull requests, issues, commit
statuses) into standardized CDEvents following the [CDEvents specification](https://cdevents.dev).

Unlike GitLab or JIRA, Bitbucket does **not** include the event type in the payload body: it is only
available in the `X-Event-Key` HTTP header. The webhook source **must** therefore keep that header
(`headers_to_keep = ["x-event-key"]`), otherwise no event is emitted.

### Event Mapping

| `X-Event-Key`                | Condition                                             | CDEvent Type          |
| ---------------------------- | ----------------------------------------------------- | --------------------- |
| `repo:push`                  | change with `new.type=branch` and `created`           | branch.created        |
| `repo:push`                  | change with `old.type=branch` and `closed`            | branch.deleted        |
| `repo:push`                  | change with `new.type=tag` and `created`              | artifact.published    |
| `pullrequest:created`        |                                                       | change.created        |
| `pullrequest:updated`        |                                                       | change.updated        |
| `pullrequest:approved`       |                                                       | change.reviewed       |
| `pullrequest:fulfilled`      |                                                       | change.merged         |
| `pullrequest:rejected`       |                                                       | change.abandoned      |
| `issue:created`              |                                                       | ticket.created        |
| `issue:updated`              | `state` in [resolved, closed, invalid, duplicate, wontfix] | ticket.closed    |
| `issue:updated`              | other states (new, open, on hold, ...)                | ticket.updated        |
| `repo:commit_status_created` | `state=INPROGRESS`                                    | pipelineRun.started   |
| `repo:commit_status_updated` | `state` in [SUCCESSFUL, FAILED, STOPPED]              | pipelineRun.finished  |

A single `repo:push` payload can carry several changes, and therefore produce several CDEvents.

Any other event key (comments, `repo:fork`, `repo:updated`, `pullrequest:unapproved`,
`pullrequest:changes_request_*`, ...) produces **no** event.

### Why commit statuses for pipelines?

Bitbucket Cloud has no webhook for Pipelines runs. CI systems (Bitbucket Pipelines, Jenkins, ...)
report their progress through the [commit status API](https://developer.atlassian.com/cloud/bitbucket/rest/api-group-commit-statuses/),
which is what `repo:commit_status_created` / `repo:commit_status_updated` carry. Each commit status
`key` is an independent CI run, so it is mapped to a `pipelineRun` (`pipelineName` =
`<workspace>/<repo>/<key>`) rather than a `taskRun`, which would require a parent pipeline id that
the payload does not provide.

Outcome mapping: `SUCCESSFUL` → `success`, `FAILED` → `failure`, `STOPPED` → `error`.

### ArtifactId Generation

For `artifact.published` events (tag push), artifactId is generated as a PURL:

- `pkg:generic/<workspace>/<repo_slug>@<tag_name>?repository_url=<repository html url>`

## Inputs

`inputs/examples` are built from the shapes documented in
[Event payloads](https://support.atlassian.com/bitbucket-cloud/docs/event-payloads/).
Each `<name>.json` has a `<name>.headers.json` sidecar providing `X-Event-Key`.

Real captured payloads can be added under `inputs/capture/<event_key>/<verb>.json`
(+ obfuscation via `mise run :obfuscate`).

### Capturing Real Bitbucket Webhook Payloads

cdviz-collector has no mode to capture incoming events "as is" to the file system, so use a
webhook inspection service such as [webhook.site](https://webhook.site) to collect payloads and
headers.

1. Visit [webhook.site](https://webhook.site) and copy your unique URL.
2. In Bitbucket, go to **Repository settings > Webhooks > Add webhook** (or
   **Workspace settings > Webhooks** to cover every repository).
3. Set the **URL** to your webhook.site URL,
   a **Secret** (used for the `X-Hub-Signature` HMAC), and select the triggers:
   - ✅ Repository: Push
   - ✅ Repository: Build status created / updated
   - ✅ Pull request: Created, Updated, Approved, Merged, Declined
   - ✅ Issue: Created, Updated
4. Trigger events (push a branch, push a tag, open a pull request, ...), then copy both the JSON
   body **and** the request headers from webhook.site into `inputs/capture/...`.

## Testing

```bash
# Check transformation against expected outputs (validates CDEvents)
mise run '//bitbucket_events:test'

# Review and update expected outputs after adding new inputs
mise run '//bitbucket_events:test' -- --mode review
```

## Transformer Configuration

The transformer is declared in `cdviz-collector.toml`:

```toml
[transformers]
bitbucket_events_0_5 = { type = "vrl", template_file = "./to_v0_5.vrl" }
```

**Webhook source configuration** (in your main `cdviz-collector.toml`):

```toml
[sources.bitbucket_webhook]
enabled = true
transformer_refs = ["bitbucket_events_0_5"]

[sources.bitbucket_webhook.extractor]
type = "webhook"
id = "000-bitbucket"
# required: bitbucket only exposes the event type as a header
headers_to_keep = ["x-event-key"]

[sources.bitbucket_webhook.extractor.headers]
"x-hub-signature" = { type = "signature", signature_encoding = "hex", signature_on = "body", signature_prefix = "sha256=", token = "changeme" }
```

Bitbucket signs the raw body with HMAC-SHA256 and sends it as `X-Hub-Signature: sha256=<hex>`
(note: **no** `-256` suffix in the header name, unlike GitHub).

Send a test event:

```bash
curl -X POST http://localhost:8080/webhook/000-bitbucket \
  -H "Content-Type: application/json" \
  -H "X-Event-Key: repo:push" \
  -d @bitbucket_events/inputs/examples/push_branch_created.json
```

## Troubleshooting

### No event produced

1. Check that the event key is mapped (see the table above) — unmapped keys are silently ignored.
2. Check that `headers_to_keep` contains `x-event-key`; without it the transformer cannot determine
   the event type and emits nothing.
3. Check the delivery in **Repository settings > Webhooks > View requests**.

### Signature rejected

1. The secret in `cdviz-collector.toml` must match the webhook secret configured in Bitbucket.
2. The header is `x-hub-signature`, with a `sha256=` prefix and hex encoding.

## References

- [Bitbucket Cloud: Manage webhooks](https://support.atlassian.com/bitbucket-cloud/docs/manage-webhooks/)
- [Bitbucket Cloud: Event payloads](https://support.atlassian.com/bitbucket-cloud/docs/event-payloads/)
- [Bitbucket Cloud REST API](https://developer.atlassian.com/cloud/bitbucket/rest/)
- [CDEvents Specification](https://cdevents.dev/)
- [VRL Documentation](https://vector.dev/docs/reference/vrl/)

## Design Decisions

### Header-based event detection

Bitbucket payloads have no `object_kind` equivalent, so `X-Event-Key` is the only discriminator.
The transformer looks up both `X-Event-Key` and `x-event-key`: the `transform` subcommand preserves
the original header case, while extractors may normalize it.

### Generic PURL for tags

Bitbucket tags may reference any kind of artifact (container images, packages, binaries), so
`pkg:generic` is used. For a specific artifact type, extend the transformer to emit a type-specific
PURL.

### Event Coverage

**Supported**:

- ✅ Branch creation / deletion
- ✅ Tag push (artifact published)
- ✅ Pull requests (created, updated, merged, declined, approved)
- ✅ Issues (created, updated, closed)
- ✅ CI status via commit statuses (started, finished)

**Not supported**: comments, forks, repository updates, deployments, `pullrequest:unapproved` and
`pullrequest:changes_request_*`. These can be added following the existing mapping pattern.

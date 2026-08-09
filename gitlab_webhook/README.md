# GitLab Events Transformer

Transform GitLab webhook events into CDEvents.

## Overview

This transformer converts GitLab webhook events (pipelines, jobs, merge requests, issues, releases, etc.) into standardized CDEvents following the [CDEvents specification](https://cdevents.dev).

The transformer uses VRL (Vector Remap Language) to detect event types from webhook payloads and map them to appropriate CDEvents.

### Event Mapping

Event type detection is performed in VRL based on the `object_kind` field or `X-Gitlab-Event` header:

| GitLab Event              | CDEvent Type           | Detection Logic                                                                            |
| ------------------------- | ---------------------- | ------------------------------------------------------------------------------------------ |
| pipeline:created/pending  | pipelineRun.queued     | `object_kind=pipeline` AND `status` in [created, waiting_for_resource, preparing, pending] |
| pipeline:running          | pipelineRun.started    | `object_kind=pipeline` AND `status=running`                                                |
| pipeline:success/failed   | pipelineRun.finished   | `object_kind=pipeline` AND `status` in [success, failed, canceled, skipped]                |
| build:created/pending     | taskRun.queued         | `object_kind=build` AND `build_status` in [created, pending]                               |
| build:running             | taskRun.started        | `object_kind=build` AND `build_status=running`                                             |
| build:success/failed      | taskRun.finished       | `object_kind=build` AND `build_status` in [success, failed, canceled]                      |
| release:created           | artifact.published     | `object_kind=release`                                                                      |
| tag_push                  | artifact.published     | `object_kind=tag_push` AND tag created                                                     |
| issue:open/reopen         | ticket.created         | `object_kind=issue` AND `action` in [open, reopen]                                         |
| issue:close               | ticket.closed          | `object_kind=issue` AND `action=close`                                                     |
| issue:update              | ticket.updated         | `object_kind=issue` AND other actions                                                      |
| merge_request:open/reopen | change.created         | `object_kind=merge_request` AND `action` in [open, reopen]                                 |
| merge_request:merge       | change.merged          | `object_kind=merge_request` AND `action=merge`                                             |
| merge_request:close       | change.abandoned       | `object_kind=merge_request` AND `action=close` (not merged)                                |
| merge_request:approved    | change.reviewed        | `object_kind=merge_request` AND `action=approved`                                          |
| merge_request:update      | change.updated         | `object_kind=merge_request` AND other actions                                              |
| push (branch)             | branch.created/deleted | `object_kind=push` AND `ref` starts with `refs/heads/`                                     |

### ArtifactId Generation

For `artifact.published` events, artifactId is generated as a PURL:

- **Release**: `pkg:generic/<project_path>@<tag_name>?repository_url=<encoded_url>`
- **Tag Push**: `pkg:generic/<project_path>@<tag_name>?repository_url=<encoded_url>`

## Inputs

- `inputs/examples` are copied/extracted from [GitLab documentation](https://docs.gitlab.com/user/project/integrations/webhook_events).
- `inputs/capture` real captured payload (+ obfuscation)

### Capturing Real GitLab Webhook Payloads

To capture authentic GitLab webhook payloads for testing:

#### 1. Set Up webhook.site

1. Visit [webhook.site](https://webhook.site)
2. Copy your unique URL (e.g., `https://webhook.site/abc123-def456-...`)
3. Keep the browser tab open to monitor incoming webhooks

#### 2. Configure GitLab Webhooks

##### Step 1: Navigate to Webhook Settings

1. Go to your GitLab project
2. Navigate to **Settings > Webhooks**
3. Or visit: `https://gitlab.com/<namespace>/<project>/-/hooks`

##### Step 2: Create Webhook

Add webhook configuration:

**URL**: `https://webhook.site/YOUR-UNIQUE-ID` (for testing) or `https://your-cdviz-collector.example.com/webhook/000-gitlab` (for production)

**Secret token**: `changeme` (optional, for webhook authentication)

**Trigger events**: Select the events you want to capture:

- ✅ Push events
- ✅ Tag push events
- ✅ Issues events
- ✅ Merge request events
- ✅ Pipeline events
- ✅ Job events
- ✅ Release events

**SSL verification**: Enable (recommended for production)

##### Step 3: Test Webhook

Click **Test** next to the webhook to send test events, or trigger real events by:

```bash
# Trigger a pipeline
git push origin main

# Create a tag
git tag v1.0.0
git push origin v1.0.0

# Create an issue
# (via GitLab UI)

# Create a merge request
git checkout -b feature-branch
git commit --allow-empty -m "Test commit"
git push origin feature-branch
# (then create MR via GitLab UI)
```

#### 3. Capture Webhook Payloads

1. Go to your webhook.site browser tab
2. Click on each webhook request to view the full payload
3. Copy the JSON body
4. Save to `inputs/capture/<event_type>/<verb>.json`

Example file naming:

- `inputs/capture/pipeline/created.json` - Pipeline queued
- `inputs/capture/pipeline/running.json` - Pipeline started
- `inputs/capture/pipeline/success.json` - Pipeline completed successfully
- `inputs/capture/pipeline/failed.json` - Pipeline failed
- `inputs/capture/job/running.json` - Job started
- `inputs/capture/job/success.json` - Job completed
- `inputs/capture/issue/open.json` - Issue created
- `inputs/capture/issue/close.json` - Issue closed
- `inputs/capture/merge_request/open.json` - MR created
- `inputs/capture/merge_request/merge.json` - MR merged
- `inputs/capture/release/created.json` - Release published
- `inputs/capture/tag_push/created.json` - Tag pushed
- `inputs/capture/push/branch_created.json` - Branch created
- `inputs/capture/push/branch_deleted.json` - Branch deleted

## Testing

### Run Transformer Tests

```bash
# Check transformation against expected outputs (validates CDEvents)
mise run '//gitlab_events:test'

# Generate/overwrite expected outputs (after adding new inputs)
cd transformers/gitlab_events
../../target/debug/cdviz-collector transform \
  --mode overwrite \
  --directory . \
  --config ./cdviz-collector.toml \
  -t gitlab_events \
  --input ./inputs \
  --output ./outputs
```

### Test with Local Server

Configure collector to accept GitLab webhooks:

```toml
# In your main cdviz-collector.toml
[sources.gitlab_webhook]
enabled = true
transformer_refs = ["gitlab_events"]

[sources.gitlab_webhook.extractor]
type = "webhook"
id = "000-gitlab"
headers_to_keep = ["X-Gitlab-Event"]

[sources.gitlab_webhook.extractor.headers]
# Optional: Verify webhook authenticity with token
"x-gitlab-token" = { type = "secret", value = "token-changeme" }
```

Start the server:

```bash
cdviz-collector connect
```

Send test events:

```bash
# Using curl with a captured webhook
curl -X POST http://localhost:8080/webhook/000-gitlab \
  -H "Content-Type: application/json" \
  -H "X-Gitlab-Token: token-changeme" \
  -d @transformers/gitlab_events/inputs/pipeline/success.json
```

## Transformer Configuration

The transformer is configured via `cdviz-collector.toml`:

```toml
[transformers.gitlab_events]
type = "vrl"
template_file = "./transformer.vrl"
```

**Webhook source configuration** (in main `cdviz-collector.toml`):

```toml
[sources.gitlab_webhook]
enabled = true
transformer_refs = ["gitlab_events"]

[sources.gitlab_webhook.extractor]
type = "webhook"
id = "000-gitlab"
headers_to_keep = ["X-Gitlab-Event"]

[sources.gitlab_webhook.extractor.headers]
# Optional: Verify webhook authenticity with token
"x-gitlab-token" = { type = "secret", value = "token-changeme" }
```

## VRL Transformation Logic

Key transformation rules:

- **Event detection**: VRL logic determines event type from `object_kind` field or `X-Gitlab-Event` header
- **context.id**: Auto-generated by collector (set to "0")
- **context.source**: `/gitlab` (consistent with other transformers)
- **subject.id**: Web URL of the entity (pipeline, job, issue, MR, etc.) or PURL for artifacts
- **subject.source**: Project web URL
- **customData.gitlab**: Preserves GitLab-specific details (project, user, event-specific metadata)

## Troubleshooting

### Webhooks Not Received

1. Verify webhook is enabled in GitLab project settings:

   ```
   Settings > Webhooks
   ```

2. Check webhook delivery logs in GitLab:

   ```
   Settings > Webhooks > Edit > Recent Deliveries
   ```

3. Verify cdviz-collector is running and accessible:

   ```bash
   curl http://localhost:8080/healthz
   ```

4. Check cdviz-collector logs for webhook processing errors

### Invalid Webhook Payloads

If webhook.site shows errors or malformed JSON:

1. Verify GitLab webhook configuration includes correct Content-Type header
2. Check GitLab version compatibility (tested with GitLab 15.0+)
3. Review Recent Deliveries in GitLab for error messages

### Token Authentication Issues

If using X-Gitlab-Token authentication:

1. Ensure token in cdviz-collector.toml matches GitLab webhook secret token
2. Verify header name is exactly `x-gitlab-token` (case-insensitive)
3. Check collector logs for authentication errors

## References

- [GitLab Webhook Events Documentation](https://docs.gitlab.com/user/project/integrations/webhook_events/)
- [GitLab Webhooks Guide](https://docs.gitlab.com/ee/user/project/integrations/webhooks.html)
- [CDEvents Specification](https://cdevents.dev/)
- [webhook.site](https://webhook.site) - Free webhook testing tool
- [VRL Documentation](https://vector.dev/docs/reference/vrl/)

## Design Decisions

### Why Event Detection in VRL?

**Centralized Logic**: Event type detection in VRL keeps transformation logic in one place, making it easier to maintain and test. GitLab provides clear event type indicators in the webhook payload (`object_kind` field), making VRL-based detection straightforward.

**Benefits**:

- Single source of truth for event mapping
- Easier to add new event types
- Testable transformation logic
- Version-controlled event detection rules

### Why Generic PURL for Releases?

**Flexible Artifact Identification**: GitLab releases can contain various artifact types (container images, packages, binaries). Using `pkg:generic` allows representing any release artifact with a consistent PURL format.

**Alternative**: For specific artifact types (container images, npm packages, etc.), the transformer can be extended to generate type-specific PURLs based on release asset information.

### Event Coverage

**Supported Events**: The transformer covers the most common GitLab CI/CD and development workflow events:

- ✅ Pipeline lifecycle (queued, started, finished)
- ✅ Job lifecycle (queued, started, finished)
- ✅ Issues (created, updated, closed)
- ✅ Merge requests (created, updated, merged, abandoned, reviewed)
- ✅ Releases and tags (artifact published)
- ✅ Branch operations (created, deleted)

**Not Yet Supported**:

- Deployment events → `service.deployed`
- Wiki page events
- Comment events
- Confidential issues/MRs
- System hooks

These can be added as needed following the existing event mapping pattern.

# gitea_webhook

Converts [Gitea webhook events](https://docs.gitea.com/usage/webhooks) into
[CDEvents](https://cdevents.dev) **v0.5**.

> [!IMPORTANT]
> For Forgejo, use [`forgejo_webhook`](../forgejo_webhook/) instead.
> Forgejo is a Gitea fork and most payloads are still byte-identical, **but the CI events are not**:
> Gitea sends `workflow_run` / `workflow_job`, while Forgejo sends `action_run_success` /
> `action_run_failure` / `action_run_recover`. Using the wrong transformer silently drops all pipeline
> events.

## Event mapping

| Gitea event                                                    | action                             | CDEvent                                             |
| -------------------------------------------------------------- | ---------------------------------- | --------------------------------------------------- |
| `workflow_run`                                                 | `requested` / `queued` / `waiting` | `pipelineRun:queued`                                |
| `workflow_run`                                                 | `in_progress`                      | `pipelineRun:started`                               |
| `workflow_run`                                                 | `completed`                        | `pipelineRun:finished`                              |
| `workflow_job`                                                 | `in_progress`                      | `taskRun:started`                                   |
| `workflow_job`                                                 | `completed`                        | `taskRun:finished`                                  |
| `workflow_job`                                                 | `waiting` / `queued`               | — (no `taskRun:queued` in CDEvents)                 |
| `package`                                                      | `created`                          | `artifact:published`                                |
| `package`                                                      | `deleted`                          | `artifact:deleted`                                  |
| `release`                                                      | `published`                        | `artifact:published` (+ one per release asset)      |
| `release`                                                      | `deleted`                          | `artifact:deleted`                                  |
| `release`                                                      | `updated`                          | — (would re-publish identical artifact coordinates) |
| `pull_request` (+ `_assign`, `_label`, `_milestone`, `_sync`)  | `opened`                           | `change:created`                                    |
| `pull_request`                                                 | `closed` (merged / not merged)     | `change:merged` / `change:abandoned`                |
| `pull_request`                                                 | any other                          | `change:updated`                                    |
| `pull_request_review` (+ `_approved`, `_rejected`, `_comment`) | `reviewed`                         | `change:reviewed`                                   |
| `pull_request_comment`                                         | any                                | `change:updated`                                    |
| `issues` (+ `issue_assign`, `_label`, `_milestone`)            | `opened`                           | `ticket:created`                                    |
| `issues`                                                       | `closed`                           | `ticket:closed`                                     |
| `issues`                                                       | any other                          | `ticket:updated`                                    |
| `issue_comment`                                                | any                                | `ticket:updated`                                    |
| `create` (`ref_type: branch`)                                  |                                    | `branch:created`                                    |
| `delete` (`ref_type: branch`)                                  |                                    | `branch:deleted`                                    |
| `create` / `delete` (`ref_type: tag`)                          |                                    | — (no tag subject in CDEvents)                      |
| `repository`                                                   | `created` / `deleted`              | `repository:created` / `repository:deleted`         |
| `fork`                                                         |                                    | `repository:created` (for the fork)                 |
| `push`, `wiki`, `status`, `schedule`                           |                                    | — (no CDEvents equivalent)                          |

### Event detection

Gitea has ~28 hook event types but only a handful of payload structs: the fine-grained types
(`issue_assign`, `pull_request_label`, `pull_request_sync`, `pull_request_review_approved`, …) send the
_same_ body as their base type and differ only by the `X-Gitea-Event` header and the `action` field. The
transformer therefore detects the event on body fields, and the test order matters:
`workflow_run` / `workflow_job` / `issue_comment` payloads all carry an **optional** `pull_request` field,
so they are tested before `.body.pull_request`.

## Known limitations

Some information CDEvents (or `RULES.md`) would like is simply absent from the Gitea payloads:

- **Container artifacts have no digest.** The `package` payload carries only the tag, so the OCI PURL uses
  the tag as version (`pkg:oci/name@1.2.3?repository_url=…&tag=1.2.3`) instead of the `sha256:…` digest
  recommended by `RULES.md`.
- **`create` / `delete` carry no timestamp.** `repository.updated_at` (bumped by the push that
  created/deleted the ref) is used instead, so `context.timestamp` is approximate for `branch:created` and
  `branch:deleted`.
- **`ticket:closed` has no real resolution.** Gitea issues have no `state_reason` equivalent, so
  `subject.content.resolution` is always `"completed"` (the field is required by the CDEvents schema).
- **Release assets have no checksum** in the payload, so only `download_url` is set as a PURL qualifier.
- **`taskName` has no workflow prefix.** The `workflow_job` payload does not include the workflow name, so
  the task name is `{repository.full_name}/{job.name}` (the `workflow_run` events do get the workflow name,
  from the sibling `.workflow` object).

## Usage

Remotely, without cloning:

```toml
[remote.transformers-community]
type = "github://cdviz-dev/transformers-community"

[transformers]
gitea_webhook = { type = "vrl", template_rfile = "transformers-community:///gitea_webhook/to_v0_5.vrl" }
```

Locally:

```toml
[transformers]
gitea_webhook = { type = "vrl", template_file = "./path/to/transformers-community/gitea_webhook/to_v0_5.vrl" }
```

Wire it to a webhook source (see `cdviz-collector.toml` for the full commented example):

```toml
[sources.gitea_webhook]
enabled = true
transformer_refs = ["gitea_webhook"]

[sources.gitea_webhook.extractor]
type = "webhook"
id = "000-gitea"

[sources.gitea_webhook.extractor.headers]
# gitea signs with HMAC-SHA256, hex encoded, over the raw body, without prefix
"x-gitea-signature" = { type = "signature", signature_encoding = "hex", signature_on = "body", token = "changeme" }
```

In Gitea, add the webhook with type **Gitea**, method `POST`, content type `application/json`, and select
the events you want (or "All events" — unmapped ones are silently ignored).

## Testing

```bash
mise run //gitea_webhook:test                  # check against the expected outputs
mise run //gitea_webhook:test -- --mode review # review and update the expected outputs
```

> [!NOTE]
> The samples under `inputs/` are **synthetic**: they were built from the Gitea payload structs
> (`modules/structs/hook.go`, `modules/structs/repo_actions.go`, …), not captured from a live instance.
> Replace one with a real (obfuscated) capture and re-run with `--mode review` whenever a real payload
> turns out to differ.

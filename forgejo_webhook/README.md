# forgejo_webhook

Converts [Forgejo webhook events](https://forgejo.org/docs/latest/user/repository/webhooks/) into
[CDEvents](https://cdevents.dev) **v0.5**.

> [!IMPORTANT]
> For Gitea, use [`gitea_webhook`](../gitea_webhook/) instead.
> Forgejo is a Gitea fork and most payloads are still byte-identical, **but the CI events are not**:
> Forgejo sends `action_run_success` / `action_run_failure` / `action_run_recover` (an `ActionPayload`),
> while Gitea sends `workflow_run` / `workflow_job`. Using the wrong transformer silently drops all
> pipeline events.

## Event mapping

| Forgejo event                                                 | action                         | CDEvent                                             |
| ------------------------------------------------------------- | ------------------------------ | --------------------------------------------------- |
| `action_run_success`                                          | `success`                      | `pipelineRun:finished` (outcome `success`)          |
| `action_run_recover`                                          | `recover`                      | `pipelineRun:finished` (outcome `success`)          |
| `action_run_failure`                                          | `failure`                      | `pipelineRun:finished` (outcome `failure`)          |
| `package`                                                     | `created`                      | `artifact:published`                                |
| `package`                                                     | `deleted`                      | `artifact:deleted`                                  |
| `release`                                                     | `published`                    | `artifact:published` (+ one per release asset)      |
| `release`                                                     | `deleted`                      | `artifact:deleted`                                  |
| `release`                                                     | `updated`                      | — (would re-publish identical artifact coordinates) |
| `pull_request` (+ `_assign`, `_label`, `_milestone`, `_sync`) | `opened`                       | `change:created`                                    |
| `pull_request`                                                | `closed` (merged / not merged) | `change:merged` / `change:abandoned`                |
| `pull_request`                                                | any other                      | `change:updated`                                    |
| `pull_request_review_approved` / `_rejected` / `_comment`     | `reviewed`                     | `change:reviewed`                                   |
| `pull_request_comment`                                        | any                            | `change:updated`                                    |
| `issues` (+ `issue_assign`, `_label`, `_milestone`)           | `opened`                       | `ticket:created`                                    |
| `issues`                                                      | `closed`                       | `ticket:closed`                                     |
| `issues`                                                      | any other                      | `ticket:updated`                                    |
| `issue_comment`                                               | any                            | `ticket:updated`                                    |
| `create` (`ref_type: branch`)                                 |                                | `branch:created`                                    |
| `delete` (`ref_type: branch`)                                 |                                | `branch:deleted`                                    |
| `create` / `delete` (`ref_type: tag`)                         |                                | — (no tag subject in CDEvents)                      |
| `repository`                                                  | `created` / `deleted`          | `repository:created` / `repository:deleted`         |
| `fork`                                                        |                                | `repository:created` (for the fork)                 |
| `push`, `wiki`, `workflow_dispatch`, `schedule`               |                                | — (no CDEvents equivalent)                          |

### Event detection

Forgejo has ~28 hook event types but only a handful of payload structs: the fine-grained types
(`issue_assign`, `pull_request_label`, `pull_request_sync`, `pull_request_review_approved`, …) send the
_same_ body as their base type and differ only by the `X-Forgejo-Event` header and the `action` field. The
transformer therefore detects the event on body fields, and the test order matters: the `issue_comment`
payload carries an **optional** `pull_request` field, so it is tested before `.body.pull_request`.

## Known limitations

Some information CDEvents (or `RULES.md`) would like is simply absent from the Forgejo payloads:

- **No `pipelineRun:queued` or `:started`.** Forgejo only notifies on terminal states
  (`success` / `failure` / `recover`), so only `pipelineRun:finished` is reachable. Gitea, which has
  `workflow_run:requested` / `:in_progress`, does emit the full lifecycle.
- **No `taskRun` at all.** Forgejo has no job-level webhook (no `workflow_job` equivalent).
- **`ActionRun` has no API URL.** Only `html_url` is provided, so `subject.id` is rebuilt as
  `{repository.url}/actions/runs/{index_in_repo}` and `subject.content.uri` keeps the `html_url`.
- **`action_run_*` payloads carry no top-level `repository`/`sender`** — both are read from `.run`
  (`.run.repository`, `.run.trigger_user`).
- **Container artifacts have no digest.** The `package` payload carries only the tag, so the OCI PURL uses
  the tag as version (`pkg:oci/name@1.2.3?repository_url=…&tag=1.2.3`) instead of the `sha256:…` digest
  recommended by `RULES.md`.
- **`create` / `delete` carry no timestamp.** `repository.updated_at` (bumped by the push that
  created/deleted the ref) is used instead, so `context.timestamp` is approximate for `branch:created` and
  `branch:deleted`.
- **`ticket:closed` has no real resolution.** Forgejo issues have no `state_reason` equivalent, so
  `subject.content.resolution` is always `"completed"` (the field is required by the CDEvents schema).
- **Release assets have no checksum** in the payload, so only `download_url` is set as a PURL qualifier.

## Usage

Remotely, without cloning:

```toml
[remote.transformers-community]
type = "github://cdviz-dev/transformers-community"

[transformers]
forgejo_webhook = { type = "vrl", template_rfile = "transformers-community:///forgejo_webhook/to_v0_5.vrl" }
```

Locally:

```toml
[transformers]
forgejo_webhook = { type = "vrl", template_file = "./path/to/transformers-community/forgejo_webhook/to_v0_5.vrl" }
```

Wire it to a webhook source (see `cdviz-collector.toml` for the full commented example):

```toml
[sources.forgejo_webhook]
enabled = true
transformer_refs = ["forgejo_webhook"]

[sources.forgejo_webhook.extractor]
type = "webhook"
id = "000-forgejo"

[sources.forgejo_webhook.extractor.headers]
# forgejo signs with HMAC-SHA256, hex encoded, over the raw body, without prefix
"x-forgejo-signature" = { type = "signature", signature_encoding = "hex", signature_on = "body", token = "changeme" }
```

In Forgejo, add the webhook with type **Forgejo**, method `POST`, content type `application/json`, and
select the events you want (or "All events" — unmapped ones are silently ignored).

## Testing

```bash
mise run //forgejo_webhook:test                  # check against the expected outputs
mise run //forgejo_webhook:test -- --mode review # review and update the expected outputs
```

> [!NOTE]
> The samples under `inputs/` are **synthetic**: they were built from the Forgejo payload structs
> (`modules/structs/hook.go`, `modules/structs/action.go`, …), not captured from a live instance.
> Replace one with a real (obfuscated) capture and re-run with `--mode review` whenever a real payload
> turns out to differ.

# gcp_cloudbuild_webhook

Convert [Cloud Build build notifications](https://cloud.google.com/build/docs/subscribe-build-notifications)
delivered via a Pub/Sub **push** subscription into CDEvents `pipelineRun` events.

## How it works

Cloud Build automatically publishes every `Build` resource update to the built-in Pub/Sub topic
`cloud-builds` in the project where the build runs. A push subscription on that topic delivers
each update as an HTTPS POST with the standard Pub/Sub push envelope:

```json
{
  "message": {
    "data": "<base64-encoded Build JSON>",
    "attributes": { "buildId": "...", "status": "..." },
    "messageId": "...",
    "publishTime": "..."
  },
  "subscription": "projects/PROJECT/subscriptions/SUBSCRIPTION"
}
```

cdviz-collector's generic `webhook` source accepts this envelope unmodified — no dedicated
Pub/Sub client is needed for push delivery, it's plain HTTPS. `to_v0_5.vrl` decodes
`message.data` and maps the [`Build`](https://cloud.google.com/build/docs/api/reference/rest/v1/projects.builds#Build)
resource's `status` transitions to `dev.cdevents.pipelinerun.{queued,started,finished}`, mirroring
`github_rest_api/workflow_runs_to_v0_5.vrl`.

Since Cloud Build publishes on every status change (not only the terminal one), a build usually
generates several webhook deliveries; each delivery re-derives the full lifecycle so far
(queued/started/finished) from the one `Build` snapshot it carries, using stable per-phase
timestamps — repeated/duplicate deliveries produce content-identical (dedupable) events.

## GCP setup

1. Enable Cloud Build in the project (this creates the `cloud-builds` topic automatically on the
   first build).
2. Create a push subscription on `cloud-builds` pointing at
   `https://<your-collector-host>/webhook/000-gcp-cloudbuild` (or whatever `id` you configure).
3. Grant the Pub/Sub service agent permission to invoke the endpoint if it's behind IAM (e.g. Cloud
   Run ingress) — see [push subscription authentication](https://cloud.google.com/pubsub/docs/authenticate-push-subscriptions).

## cdviz-collector configuration

See the commented example at the top of `cdviz-collector.toml`.

## Testing

```bash
mise run test                      # check mode
mise run test -- --mode overwrite  # regenerate outputs/0_5 after editing to_v0_5.vrl
```

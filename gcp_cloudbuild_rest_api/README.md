# gcp_cloudbuild_rest_api

Convert [Cloud Build `builds.list`](https://cloud.google.com/build/docs/api/reference/rest/v1/projects.builds/list)
REST API responses into CDEvents `pipelineRun` events, for polling/backfill setups without a
Pub/Sub push endpoint (see `gcp_cloudbuild_webhook/` for the real-time push-based alternative).

## How it works

`builds_to_v0_5.vrl` maps each `Build`'s `status` transitions to
`dev.cdevents.pipelinerun.{queued,started,finished}`, following the same multi-phase, idempotent
pattern as `github_rest_api/workflow_runs_to_v0_5.vrl`: a terminal-status build re-synthesizes all
prior lifecycle events from that single poll response, using each phase's own stable timestamp and
excluding volatile fields (`status`, `images`) from earlier phases' `customData` — so polling the
same build repeatedly produces content-identical (dedupable) events.

## GCP setup

The polling identity (service account or user) needs `cloudbuild.builds.list` on the target
project (role `roles/cloudbuild.builds.viewer` or broader).

## Auth caveat

cdviz-collector's `http_polling` engine has **no built-in OAuth2/service-account token refresh**.
`cdviz-collector.sources.example.toml` uses a `secret` header pointed at a token file — you are
responsible for keeping that file populated with a valid, unexpired
`gcloud auth print-access-token` (or equivalent) output via an external process.

## cdviz-collector configuration

See `cdviz-collector.sources.example.toml` for a ready-to-adapt `http_polling` source, and
`cdviz-collector.toml` for the transformer definition.

## Testing

```bash
mise run test                      # check mode
mise run test -- --mode overwrite  # regenerate outputs/0_5 after editing builds_to_v0_5.vrl
```

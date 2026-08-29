# gcp_artifactregistry_rest_api

Convert [Artifact Registry `packages.versions.list`](https://cloud.google.com/artifact-registry/docs/reference/rest/v1/projects.locations.repositories.packages.versions/list)
responses into CDEvents `artifact.published` events, for polling/backfill setups without a
Pub/Sub push endpoint (see `gcp_artifactregistry_webhook/` for the audit-log-based real-time
alternative).

## How it works

`versions_to_v0_5.vrl` treats every version in the response as a publish event and builds an OCI
PURL from the version's digest (`.../versions/sha256:DIGEST`), the immutable identifier — same
digest-first rule as the webhook package. Since `context.id` is content-based, re-polling an
already-seen version produces an identical (dedupable) event.

**Scope**: Docker/OCI packages only (see `gcp_artifactregistry_webhook/README.md` for why).

## GCP setup

The polling identity needs `artifactregistry.repositories.downloadArtifacts` +
`artifactregistry.versions.list` on the target repository
(role `roles/artifactregistry.reader` or broader).

## Auth caveat

Same as `gcp_cloudbuild_rest_api/` — no built-in GCP OAuth2/service-account token refresh in
cdviz-collector's `http_polling` engine; supply and rotate the bearer token yourself.

## cdviz-collector configuration

See `cdviz-collector.sources.example.toml` for a ready-to-adapt `http_polling` source (discovers
packages in a repository, then fans out to each one's versions).

## Testing

```bash
mise run test                      # check mode
mise run test -- --mode overwrite  # regenerate outputs/0_5 after editing versions_to_v0_5.vrl
```

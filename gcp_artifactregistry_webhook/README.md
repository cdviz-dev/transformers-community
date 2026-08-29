# gcp_artifactregistry_webhook

Convert Artifact Registry admin-activity [audit log](https://cloud.google.com/artifact-registry/docs/audit-logging)
entries, delivered via a Pub/Sub push subscription, into CDEvents `artifact.published` events.

This is the source for provenance of images/packages pushed to GCP from _outside_ GCP (local
Docker, GitHub Actions, etc.) — the flow where Cloud Build isn't involved at all, but Cloud
Run/GKE still need to pull a known-good artifact.

## How it works

Like Cloud Run, Artifact Registry publishes no dedicated event topic; route
`artifactregistry.googleapis.com` admin-activity audit logs to Pub/Sub via a
[logging sink](https://cloud.google.com/logging/docs/export/configure_export_v2), then push that
topic to cdviz-collector's `webhook` source (same push-envelope handling as
`gcp_cloudbuild_webhook/`).

`to_v0_5.vrl` treats `UploadDockerImage`, `CreateDockerImage`, and `CreateTag` methods as a
publish event and builds `subject.id` as an OCI PURL parsed out of the audit log's
`resourceName` (`projects/P/locations/L/repositories/R/dockerImages/NAME@sha256:DIGEST`). All
other methods (reads, deletes, repository management) are ignored.

**Scope**: Docker/OCI images only for this first pass — the concrete need driving this package.
Artifact Registry also hosts npm/Maven/generic formats with different `resourceName` shapes; add
those if/when needed, following the same pattern.

## GCP setup

1. Create a Cloud Logging sink on `artifactregistry.googleapis.com` audit logs (filter:
   `protoPayload.serviceName="artifactregistry.googleapis.com" AND protoPayload.methodName:("UploadDockerImage" OR "CreateDockerImage" OR "CreateTag")`),
   destination = a Pub/Sub topic.
2. Create a push subscription on that topic pointing at
   `https://<your-collector-host>/webhook/000-gcp-artifactregistry`.

## cdviz-collector configuration

See the commented example at the top of `cdviz-collector.toml`.

## Testing

```bash
mise run test                      # check mode
mise run test -- --mode overwrite  # regenerate outputs/0_5 after editing to_v0_5.vrl
```

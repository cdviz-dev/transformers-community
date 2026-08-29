# gcp_cloudrun_webhook

Convert Cloud Run admin-activity [audit log](https://cloud.google.com/run/docs/audit-logging)
entries, delivered via a Pub/Sub push subscription, into CDEvents `service` events.

## How it works

Unlike Cloud Build, Cloud Run has no dedicated event topic — service mutations are only visible
via Cloud Audit Logs. Route them to Pub/Sub with a
[logging sink](https://cloud.google.com/logging/docs/export/configure_export_v2) filtered to
`run.googleapis.com` admin-activity entries, then push that topic to cdviz-collector's `webhook`
source, same push-envelope handling as `gcp_cloudbuild_webhook/`.

`to_v0_5.vrl` maps `protoPayload.methodName` to a CDEvent:

| methodName                                   | cdevent            |
| -------------------------------------------- | ------------------ |
| `...Services.CreateService`                  | `service.deployed` |
| `...Services.UpdateService`/`ReplaceService` | `service.upgraded` |
| `...Services.DeleteService`                  | `service.removed`  |

**Not implemented**: `service.rolledback`. Cloud Run's audit log doesn't self-label a traffic
revert as distinct from a regular update — every update is treated as `upgraded`. Detecting an
actual rollback would require comparing the new revision against prior traffic-split history,
which this first pass doesn't attempt.

When the mutation response carries a container image, `subject.content.artifactId` is set as a
PURL. GCP image references are digest-form (`repo/name@sha256:...`); the transformer builds the
PURL manually instead of using the `purl_from_oci_image()` stdlib helper, which is built for
tag-form references and mis-parses the `@sha256:` suffix.

## GCP setup

1. Create a Cloud Logging sink on `run.googleapis.com` audit logs (filter:
   `resource.type="cloud_run_revision" AND logName:"cloudaudit.googleapis.com%2Factivity"`),
   destination = a Pub/Sub topic.
2. Create a push subscription on that topic pointing at
   `https://<your-collector-host>/webhook/000-gcp-cloudrun`.

## cdviz-collector configuration

See the commented example at the top of `cdviz-collector.toml`.

## Testing

```bash
mise run test                      # check mode
mise run test -- --mode overwrite  # regenerate outputs/0_5 after editing to_v0_5.vrl
```

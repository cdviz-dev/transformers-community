# gcp_cloudrun_rest_api

Convert [Cloud Run Admin API `services.list`](https://cloud.google.com/run/docs/reference/rest/v2/projects.locations.services/list)
responses into CDEvents `service.deployed` events, for polling setups without a Pub/Sub push
endpoint (see `gcp_cloudrun_webhook/` for the audit-log-based real-time alternative).

## How it works and limitations

This is a state snapshot, not a change feed: every poll reports every service's _current_ state
as `service.deployed`. Because `context.id` is content-based, polling an unchanged service
repeatedly produces an identical (dedupable) event — only an actual revision/image change produces
a new one. This package **cannot** distinguish `upgraded` vs `removed` vs `rolledback`; if you
need that, use `gcp_cloudrun_webhook/` (audit-log based) instead, or run both.

`subject.content.artifactId` is built as a PURL from the serving container image, same digest-form
handling as `gcp_cloudrun_webhook/` (see that package's README for why `purl_from_oci_image()`
isn't used directly).

## GCP setup

The polling identity needs `run.services.list` on the target project/location
(role `roles/run.viewer` or broader).

## Auth caveat

Same as `gcp_cloudbuild_rest_api/` — no built-in GCP OAuth2/service-account token refresh in
cdviz-collector's `http_polling` engine; supply and rotate the bearer token yourself.

## cdviz-collector configuration

See `cdviz-collector.sources.example.toml` for a ready-to-adapt `http_polling` source.

## Testing

```bash
mise run test                      # check mode
mise run test -- --mode overwrite  # regenerate outputs/0_5 after editing services_to_v0_5.vrl
```

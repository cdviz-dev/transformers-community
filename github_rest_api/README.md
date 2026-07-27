# github_rest_api

Transforms paginated [GitHub REST API](https://docs.github.com/en/rest) responses into [CDEvents](https://cdevents.dev) (v0.5).

Use for historical **backfill** before switching to webhook-based ingestion ([github_events](../github_events/)), or for environments where GitHub webhooks are not available.

## Event Mappings

| GitHub REST API endpoint                                        | CDEvent type                          |
| --------------------------------------------------------------- | ------------------------------------- |
| `GET /repos/{owner}/{repo}/actions/runs` — status `queued`      | `pipelinerun.queued`                  |
| `GET /repos/{owner}/{repo}/actions/runs` — status `in_progress` | `pipelinerun.started`                 |
| `GET /repos/{owner}/{repo}/actions/runs` — status `completed`   | `pipelinerun.finished`                |
| `GET /repos/{owner}/{repo}/pulls` — any                         | `change.created`                      |
| `GET /repos/{owner}/{repo}/pulls` — state `closed`, merged      | `change.created` + `change.merged`    |
| `GET /repos/{owner}/{repo}/pulls` — state `closed`, not merged  | `change.created` + `change.abandoned` |
| `GET /repos/{owner}/{repo}/releases`                            | `artifact.published`                  |
| `GET /repos/{owner}/{repo}/releases` — each asset               | `artifact.published`                  |
| `GET /{orgs\|users}/{owner}/packages/{type}/{name}/versions`    | `artifact.published`                  |
| `GET /repos/{owner}/{repo}/issues` — any (non-PR)               | `ticket.created`                      |
| `GET /repos/{owner}/{repo}/issues` — state `closed` (non-PR)    | `ticket.created` + `ticket.closed`    |
| `GET /repos/{owner}/{repo}/deployments`                         | `service.deployed`                    |
| `GET /orgs/{org}/repos` or `/user/repos`                        | `repository.created`                  |
| `GET /repos/{owner}/{repo}/environments`                        | `environment.created`                 |
| `GET /repos/{owner}/{repo}/branches`                            | `branch.created`                      |

> **Note on branches**: GitHub's branches API does not expose branch creation timestamps.
> The transformer uses `.metadata.ts_after` (polling window start) as a proxy, falling back
> to `now()`. Timestamps for branch events are approximate.

> [!NOTE]
> A REST snapshot only reports the _current_ state, so backfilling already-closed items would
> otherwise never produce the lifecycle start. Each item therefore fans out into every phase whose
> timestamp the payload carries: a merged PR yields `change.created` (at `created_at`) **and**
> `change.merged` (at `merged_at`); a closed issue yields `ticket.created` **and** `ticket.closed`;
> a completed workflow run yields queued + started + finished. This keeps lead time, cycle time and
> run duration computable from CDEvents alone.
> `customData` for the earlier phases deliberately omits volatile fields (`state`, `conclusion`,
> `merged_at`, …) so that re-polling the same item later yields the same content-based id instead of
> a duplicate event.

## Usage

### Remote (recommended)

```toml
[remote.transformers-community]
type = "github"
owner = "cdviz-dev"
repo = "transformers-community"

[transformers.github_rest_workflow_runs]
type = "vrl"
template_rfile = "transformers-community:///github_rest_api/workflow_runs_to_v0_5.vrl"

[transformers.github_rest_pull_requests]
type = "vrl"
template_rfile = "transformers-community:///github_rest_api/pull_requests_to_v0_5.vrl"

[transformers.github_rest_releases]
type = "vrl"
template_rfile = "transformers-community:///github_rest_api/releases_to_v0_5.vrl"

[transformers.github_rest_package_versions]
type = "vrl"
template_rfile = "transformers-community:///github_rest_api/package_versions_to_v0_5.vrl"

# Add other transformers as needed: issues, deployments, repositories, environments, branches
```

### Source configuration (multi-pass discovery)

Sources use the [`http_polling`](../../cdviz-collector/src/sources/http_polling/README.md)
extractor. A single inline `driver_vrl` script builds a worklist of HTTP requests; each
response is **routed**:

- `pipeline` (default) — body is parsed and forwarded to the transformer.
- `feedback` — response is handed back to `driver_vrl` to compute more requests (discovery,
  pagination); **not** emitted downstream.
- `both` — emit **and** feed back (used for resource pages that must be transformed _and_
  paginated).

This enables **discovery**: list an org's repositories (or packages), then fan out to fetch
each one's resources — no need to hardcode every repo/package. The discovery list is routed
`feedback` (never transformed); only the per-item resource pages are routed to the transformer.

Pagination is manual — read the `Link` header and re-issue the `rel="next"` URL. Because
`.state` is cloned into _every_ request of one driver call, responses are distinguished by
their originating URL (`.request.url`), not by state.

```toml
[sources.github_rest_releases]
enabled = true
transformer_refs = ["github_rest_releases"]

[sources.github_rest_releases.extractor]
type                 = "http_polling"
polling_interval     = "1h"
min_request_interval = "720ms"
parser               = "json"
# ts_after        = "2024-01-01T00:00:00Z"   # optional backfill lower bound
# ts_before_limit = "2025-01-01T00:00:00Z"   # optional: stop after this window

driver_vrl = """
org = "myorg"
api = "https://api.github.com"
reqs = []
if .response == null {
  # pass 1: discover repos (feedback only — the repos list is not a release event)
  reqs = [{ "url": api + "/orgs/" + org + "/repos", "query": {"per_page": "100"}, "route": "feedback" }]
} else {
  url = string(.request.url) ?? ""
  is_resource = contains(url, "/releases")
  # paginate the current listing via the Link header
  matched = parse_regex(string(.response.headers.link) ?? "", r'<(?P<next>[^>]+)>;\\s*rel="next"') ?? {}
  if exists(matched.next) {
    reqs = push(reqs, { "url": matched.next, "route": (if is_resource { "both" } else { "feedback" }) })
  }
  if !is_resource {
    # pass 2: per repo, fetch its releases (emitted via "both", paginated above)
    for_each(array!(.response.body)) -> |_i, repo| {
      reqs = push(reqs, { "url": string!(repo.url) + "/releases", "query": {"per_page": "100"}, "route": "both" })
    }
  }
}
.requests = reqs
"""

[sources.github_rest_releases.extractor.headers]
authorization          = { type = "secret", value_file = "/secrets/github_pat_token", prefix = "Bearer " }
"x-github-api-version" = { type = "static", value = "2022-11-28" }
```

The same per-repo pattern applies to every repository-scoped resource — only the endpoint
suffix and `transformer_refs` change:

| Transformer                 | Per-repo endpoint suffix             |
| --------------------------- | ------------------------------------ |
| `github_rest_releases`      | `/releases`                          |
| `github_rest_workflow_runs` | `/actions/runs?created=>{ts_after}`  |
| `github_rest_pull_requests` | `/pulls?state=all`                   |
| `github_rest_issues`        | `/issues?state=all&since={ts_after}` |
| `github_rest_deployments`   | `/deployments`                       |
| `github_rest_environments`  | `/environments`                      |
| `github_rest_branches`      | `/branches`                          |

`github_rest_repositories` is simpler: the repos list _is_ the event, so route it `pipeline`
directly (no fan-out).

[`cdviz-collector.sources.example.toml`](./cdviz-collector.sources.example.toml) ships
ready-to-use `enabled = false` source blocks for all of these — copy one, set `org`, replace the
authorization secret, enable it. (Requires a cdviz-collector with `driver_vrl`; the older
`request_vrl` API is not compatible.)

#### Package versions (multi-pass)

Multi-pass removes the old "one source per package" limit: discover an org's packages, then
fan out to each package's versions endpoint.

```toml
[sources.github_rest_packages]
enabled = true
transformer_refs = ["github_rest_package_versions"]

[sources.github_rest_packages.extractor]
type             = "http_polling"
polling_interval = "1h"
parser           = "json"

driver_vrl = """
org = "myorg"
api = "https://api.github.com"
reqs = []
if .response == null {
  # pass 1: discover packages, one request per package type (feedback only)
  types = ["container", "npm", "maven", "nuget", "rubygems", "docker"]
  reqs = map_values(types) -> |t| {
    { "url": api + "/orgs/" + org + "/packages", "query": {"package_type": t, "per_page": "100"}, "route": "feedback" }
  }
} else {
  url = string(.request.url) ?? ""
  is_versions = contains(url, "/versions")
  matched = parse_regex(string(.response.headers.link) ?? "", r'<(?P<next>[^>]+)>;\\s*rel="next"') ?? {}
  if exists(matched.next) {
    reqs = push(reqs, { "url": matched.next, "route": (if is_versions { "both" } else { "feedback" }) })
  }
  if !is_versions {
    # pass 2: per package, fetch its versions (emitted via "both")
    for_each(array!(.response.body)) -> |_i, pkg| {
      reqs = push(reqs, { "url": string!(pkg.url) + "/versions", "query": {"per_page": "100"}, "route": "both" })
    }
  }
}
.requests = reqs
"""

[sources.github_rest_packages.extractor.headers]
authorization          = { type = "secret", value_file = "/secrets/github_pat_token", prefix = "Bearer " }
"x-github-api-version" = { type = "static", value = "2022-11-28" }
```

Required GitHub token permissions:

| Endpoint         | Permission           |
| ---------------- | -------------------- |
| workflow runs    | `actions:read`       |
| pull requests    | `pull_requests:read` |
| releases         | `contents:read`      |
| package versions | `packages:read`      |
| issues           | `issues:read`        |
| deployments      | `deployments:read`   |
| repositories     | `metadata:read`      |
| environments     | `deployments:read`   |
| branches         | `contents:read`      |

## Development

### Capture real inputs

```bash
# From this directory — downloads page 1 of each endpoint for a pinned 7-day window
mise run :capture

# Custom repo/window
mise run :capture -- --repo myorg/myrepo --ts-after 2024-06-01T00:00:00Z --ts-before 2024-06-08T00:00:00Z

# Redact sensitive data before committing
cd .. && mise run :obfuscate
```

### Test

```bash
# From this directory
mise run :test                           # check all against expected outputs
mise run :test -- --mode review          # interactive review to accept new outputs
mise run :test:workflow_runs             # check one transformer
mise run :test:workflow_runs -- --mode review  # review one transformer

# From repo root
mise run //github_rest_api:test
mise run //github_rest_api:test -- --mode review
```

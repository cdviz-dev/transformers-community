# github_rest_api

Transforms paginated [GitHub REST API](https://docs.github.com/en/rest) responses into [CDEvents](https://cdevents.dev) (v0.5).

Use for historical **backfill** before switching to webhook-based ingestion ([github_events](../github_events/)), or for environments where GitHub webhooks are not available.

## Event Mappings

| GitHub REST API endpoint                                        | CDEvent type           |
| --------------------------------------------------------------- | ---------------------- |
| `GET /repos/{owner}/{repo}/actions/runs` — status `queued`      | `pipelinerun.queued`   |
| `GET /repos/{owner}/{repo}/actions/runs` — status `in_progress` | `pipelinerun.started`  |
| `GET /repos/{owner}/{repo}/actions/runs` — status `completed`   | `pipelinerun.finished` |
| `GET /repos/{owner}/{repo}/pulls` — state `open`                | `change.created`       |
| `GET /repos/{owner}/{repo}/pulls` — state `closed`, merged      | `change.merged`        |
| `GET /repos/{owner}/{repo}/pulls` — state `closed`, not merged  | `change.abandoned`     |
| `GET /repos/{owner}/{repo}/releases`                            | `artifact.published`   |
| `GET /repos/{owner}/{repo}/issues` — state `open` (non-PR)      | `ticket.created`       |
| `GET /repos/{owner}/{repo}/issues` — state `closed` (non-PR)    | `ticket.closed`        |
| `GET /repos/{owner}/{repo}/deployments`                         | `service.deployed`     |
| `GET /orgs/{org}/repos` or `/user/repos`                        | `repository.created`   |
| `GET /repos/{owner}/{repo}/environments`                        | `environment.created`  |
| `GET /repos/{owner}/{repo}/branches`                            | `branch.created`       |

> **Note on branches**: GitHub's branches API does not expose branch creation timestamps.
> The transformer uses `.metadata.ts_after` (polling window start) as a proxy, falling back
> to `now()`. Timestamps for branch events are approximate.

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

# Add other transformers as needed: issues, deployments, repositories, environments, branches
```

### Source configuration for backfill

Each transformer expects one GitHub REST API endpoint as its source. Use `http_polling` with `follow_link_header = true` to consume all pages:

```toml
[sources.github_workflow_runs_myrepo]
enabled = true
transformer_refs = ["github_rest_workflow_runs"]

[sources.github_workflow_runs_myrepo.extractor]
type                 = "http_polling"
polling_interval     = "1s"
follow_link_header   = true
min_request_interval = "720ms"
ts_after             = "2024-01-01T00:00:00Z"
ts_before_limit      = "2025-01-01T00:00:00Z"
parser               = "json"

request_vrl = """
.url = "https://api.github.com/repos/myorg/myrepo/actions/runs"
.query.created  = ">" + to_string!(.metadata.ts_after)
.query.per_page = "100"
"""

[sources.github_workflow_runs_myrepo.extractor.headers]
Authorization        = { type = "secret", value_file = "/secrets/github_pat_token", prefix = "Bearer " }
X-GitHub-Api-Version = { type = "static", value = "2022-11-28" }

[sources.github_workflow_runs_myrepo.extractor.metadata]
environment_id = "/github/myorg/myrepo"
```

Repeat for each endpoint/transformer you need. See [`cdviz-collector.toml`](./cdviz-collector.toml) for all transformer definitions.

Required GitHub token permissions:

| Endpoint      | Permission           |
| ------------- | -------------------- |
| workflow runs | `actions:read`       |
| pull requests | `pull_requests:read` |
| releases      | `contents:read`      |
| issues        | `issues:read`        |
| deployments   | `deployments:read`   |
| repositories  | `metadata:read`      |
| environments  | `deployments:read`   |
| branches      | `contents:read`      |

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

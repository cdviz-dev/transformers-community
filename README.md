# CDviz Community Transformers

Community driven transformers for [cdviz-collector](https://github.com/cdviz-dev/cdviz-collector) that convert various event sources into [CDEvents](https://cdevents.dev).

## Available Transformers

| Transformer                                       | Source             | Description                                                                                                                                                                    |
| ------------------------------------------------- | ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| [argocd_notifications](./argocd_notifications/)   | ArgoCD Webhooks    | Converts ArgoCD application lifecycle events to CDEvents                                                                                                                       |
| [bitbucket_webhook](./bitbucket_webhook/)         | Bitbucket Webhooks | Converts Bitbucket Cloud webhook events (pushes, pull requests, issues, commit statuses) to CDEvents                                                                           |
| [forgejo_webhook](./forgejo_webhook/)             | Forgejo Webhooks   | Converts Forgejo webhook events (action runs, packages, releases, issues, PRs, branches, repositories) to CDEvents                                                             |
| [cdevents](./cdevents/)                           | CDEvents           | Convert CDEvents from a version to the next version (can be chained)                                                                                                           |
| [gitea_webhook](./gitea_webhook/)                 | Gitea Webhooks     | Converts Gitea webhook events (workflow runs/jobs, packages, releases, issues, PRs, branches, repositories) to CDEvents                                                        |
| [github_events](./github_events/)                 | GitHub Webhooks    | Converts GitHub events (workflow runs, issues, PRs, releases) to CDEvents                                                                                                      |
| [github_rest_api](./github_rest_api/)             | GitHub REST API    | Converts GitHub REST API responses (workflow runs, PRs, releases, issues, deployments, repositories, environments, branches) to CDEvents — for backfill or polling-only setups |
| [gitlab_webhook](./gitlab_webhook/)               | GitLab Webhooks    | Converts GitLab webhook events (pipelines, jobs, merge requests, issues, releases) to CDEvents                                                                                 |
| [jenkins_rest_api](./jenkins_rest_api/)           | Jenkins REST API   | Converts Jenkins build history polled via the Jenkins Remote API to CDEvents — for instances without webhook support                                                           |
| [jira_webhook](./jira_webhook/)                   | JIRA Webhooks      | Converts JIRA webhook events (issues, versions) to CDEvents                                                                                                                    |
| [kubewatch_cloudevents](./kubewatch_cloudevents/) | Kubernetes Events  | Converts Kubewatch CloudEvents to CDEvents                                                                                                                                     |
| [passthrough](./passthrough/)                     | CDEvents           | Passthrough transformer for existing CDEvents                                                                                                                                  |

### CDEvents Coverage

Rows are source applications, columns are [CDEvents subjects](https://cdevents.dev/docs/), cells list the supported predicates.

| Source                                                                    | artifact           | branch           | change                                        | environment | incident | pipelineRun               | repository       | service                     | taskRun           | ticket                   |
| ------------------------------------------------------------------------- | ------------------ | ---------------- | --------------------------------------------- | ----------- | -------- | ------------------------- | ---------------- | --------------------------- | ----------------- | ------------------------ |
| ArgoCD Notifications ([argocd_notifications](./argocd_notifications/))    |                    |                  |                                               |             | detected |                           |                  | deployed, removed           |                   |                          |
| Bitbucket Webhooks ([bitbucket_webhook](./bitbucket_webhook/))            | published          | created, deleted | created, updated, merged, abandoned, reviewed |             |          | started, finished         |                  |                             |                   | created, updated, closed |
| Forgejo Webhooks ([forgejo_webhook](./forgejo_webhook/))                  | published, deleted | created, deleted | created, updated, merged, abandoned, reviewed |             |          | queued, started, finished | created, deleted |                             |                   | created, updated, closed |
| Gitea Webhooks ([gitea_webhook](./gitea_webhook/))                        | published, deleted | created, deleted | created, updated, merged, abandoned, reviewed |             |          | queued, started, finished | created, deleted |                             | started, finished | created, updated, closed |
| GitHub REST API ([github_rest_api](./github_rest_api/))                   | published          |                  | created, merged, abandoned                    | created     |          | queued, started, finished | created          | deployed                    |                   | created, closed          |
| GitHub Webhooks ([github_events](./github_events/))                       | published          | created, deleted | created, updated, merged, abandoned, reviewed |             |          | queued, started, finished |                  |                             | started, finished | created, updated, closed |
| GitLab Webhooks ([gitlab_webhook](./gitlab_webhook/))                     | published          | created, deleted | created, updated, merged, abandoned, reviewed |             |          | queued, started, finished |                  |                             | started, finished | created, updated, closed |
| Jenkins REST API ([jenkins_rest_api](./jenkins_rest_api/))                |                    |                  |                                               |             |          | started, finished         |                  |                             |                   |                          |
| JIRA Webhooks ([jira_webhook](./jira_webhook/))                           | published          |                  |                                               |             |          |                           |                  |                             |                   | created, updated, closed |
| Kubewatch CloudEvents ([kubewatch_cloudevents](./kubewatch_cloudevents/)) |                    |                  |                                               |             |          |                           |                  | deployed, removed, upgraded |                   |                          |

Some predicates are _inferred_ rather than observed: when a source only reports a terminal state but
its payload carries the earlier timestamps, the transformer reconstructs the missing lifecycle events.
This applies to `pipelineRun:queued`/`started` for Forgejo (flagged with `customData.inferred`), and to
`change:created` / `ticket:created` when backfilling already-closed items from the GitHub REST API
(not flagged — a REST snapshot is a state, not an event, so every predicate it yields is reconstructed).

This table is maintained from the `to_v0_x.vrl` sources, and is merged with the equivalent tables of the other transformer repositories into the documentation site "integrations" page.

## Usage

### Import Transformers Remotely

Add to your `cdviz-collector.toml`:

```toml
[remote.transformers-community]
type = "github://cdviz-dev/transformers-community"
# token = "gh...."  # Optional: github token

[transformers]
cdevents_v0_3_to_v0_4 = { type = "vrl", template_rfile = "transformers-community:///cdevents/cdevents_v0_3/to_v0_4.vrl"}
cdevents_v0_4_to_v0_5 = { type = "vrl", template_rfile = "transformers-community:///cdevents/cdevents_v0_4/to_v0_5.vrl"}

argocd_notifications = { type = "vrl", template_rfile = "transformers-community:///argocd_notifications/to_v0_4.vrl" }
bitbucket_webhook = { type = "vrl", template_rfile = "transformers-community:///bitbucket_webhook/to_v0_5.vrl" }
github_events = { type = "vrl", template_rfile = "transformers-community:///github_events/to_v0_5.vrl" }
gitlab_webhook = { type = "vrl", template_rfile = "transformers-community:///gitlab_webhook/to_v0_5.vrl" }
jenkins_rest_api = { type = "vrl", template_rfile = "transformers-community:///jenkins_rest_api/to_v0_5.vrl" }
jira_webhook = { type = "vrl", template_rfile = "transformers-community:///jira_webhook/to_v0_5.vrl" }
kubewatch_cloudevents = { type = "vrl", template_rfile = "transformers-community:///kubewatch_cloudevents/to_v0_4.vrl" }
```

### Use Transformers Locally

Clone the repository and reference transformers directly:

```toml
[transformers]
github_events = { type = "vrl", template_file = "./path/to/transformers-community/github_events/to_v0_5.vrl" }
```

## Development

### Testing

Each transformer includes sample inputs and expected outputs for testing:

```bash
# Test all transformers
mise run :test

# Test specific transformer
mise run //github_events:test

# Review and update expected outputs
mise run //github_events:test -- --mode review
```

### Code Style

```bash
# Format code
mise run :format
```

See [AGENTS.md](./AGENTS.md) for detailed guidelines on code style and conventions.

## Contributing

Contributions are welcome! Each transformer should include:

- `to_v0_x.vrl` - VRL transformation logic to convert to CDEvents v0.x
- `transformer.vrl` - legacy / deprecated it's a symlink to `to_v0_4.vrl`, kept for backward compatibility
- `cdviz-collector.toml` - Configuration example
- `inputs/` - Sample input events
- `outputs/` - Expected output CDEvents
- `README.md` - Documentation with usage examples
- `mise.toml` - Task to test/review the transformer against its inputs.

## License

Apache-2.0

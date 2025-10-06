# CDviz Community Transformers

Community driven transformers for [cdviz-collector](https://github.com/cdviz-dev/cdviz-collector) that convert various event sources into [CDEvents](https://cdevents.dev).

## Available Transformers

| Transformer                                       | Source            | Description                                                               |
| ------------------------------------------------- | ----------------- | ------------------------------------------------------------------------- |
| [github_events](./github_events/)                 | GitHub Webhooks   | Converts GitHub events (workflow runs, issues, PRs, releases) to CDEvents |
| [argocd_notifications](./argocd_notifications/)   | ArgoCD Webhooks   | Converts ArgoCD application lifecycle events to CDEvents                  |
| [kubewatch_cloudevents](./kubewatch_cloudevents/) | Kubernetes Events | Converts Kubewatch CloudEvents to CDEvents                                |
| [passthrough](./passthrough/)                     | CDEvents          | Passthrough transformer for existing CDEvents                             |

## Usage

### Import Transformers Remotely

Add to your `cdviz-collector.toml`:

```toml
[remote.transformers-community]
type = "github"
owner = "cdviz-dev"
repo = "transformers-community"
# reference = "main"  # Optional: pin to specific branch/tag

[transformers]
github_events = { type = "vrl", template_rfile = "transformers-community:///github_events/transformer.vrl" }
kubewatch_cloudevents = { type = "vrl", template_rfile = "transformers-community:///kubewatch_cloudevents/transformer.vrl" }
argocd_notifications = { type = "vrl", template_rfile = "transformers-community:///argocd_notifications/transformer.vrl" }
```

### Use Transformers Locally

Clone the repository and reference transformers directly:

```toml
[transformers.github_events]
type = "vrl"
template_file = "./path/to/transformers-community/github_events/transformer.vrl"
```

## Development

### Testing

Each transformer includes sample inputs and expected outputs for testing:

```bash
# Test all transformers
mise run test

# Test specific transformer
mise run test:transform:github_events

# Review and update expected outputs
mise run test:transform:github_events -- --mode review
```

### Code Style

```bash
# Format code
mise run format
```

See [AGENTS.md](./AGENTS.md) for detailed guidelines on code style and conventions.

## Contributing

Contributions are welcome! Each transformer should include:

- `transformer.vrl` - VRL transformation logic
- `cdviz-collector.toml` - Configuration example
- `inputs/` - Sample input events
- `outputs/` - Expected output CDEvents
- `README.md` - Documentation with usage examples

And a command into the `.mise.toml` to test/review the transformer against its inputs.

## License

Apache-2.0

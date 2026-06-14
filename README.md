# CDviz Community Transformers

Community driven transformers for [cdviz-collector](https://github.com/cdviz-dev/cdviz-collector) that convert various event sources into [CDEvents](https://cdevents.dev).

## Available Transformers

| Transformer                                       | Source            | Description                                                               |
| ------------------------------------------------- | ----------------- | ------------------------------------------------------------------------- |
| [github_events](./github_events/)                 | GitHub Webhooks   | Converts GitHub events (workflow runs, issues, PRs, releases) to CDEvents |
| [argocd_notifications](./argocd_notifications/)   | ArgoCD Webhooks   | Converts ArgoCD application lifecycle events to CDEvents                  |
| [kubewatch_cloudevents](./kubewatch_cloudevents/) | Kubernetes Events | Converts Kubewatch CloudEvents to CDEvents                                |
| [cdevents](./cdevents/)                           | CDEvents          | Convert CDEvents from a version to the next version (can be chained)      |
| [passthrough](./passthrough/)                     | CDEvents          | Passthrough transformer for existing CDEvents                             |

## Usage

### Import Transformers Remotely

Add to your `cdviz-collector.toml`:

```toml
[remote.transformers-community]
type = "github://cdviz-dev/transformers-community"
# token = "gh...."  # Optional: github token

[transformers]
github_events = { type = "vrl", template_rfile = "transformers-community:///github_events/to_v0_5.vrl" }
kubewatch_cloudevents = { type = "vrl", template_rfile = "transformers-community:///kubewatch_cloudevents/to_v0_4.vrl" }
argocd_notifications = { type = "vrl", template_rfile = "transformers-community:///argocd_notifications/to_v0_4.vrl" }
cdevents_v0_3_to_v0_4 = { type = "vrl", template_rfile = "transformers-community:///cdevents/cdevents_v0_3/to_v0_4.vrl"}
cdevents_v0_4_to_v0_5 = { type = "vrl", template_rfile = "transformers-community:///cdevents/cdevents_v0_4/to_v0_5.vrl"}
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

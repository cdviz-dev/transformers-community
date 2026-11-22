# dockerhub_webhook

Transform [Docker Hub repository webhooks](https://docs.docker.com/docker-hub/repos/manage/webhooks/) (image push) into CDEvents `artifact.published` events.

## Event Mapping

| Docker Hub trigger | CDEvent Type               |
| ------------------ | -------------------------- |
| image push         | `artifact.published.0.3.0` |

## Subject Fields

| CDEvent Field       | Docker Hub Source                                                                               |
| ------------------- | ----------------------------------------------------------------------------------------------- |
| `subject.id`        | `pkg:oci/<repository.name>?repository_url=docker.io/<repository.repo_name>&tag=<push_data.tag>` |
| `content.user`      | `push_data.pusher`                                                                              |
| `context.timestamp` | `push_data.pushed_at` (epoch seconds)                                                           |

`customData.dockerhub` keeps `push_data` and the identity fields of `repository`. `callback_url` is not kept: anybody with it can post a validation status for the webhook.

## Limits

- **No digest**: the Docker Hub payload has no image digest, so the PURL has no version and uses the `tag` qualifier.
  A tag can move, so the id is not immutable. When you need the digest, add a later transformer or enricher that resolves it from the registry.

## Docker Hub setup

1. In Docker Hub: open the repository → **Webhooks**.
2. Set the URL to `https://<your-collector-host>/webhook/000-dockerhub`.
3. Docker Hub does not sign webhooks: use a hard to guess extractor `id` and keep the URL secret.

## Configuration

See the commented example at the top of [`cdviz-collector.toml`](./cdviz-collector.toml).

```toml
[remote.transformers-community]
type = "github://cdviz-dev/transformers-community"

[transformers]
dockerhub_webhook = { type = "vrl", template_rfile = "transformers-community:///dockerhub_webhook/to_v0_5.vrl" }
```

## Testing

```bash
mise run test                   # check mode
mise run test -- --mode review  # review and update outputs/0_5
```

## References

- [Docker Hub Webhooks](https://docs.docker.com/docker-hub/repos/manage/webhooks/)
- [PURL OCI type](https://github.com/package-url/purl-spec/blob/main/types-doc/oci-definition.md)

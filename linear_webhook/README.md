# linear_webhook

Transform [Linear](https://linear.app) webhook events into CDEvents.

## Event Mapping

Only `Issue` webhooks are handled. Other types (`Comment`, `Project`, `Cycle`, ...) are ignored.

| Linear `type` / `action`                  | CDEvent Type           | Notes                                                |
| ----------------------------------------- | ---------------------- | ---------------------------------------------------- |
| `Issue` / `create`                        | `ticket.created.0.2.0` |                                                      |
| `Issue` / `update`, state moved to closed | `ticket.closed.0.2.0`  | `updatedFrom.stateId` set and `state.type` is closed |
| `Issue` / `update`, other                 | `ticket.updated.0.2.0` | includes edits of an already closed issue            |
| `Issue` / `remove`, issue not closed      | `ticket.closed.0.2.0`  | `resolution = "withdrawn"`                           |
| `Issue` / `remove`, issue already closed  | _(skipped)_            | `ticket.closed` was emitted by the earlier `update`  |

A state is closed when `state.type` is `completed` or `canceled`.

## Subject Fields

| CDEvent Field        | Linear Source                            | Notes                                                                                                 |
| -------------------- | ---------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| `subject.id`         | `data.url` without the title slug        | `https://linear.app/<workspace>/issue/<identifier>`                                                   |
| `content.summary`    | `data.title`                             |                                                                                                       |
| `content.ticketType` | `"issue"`                                | Linear has no issue type                                                                              |
| `content.uri`        | same as `subject.id`                     |                                                                                                       |
| `content.priority`   | `data.priorityLabel` (lowercased)        | `no priority`, `urgent`, `high`, `medium`, `low`                                                      |
| `content.creator`    | `data.creatorId`                         |                                                                                                       |
| `content.assignees`  | `[data.assigneeId]`                      | empty when unassigned                                                                                 |
| `content.labels`     | `data.labels[].name`                     |                                                                                                       |
| `content.milestone`  | `data.projectMilestone.name`             | or null                                                                                               |
| `content.resolution` | `data.state`                             | `completed` → `completed`; `canceled` named `Duplicate` → `duplicate`; other `canceled` → `withdrawn` |
| `context.timestamp`  | `createdAt` (time of the webhook action) |                                                                                                       |

`customData.linear` keeps the webhook envelope (`action`, `type`, `actor`, `updatedFrom`, `organizationId`, `webhookId`) and the issue identity (`id`, `identifier`, `number`, `url`, `state`, `team`, `project`, `cycle`).

## Linear setup

1. In Linear: **Settings → API → Webhooks → New webhook**.
2. Set the URL to `https://<your-collector-host>/webhook/000-linear`.
3. Select the **Issues** data change events.
4. Copy the signing secret into the collector config (see `cdviz-collector.toml`).

## Configuration

See the commented example at the top of [`cdviz-collector.toml`](./cdviz-collector.toml).

```toml
[remote.transformers-community]
type = "github://cdviz-dev/transformers-community"

[transformers]
linear_webhook = { type = "vrl", template_rfile = "transformers-community:///linear_webhook/to_v0_5.vrl" }
```

## Testing

```bash
mise run test                   # check mode
mise run test -- --mode review  # review and update outputs/0_5
```

## Limits

- `subject.id` uses the issue identifier (`ENG-123`). When an issue moves to another team, Linear gives it a new identifier, so later events have a new `subject.id`. `customData.linear.issue.id` stays stable.

## References

- [Linear Webhooks](https://linear.app/developers/webhooks)
- [CDEvents Ticket Events](https://cdevents.dev/docs/)

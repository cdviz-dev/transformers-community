# linear_graphql_api

Transform pages of the [Linear GraphQL API](https://linear.app/developers/graphql) `issues` query into CDEvents.
Use it with an `http_polling` source, to backfill history, or in place of [`linear_webhook`](../linear_webhook/) when Linear cannot reach the collector.

Linear has no REST API: the source sends GraphQL `POST` requests and follows the `pageInfo.endCursor` pagination.

## Event Mapping

A polled issue is a state, not a change, so the transformer rebuilds the lifecycle events from the issue timestamps.

| Issue state                       | CDEvent(s) emitted                             | Timestamp                    |
| --------------------------------- | ---------------------------------------------- | ---------------------------- |
| any issue                         | `ticket.created.0.2.0`                         | `createdAt`                  |
| `completedAt` or `canceledAt` set | `ticket.created.0.2.0` + `ticket.closed.0.2.0` | `completedAt` / `canceledAt` |

No `ticket.updated` is emitted.

## Subject Fields

`subject.id` is the same as in [`linear_webhook`](../linear_webhook/README.md#subject-fields), so events from both transformers are about the same subject.

| CDEvent Field        | Linear Source                | Notes                                                                                                          |
| -------------------- | ---------------------------- | -------------------------------------------------------------------------------------------------------------- |
| `subject.id`         | `url` without the title slug | `https://linear.app/<workspace>/issue/<identifier>`                                                            |
| `content.summary`    | `title`                      |                                                                                                                |
| `content.ticketType` | `"issue"`                    | Linear has no issue type                                                                                       |
| `content.uri`        | same as `subject.id`         |                                                                                                                |
| `content.creator`    | `creator.id`                 |                                                                                                                |
| `content.resolution` | `completedAt`, `state`       | `ticket.closed` only: `completedAt` set → `completed`; canceled `Duplicate` → `duplicate`; other → `withdrawn` |

`customData.linear.issue` keeps `id`, `identifier`, `number`, `team` (and `state` for `ticket.closed`).

Only fields that do not change after creation are kept (`summary` is required by CDEvents).
Linear returns an issue again after each edit, and the collector computes `context.id` from the event content:
a changed field would give the same lifecycle step a new id, so a duplicate event.
Labels, assignees, priority, milestone, project and cycle are not polled: use [`linear_webhook`](../linear_webhook/) for them.

## Polling setup

See the commented source in [`cdviz-collector.toml`](./cdviz-collector.toml). The driver:

1. Requests issues with `updatedAt` in the poll time window (`ts_after` included .. `ts_before` excluded, millisecond precision), 100 per page, archived issues included.
2. Feeds each page back (`route = "both"`) and requests the next page while `pageInfo.hasNextPage` is true.
3. Holds the window on HTTP `400` and `429` (Linear GraphQL errors and rate limit): the next poll retries it.

| Setting                | Notes                                                                             |
| ---------------------- | --------------------------------------------------------------------------------- |
| `authorization` header | Personal API key as is (`lin_api_...`), or `Bearer <token>` for an OAuth token    |
| `ts_after`             | Set to a past date for a bounded backfill. Unset: the first poll reads all issues |
| `min_request_interval` | Linear allows 1500 requests per hour per API key: `3s` stays below it             |

## Limits

- **Title change**: `summary` is required, so a title edit gives a new `ticket.created` (and `ticket.closed`) event for the issue.
- **GraphQL errors with a `2xx` status**: the driver fails and the collector logs `driver feedback execution failed`, but the window still advances. The remaining pages of that window are not fetched: run a backfill (`ts_after`, `ts_before_limit`) for that period.
- **Do not run with `linear_webhook` at the same time**: both emit `ticket.created` and `ticket.closed` with different content and timestamps, so each step is stored twice. Use polling to backfill up to a date, then the webhook.

## Testing

```bash
mise run test                   # check mode
mise run test -- --mode review  # review and update outputs/0_5
```

`inputs/` holds sample responses of the query (not captured from a real workspace).

## References

- [Linear GraphQL API](https://linear.app/developers/graphql)
- [Linear pagination](https://linear.app/developers/pagination)
- [Linear rate limiting](https://linear.app/developers/rate-limiting)
- [cdviz-collector `http_polling` source](https://cdviz.dev/docs/cdviz-collector/sources/http_polling)

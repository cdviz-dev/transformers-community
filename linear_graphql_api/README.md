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

The fields are the same as in [`linear_webhook`](../linear_webhook/README.md#subject-fields), so events from both transformers are about the same subject:

- `subject.id` = issue `url` without the title slug (`https://linear.app/<workspace>/issue/<identifier>`).
- `content.resolution` (`ticket.closed` only): `completedAt` set → `completed`; canceled state named `Duplicate` → `duplicate`; other canceled → `withdrawn`.

`customData.linear.issue` keeps `id`, `identifier`, `number`, `url`, `team`, `project`, `cycle` (and `state` for `ticket.closed`).

## Polling setup

See the commented source in [`cdviz-collector.toml`](./cdviz-collector.toml). The driver:

1. Requests issues with `updatedAt` in the poll time window (`ts_after` .. `ts_before`), 100 per page, archived issues included.
2. Feeds each page back (`route = "both"`) and requests the next page while `pageInfo.hasNextPage` is true.

| Setting                | Notes                                                                             |
| ---------------------- | --------------------------------------------------------------------------------- |
| `authorization` header | Personal API key as is (`lin_api_...`), or `Bearer <token>` for an OAuth token    |
| `ts_after`             | Set to a past date for a bounded backfill. Unset: the first poll reads all issues |
| `min_request_interval` | Linear allows 1500 requests per hour per API key: `3s` stays below it             |

The same issue is returned again each time it is updated. The collector computes `context.id` from the event content, so a re-polled event with unchanged content gets the same id. A change of title, labels, assignee, ... makes a new event for the same lifecycle step.

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

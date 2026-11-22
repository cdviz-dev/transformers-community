# linear_graphql_api

Transform pages of the [Linear GraphQL API](https://linear.app/developers/graphql) `issues` query into CDEvents.
Use it with an `http_polling` source, to backfill history, or in place of [`linear_webhook`](../linear_webhook/) when Linear cannot reach the collector.

Linear has no REST API: the source sends GraphQL `POST` requests and follows the `pageInfo.endCursor` pagination.

## Event Mapping

A polled issue is a state, not a change: the transformer rebuilds the lifecycle events from the issue timestamps.
Each event is emitted only when its timestamp is in the poll time window (`ts_after` included .. `ts_before` excluded).
So each lifecycle step is emitted once, even when Linear returns the issue again after later edits.

| Condition                                                       | CDEvent Type           | Timestamp                    |
| --------------------------------------------------------------- | ---------------------- | ---------------------------- |
| `createdAt` in the window                                       | `ticket.created.0.2.0` | `createdAt`                  |
| `completedAt` or `canceledAt` in the window                     | `ticket.closed.0.2.0`  | `completedAt` / `canceledAt` |
| `updatedAt` in the window, not equal to the created/closed time | `ticket.updated.0.2.0` | `updatedAt`                  |

Without window metadata (for example with `cdviz-collector transform`), every event is emitted.

Polling is an approximation of the history:

- the content of every event is the issue state at poll time (for example, `ticket.closed` shows labels as they were at the first poll after closing);
- several edits between two polls give one `ticket.updated`, with the last state.

## Subject Fields

The fields are the same as in [`linear_webhook`](../linear_webhook/README.md#subject-fields), so events from both transformers are about the same subject.

| CDEvent Field        | Linear Source                | Notes                                                                                                          |
| -------------------- | ---------------------------- | -------------------------------------------------------------------------------------------------------------- |
| `subject.id`         | `url` without the title slug | `https://linear.app/<workspace>/issue/<identifier>`                                                            |
| `content.summary`    | `title`                      |                                                                                                                |
| `content.ticketType` | `"issue"`                    | Linear has no issue type                                                                                       |
| `content.uri`        | same as `subject.id`         |                                                                                                                |
| `content.priority`   | `priorityLabel` (lowercased) | `no priority`, `urgent`, `high`, `medium`, `low`                                                               |
| `content.creator`    | `creator.id`                 |                                                                                                                |
| `content.assignees`  | `[assignee.id]`              | empty when unassigned                                                                                          |
| `content.labels`     | `labels.nodes[].name`        |                                                                                                                |
| `content.milestone`  | `projectMilestone.name`      | or null                                                                                                        |
| `content.resolution` | `completedAt`, `state`       | `ticket.closed` only: `completedAt` set → `completed`; canceled `Duplicate` → `duplicate`; other → `withdrawn` |

`customData.linear.issue` keeps `id`, `identifier`, `number`, `url`, `state`, `team`, `project`, `cycle`.

## Polling setup

See the commented source in [`cdviz-collector.toml`](./cdviz-collector.toml). The driver:

1. Requests issues updated since `ts_after` (included, millisecond precision), 100 per page, archived issues included.
   There is no upper bound: Linear knows only the last `updatedAt` of an issue, so an upper bound would drop an issue edited again before the query runs, and lose its events of the window.
   The transformer emits only the events in the window. An issue edited during paging can come back twice in the same poll: at worst a rare duplicate event, not a lost one.
2. Feeds each page back (`route = "both"`) and requests the next page while `pageInfo.hasNextPage` is true.
3. Holds the window on HTTP `400` and `429` (Linear GraphQL errors and rate limit): the next poll retries it.

| Setting                | Notes                                                                             |
| ---------------------- | --------------------------------------------------------------------------------- |
| `authorization` header | Personal API key as is (`lin_api_...`), or `Bearer <token>` for an OAuth token    |
| `ts_after`             | Set to a past date for a bounded backfill. Unset: the first poll reads all issues |
| `min_request_interval` | Linear allows 1500 requests per hour per API key: `3s` stays below it             |

## Limits

- **Team move**: Linear gives a moved issue a new identifier, so a new `subject.id`. Its `createdAt` is in an older window, so the new `subject.id` gets no `ticket.created`, only its later events.
- **Backfill start**: an issue created before the first `ts_after` gets no `ticket.created` (only its later events). Leave `ts_after` unset on the first run to read the full history.
- **GraphQL errors with a `2xx` status**: the driver fails and the collector logs `driver feedback execution failed`, but the window still advances. The remaining pages of that window are not fetched: run a backfill (`ts_after`, `ts_before_limit`) for that period.
- **Do not run with `linear_webhook` at the same time**: both emit `ticket.created` and `ticket.closed` with different content and timestamps, so each step is stored twice. Use polling to backfill up to a date, then the webhook.

## Testing

```bash
mise run test                         # check mode, all tasks
mise run test:0_5 -- --mode review    # review and update outputs/0_5 (no window: every event)
mise run test:window -- --mode review # review and update outputs/0_5_window (fixed window)
```

`inputs/` and `inputs_window/` hold sample responses of the query (not captured from a real workspace).
`cdviz-collector.test-window.toml` sets a fixed window in `metadata`, as `http_polling` does, because `cdviz-collector transform` provides none.

## References

- [Linear GraphQL API](https://linear.app/developers/graphql)
- [Linear pagination](https://linear.app/developers/pagination)
- [Linear rate limiting](https://linear.app/developers/rate-limiting)
- [cdviz-collector `http_polling` source](https://cdviz.dev/docs/cdviz-collector/sources/http_polling)

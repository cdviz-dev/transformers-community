# Jenkins Builds Transformer

Transform Jenkins build history, pulled via the Jenkins Remote API, into CDEvents (0.5).

## Overview

Jenkins has no first-class outbound webhook for build events (only optional plugins, which vary by
install), so this transformer is paired with cdviz-collector's [`http_polling` source](https://github.com/cdviz-dev/cdviz-collector/blob/main/src/sources/http_polling/README.md)
instead of a webhook. The source's `driver_vrl` script recursively **discovers** every job on the
Jenkins instance and polls each one's builds; this transformer converts each polled page into
`pipelinerun.started` / `pipelinerun.finished` CDEvents.

### Discoverability (no hardcoded job/pipeline names)

The driver starts at `{jenkins_base}/api/json?tree=jobs[name,url,_class]` and, for every job it
finds:

- if the job's `_class` looks like a **container** (contains `"folder"` or `"multibranch"` —
  covers `Folder`, `WorkflowMultiBranchProject`, `OrganizationFolder`, ...), it descends into it
  with another discovery request (`route: "feedback"`) — at any depth, not just one or two levels;
- otherwise it's a **buildable leaf job**, and its builds are fetched
  (`{job.url}api/json?tree=builds[number,url,timestamp,duration,result,building]`,
  `route: "pipeline"`).

New jobs, folders, and multibranch pipeline branches are picked up automatically on the next poll —
nothing in the config names a specific job.

> ponytail: leaf jobs are fetched with a single request — whatever builds Jenkins currently retains
> per the job's discard policy. There's no `{M,N}` range pagination for jobs with more build
> history than Jenkins keeps in memory. Upgrade path: add per-job range chaining (`route: "both"`,
> slicing `builds[...]{start,end}`), following the same shape as the http_polling README's
> "multi-pass" example.

### Job name / job url derivation

The driver doesn't need to pass job identity through polling state: the transformer derives it
directly from each build's own `url` (e.g.
`https://jenkins.example.com/job/team-a/job/service/job/main/44/`) by stripping the trailing build
number to get the job url, then flattening the Jenkins `/job/a/job/b/job/c/` path into
`a/b/c` for `pipelineName`. This works for a job at the Jenkins root or nested under any number of
folders/multibranch projects, and needs no cooperation from the driver.

### Event Mapping

| Build state                          | CDEvent Type           |
| ------------------------------------ | ---------------------- |
| `building = true` or `result = null` | `pipelinerun.started`  |
| `result` set                         | `pipelinerun.finished` |

Outcome mapping (on `finished`): `SUCCESS` → `success`, `ABORTED`/`NOT_BUILT` → `error`,
`FAILURE`/`UNSTABLE` → `failure`.

`subject.id` / `subject.content.uri` are the build's own URL; `customData.jenkins.build` carries
the raw polled build object, and `customData.jenkins.job_url` the derived job URL.

## Inputs

`inputs/examples/builds.json` is a `builds[...]` page as the http_polling driver would fetch it for
one leaf job — one running build and two finished builds (`SUCCESS`, `FAILURE`) under a nested
folder + multibranch-pipeline path, to exercise job-name derivation at depth.

The `transform` CLI doesn't feed `.metadata.ts_after`/`.metadata.ts_before` for local fixture runs
(those are populated by the live `http_polling` source), so the transformer defaults to an
effectively unbounded window when they're absent — every build in the fixture is emitted.

## Testing

```bash
# Check transformation against expected outputs (validates CDEvents)
mise run '//jenkins_builds:test'

# Review and update expected outputs after adding new inputs
mise run '//jenkins_builds:test' -- --mode review
```

## Transformer Configuration

The transformer is declared in `cdviz-collector.toml`:

```toml
[transformers]
jenkins_builds_0_5 = { type = "vrl", template_file = "./to_v0_5.vrl" }
```

**Source configuration** (see the full commented example in `cdviz-collector.toml`):

```toml
[sources.jenkins]
enabled = true
transformer_refs = ["jenkins_builds_0_5"]

[sources.jenkins.extractor]
type                 = "http_polling"
polling_interval     = "5m"
parser               = "json"
min_request_interval = "200ms"

driver_vrl = """
if .response == null {
    .requests = [{
        "url": "https://jenkins.example.com/api/json",
        "query": { "tree": "jobs[name,url,_class]" },
        "route": "feedback"
    }]
} else {
    reqs = []
    for_each(array(.response.body.jobs) ?? []) -> |_i, job| {
        class = downcase(to_string(job._class) ?? "")
        if contains(class, "folder") || contains(class, "multibranch") {
            reqs = push(reqs, {
                "url": string!(job.url) + "api/json",
                "query": { "tree": "jobs[name,url,_class]" },
                "route": "feedback"
            })
        } else {
            reqs = push(reqs, {
                "url": string!(job.url) + "api/json",
                "query": { "tree": "builds[number,url,timestamp,duration,result,building]" },
                "route": "pipeline"
            })
        }
    }
    .requests = reqs
}
"""

[sources.jenkins.extractor.headers]
Authorization = { type = "secret", value = "Basic BASE64_ENCODED_CREDENTIALS" }
```

`Authorization` is HTTP Basic auth, base64 of `username:api_token` (create an API token under
**Jenkins > Your User > Configure > API Token**). The account needs read access to every job you
want discovered (`Overall/Read`, `Job/Read`).

## Troubleshooting

### No jobs discovered

Confirm the credentials can read `{jenkins_base}/api/json?tree=jobs[name,url,_class]` directly
(`curl -u user:token ...`) — a 403 here means the account lacks `Overall/Read`.

### A folder's jobs never appear

Check its `_class` (via the same URL) actually contains `"folder"` or `"multibranch"`. Third-party
folder-like plugins with a different naming convention need an extra substring added to the
`contains(...)` check in `driver_vrl`.

### Builds missing beyond a certain age

Jenkins only returns builds it currently retains (per job discard policy) in a single `builds[...]`
request — see the pagination note above.

## References

- [Jenkins Remote access API](https://www.jenkins.io/doc/book/using/remote-access-api/)
- [cdviz-collector http_polling source](https://github.com/cdviz-dev/cdviz-collector/blob/main/src/sources/http_polling/README.md)
- [CDEvents Specification](https://cdevents.dev/)
- [VRL Documentation](https://vector.dev/docs/reference/vrl/)

## Design Decisions

### `_class`-based container detection instead of a job-type allowlist

Jenkins has many plugins that add job or folder types; matching on `_class` substrings
(`"folder"`, `"multibranch"`) is much less brittle than hardcoding an allowlist of exact class
names, at the cost of possibly missing an exotic container plugin (see Troubleshooting above).

### Deriving job identity from the build URL, not driver state

http_polling's `driver_vrl` state is a _single_ snapshot cloned into every request emitted by one
driver invocation — it can't carry different data to different fanned-out requests (e.g. one
discovery response fanning out into many different jobs' builds requests). Deriving job
name/url from each build's own URL in the transformer sidesteps that limitation entirely, and keeps
the driver itself stateless.

### Event Coverage

**Supported**: pipeline run started (build in progress) and finished (build completed), for
freestyle jobs, pipeline jobs, and multibranch pipeline branches at any folder depth.

**Not supported**: an explicit `pipelinerun.queued` event (Jenkins' builds API doesn't expose a
queue timestamp distinct from build start), and per-stage `taskrun` events (would require polling
each build's `wfapi/describe` endpoint — a further multi-pass extension, not implemented here).

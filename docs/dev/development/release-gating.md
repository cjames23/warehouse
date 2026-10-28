# Release gating

Release gating is a reusable hold on a `Release`. While a gate blocks, the
release is not installable: it is absent from the Simple index and the JSON
API, and its artifacts are not in the bucket the CDN serves from.

This is only the mechanism. **Warehouse registers no gates.** With nothing
registered the mechanism is inert and uploads publish exactly as they did
before it existed. A feature that needs to withhold releases brings its own
`IGatePolicy`.

## Why there are two enforcement points

Keeping a release out of the Simple index makes it undiscoverable: no installer
can resolve a version it cannot see. That alone is not the same as making the
artifact unavailable, because file URLs are served directly by the CDN from a
storage bucket and are not checked against the database on each request.

So a gate enforces in two places:

| Enforcement point | What it prevents | Where |
|---|---|---|
| Index and API exclusion | Resolution: `pip install foo` cannot select the version | `warehouse/packaging/utils.py`, `warehouse/legacy/api/json.py` |
| Deferred cache sync | Fetching: the artifact is not in the CDN-backed bucket at all | `warehouse/gating/services.py`, `warehouse/packaging/tasks.py` |

An upload always writes to *archive* storage synchronously. Normally the upload
view then dispatches `sync_file_to_cache`, which copies the file into the cache
bucket and is what actually publishes it. For a gated release that dispatch is
withheld, and `IGateService` performs it when the last gate clears.

Three other code paths could otherwise copy a gated file into the cache bucket
and publish it by accident. All three check `Release.gated`:

- `sync_file_to_cache` itself, for any caller that dispatches it directly.
- `reconcile_file_storages`, whose whole job is closing cache gaps. A
  gated file is an intentional gap, not a gap to close.
- `check_file_cache_tasks_outstanding`, which gauges uncached files to alert on
  a stuck queue; counting gated files would make it alert on normal gating.

## Vocabulary

A `ReleaseGate` row is one hold, identified by `(release_id, kind)`.

`kind` is the name of the `IGatePolicy` that owns the hold. It is a plain
string, not a database enum, which is what makes the set of gates open: adding
one is a code change in one package, with no migration against `release_gates`.
It is validated at registration, which happens at application startup, rather
than at insert time, so a malformed policy fails the deployment instead of an upload.

`GateStatus` is the state of the hold:

| Status | Blocks? | Meaning |
|---|---|---|
| `pending` | yes | Not yet decided |
| `failed` | yes | Decided unfavorably; needs an administrator |
| `passed` | no | Decided favorably by the policy |
| `bypassed` | no | An administrator overrode the policy |
| `expired` | no | The policy never decided and the gate failed open |

Only `pending` and `failed` block. Everything else lets the release through, so
a newly added status defaults to non-blocking and cannot silently strand
releases that were already published.

Gates are never deleted when they resolve, so a release carries an auditable
record of what held it and why it was let go. A consequence worth knowing: gate
rows outlive the policy that created them, so a `kind` is effectively permanent
and renaming one orphans its history.

`details` is a JSONB column the mechanism never reads. It is there so a policy
can attach whatever evidence drove its decision without the generic schema
having to know the shape.

### `Release.gated`

`Release.gated` is denormalized from the gate rows. The Simple index is one of
the hottest queries in the application, and joining it against gate rows on
every request is not worth the cost, so the read paths filter on this single
indexed boolean instead.

It is only ever written through `IGateService`, which recomputes it from the
gates whenever one resolves. Writing it directly will desynchronize it from the
gate rows.

## Adding a gate

Implement `IGatePolicy` and register it. That is the whole extension point:
the mechanism does not change and neither does the schema.

```python
@implementer(IGatePolicy)
class MyPolicy:
    kind = "my-gate"
    timeout = datetime.timedelta(hours=2)
    timeout_status = GateStatus.Expired

    def applies(self, release, request):
        return True  # called on every upload; must be cheap and side-effect free
```

```python
def includeme(config):
    config.register_gate_policy(MyPolicy())
```

Register it from a package included *after* `.gating` in
`warehouse/config.py`, since `register_gate_policy` is a directive that package
adds.

A policy decides only *whether a release needs this hold* and *how long it may
stay pending*. It does not resolve its own gates; whatever does the work (a
Celery task, a webhook) calls `IGateService.resolve`.

Nothing stops a policy from resolving its gate the instant it has an answer,
and that is usually the right design: the timeout is then a *ceiling* on the
hold rather than a waiting period, and in the normal case it never fires.

### Timeouts

`resolve_timed_out_gates` runs every minute and is the backstop that stops a
stalled decision from withholding a release forever. `timeout_status` says what
elapsing means for that policy:

- `Passed`: waiting *is* the mechanism, so the window elapsing is the decision.
- `Expired`: fail open after never getting an answer. Distinct from `Passed`
  so the audit trail separates "decided favorably" from "gave up waiting", and
  so the metric on it is alertable.
- `Failed`: fail closed, keep withholding.

A policy with `timeout = None` waits indefinitely.

A pending gate whose kind has no registered policy (a retired kind, or a
feature switched off) is expired by the same sweep, because nothing is left
that could decide it. For the same reason `IGateService.resolve` is keyed on
the gate row rather than on the registry: retiring a policy must not make its
outstanding gates unresolvable.

### Reopening

`open_gates` runs on *every* upload, not only on new releases. A file added to
an already-cleared release reopens that release's gates, because the earlier
decision was about the files that existed at the time. The unique constraint on
`(release_id, kind)` means the existing row is reset rather than duplicated.

## Administrative override

`/admin/gating/` lists every gate currently withholding a release and offers a
bypass. This exists because a `failed` gate is the one state nothing will
resolve on its own. Without it, a release could be withheld permanently with no
way to see why.

A bypass is recorded as `bypassed` rather than `passed`, with the
administrator's username in `resolved_by`, so the audit trail keeps the
distinction between a policy clearing a release and a person overriding it.

## Metrics

| Metric | Meaning |
|---|---|
| `warehouse.gating.gates` | Gauge of gate rows by kind and status |
| `warehouse.gating.timed_out` | A gate was resolved by the timeout sweep |
| `warehouse.upload.gated` | An upload was withheld |

A rising `warehouse.gating.timed_out` with `status:expired` means releases are
being published without the decision their gate was waiting for.

## Trying it locally

With no policies registered there is nothing to observe, so register one first.
A policy that holds every release until the timeout elapses is enough to watch
the mechanism work end to end.

```shell
make serve
```

Upload a distribution to the local instance. The release is in the database and
visible on the project page, but absent from `/simple/<name>/` while its gate
is pending:

```shell
curl -s http://localhost:80/simple/<name>/ | grep <version>
```

```sql
SELECT r.version, g.kind, g.status, g.resolved, g.note
FROM release_gates g JOIN releases r ON r.id = g.release_id
ORDER BY g.created DESC;
```

Taking the gate to `failed` is the case worth exercising by hand, since that is
the state that needs the admin bypass at `/admin/gating/`.

# octoDNS provider bugs found during GCore→ClouDNS/rcode0 NS migration

Found 2026-09-20 while executing a staged NS-consolidation and GCore-removal
plan on the `octodns` repo (`inventory/production.yaml`, `config/*.yaml`).
Both bugs were hit performing routine, low-risk operations (a TTL-only
change, and lowering root-NS TTL) — not edge cases. Filed here for
porting into our forks and upstreaming.

---

## 1. `octodns-gcore` 1.0.0 — root NS update sends an empty rrset

**Package:** `octodns-gcore`
**Version affected:** 1.0.0 (confirmed present; not checked further back than
0.0.5, where root-NS support was introduced)
**Severity:** High for any zone GCore is actually authoritative for — a
root-NS *update* can silently attempt to push an empty `NS` rrset, which
GCore's API happens to reject (`"rrset must contain at least one record"`),
but a less strict API or a future relaxation on GCore's side would not save
us here.

### Background

Root NS management was added deliberately in `octodns-gcore` v0.0.5
via [issue #48](https://github.com/octodns/octodns-gcore/issues/48)
(filed by us: `istr`) and
[PR #49](https://github.com/octodns/octodns-gcore/pull/49). The PR is a
single-line change:

```python
SUPPORTS_ROOT_NS = True
```

No new update-handling logic was added — it reuses the provider's existing
generic `NS`-type code path (`_data_for_NS`, `_params_for_NS`,
`_apply_update`). The PR's test diff only exercises **creating** a root NS
rrset (a `POST`); there is no test coverage for **updating** an
already-existing one (a `PUT`) — which is exactly the path we hit.

### Repro

1. A zone already has a root NS rrset on GCore (from an earlier sync or a
   pre-existing zone).
2. `config/<zone>.yaml` has a root `NS` record whose **content differs**
   from what's currently on GCore (in our case: GCore's copy was stale,
   listing `ns.mx-is.com.`/`ns.mx-is.de.`/`ns.mx-is.net.`/`ns.mx-is.org.`,
   while config had already moved to `ns1.mx-is.com.`/`ns1.mx-is.de.`/
   `ns.mx-is.net.`/`ns.mx-is.org.`).
3. Run `octodns-sync --doit --force` (force is required regardless, since
   octoDNS core's `Plan.raise_if_unsafe()` always flags root-NS changes).
4. Observed:

   ```
   ERROR  GCoreProvider[gcore] bad request {'ttl': 300, 'resource_records': []}
   has been sent to '.../zones/mx-is.de/mx-is.de./NS':
   {"exception":"validation_error","message":"validation error",
    "error":"rrset must contain at least one record", ...}

   octodns_gcore.GCoreClientBadRequest: ...
   ```

   `resource_records` is empty despite the desired `NS` record clearly
   having 4 values in the octoDNS plan.

### What we know, what we don't

- Confirmed: GCore's API itself fully supports `NS` at the zone apex for
  both create and update (`docs.gcore.com` RRset reference lists `NS`
  as a normal type, update endpoint just requires a non-empty
  `resource_records`). This is **not** a GCore platform limitation.
- Confirmed: the failure is specific to the **update** path — a `POST`
  (create) was tested upstream; a `PUT` against an existing root NS rrset
  was not.
- Not yet isolated: exactly where the value list becomes empty between
  `Plan` construction (`change.new.values` has 4 entries, visible in the
  printed plan) and `_params_for_NS(new)` being called inside
  `_apply_update`. Candidates to check first:
  - Whether `_process_existing_zone`/`_process_desired_zone`'s interaction
    with `SUPPORTS_ROOT_NS` + `strict_supports: false` (set in our config)
    mutates the record in a way that clears `.values` for this specific
    type in this specific (update, not create) code path.
  - Whether `Change` construction for root-NS specifically goes through a
    different path than other `Update` changes (e.g. via
    `_extra_changes`, or some root-NS-specific branch not visible in a
    surface read of `__init__.py`).
- No live damage occurred: GCore's own validation rejected the empty
  payload before writing anything, and the affected zone (`mx-is.de.`)
  was not actually delegated to GCore in the real world anyway (an inert
  shadow copy). This bug would be materially more dangerous against a zone
  GCore is genuinely authoritative for, if GCore's API were laxer about
  empty rrsets, or if the empty-payload bug can also surface as a
  `DELETE`-then-empty-`POST` pair instead of a rejected `PUT` in some other
  code path we haven't hit yet.

### Roadmap

1. **Reproduce with a unit test**, using the existing `requests_mock`-based
   test harness style already in the repo (see `PR #49`'s test diff for a
   template). Minimal case: existing zone has a root NS rrset with values
   `[a, b]` at ttl 3600; desired has the **same or different** values at a
   different ttl. Assert the `PUT` payload's `resource_records` is
   non-empty and correct in both cases (TTL-only and content-changed).
2. **Bisect the empty-payload path** with the failing test from (1) — add
   targeted `pdb`/logging around `_process_desired_zone`,
   `_process_existing_zone`, and `_apply_update` to find exactly where
   `.values` gets dropped for root NS specifically (vs. non-root NS, which
   presumably works fine — worth a comparative test too).
3. **Fix + regression test.** Once root-caused, the fix is likely small
   (this is the same shape of bug as #48/#49: an under-tested one-liner
   class of issue). Add explicit update-path test coverage for root NS
   alongside the existing create-path test, so this class of regression
   can't recur silently again.
4. **Upstream**, referencing #48/#49 for context. Given we're the original
   requester of root-NS support, this is worth doing regardless of our own
   migration timeline — other `octodns-gcore` users managing root NS via
   octoDNS are exposed to the same bug today, with 0 open issues currently
   reported.
5. **No urgency for our own repo**: GCore has already been fully disabled
   as a sync target as part of the migration (see commit `5a4234a`) and is
   being decommissioned outright, so this fix is upstream-community value,
   not a blocker for us.

---

## 2. `octodns-cloudns` 0.0.17 — TTL-only updates silently no-op

**Package:** `octodns-cloudns`
**Version affected:** 0.0.17 (current latest at time of writing)
**Severity:** High — this is a **silent** failure: `octodns-sync --doit
--force` reports success (`N total changes`, exit code 0) while making
**zero** API calls and changing nothing. There is no error, warning, or
any other signal. Anyone trusting the sync's reported exit status/change
count would believe the change landed.

### Root cause

`ClouDNSProvider._apply_update` (`octodns_cloudns/__init__.py:577`) diffs
purely on record **value**, never on TTL:

```python
def _apply_update(self, change):
    existing = change.existing
    zone = existing.zone
    records = self._records_are_same(existing)
    try:
        to_delete = set(existing.values).difference(change.new.values)
        to_create = set(change.new.values).difference(existing.values)
    except TypeError:
        replace_all = True
    else:
        replace_all = any('record' not in record for record in records)

    if replace_all:
        for record in records:
            self._client.record_delete(zone.name[:-1], record['id'])
        self._apply_create(change)
        return

    for record in records:
        if record['record'] in to_delete:
            self._client.record_delete(zone.name[:-1], record['id'])

    stripped_change = Change(existing=existing, new=change.new)
    stripped_change.new.values = to_create
    self._apply_create(stripped_change)
```

When only `ttl` changes and the value set is identical, `to_delete` and
`to_create` are both **empty sets**. `_apply_create` is then called with
`new.values = []`; its own loop (`for value in new.values: ...`) iterates
zero times. Net effect: **no HTTP calls at all**, for any record type that
goes through the hashable-value branch (i.e. everything except CAA/
NAPTR/SSHFP/SRV, which have their own structured-value comparison in
`_records_are_same` and hit `replace_all = True` instead — untested by us
whether *that* path handles TTL-only changes correctly, see Roadmap).

### Why this is worse than it first looks

- Structural: `ClouDNSClient` (the HTTP wrapper class) only implements
  `record_create` (`dns/add-record`), `record_delete`
  (`dns/delete-record`), and reads (`dns/records`, `dns/get-zone-info`).
  There is **no `record_update`/`dns/mod-record` method at all** — the
  provider was built entirely around delete+recreate, never around
  in-place editing. ClouDNS's API does document a `dns/mod-record`
  endpoint for editing an existing record (including TTL) without
  deleting it first; this provider simply never wires it up. This is the
  deeper reason a "TTL-only" update degenerates into a no-op instead of a
  small PATCH: there's no PATCH-shaped code path in the client at all,
  only "diff values, delete removed ones, create added ones" — and TTL
  isn't a value.
- We have **direct evidence this has already caused silent production
  drift**, independent of anything we were doing: `config/deflect.me.yaml`
  specified TTLs of 600/3600/86400 for various records, but ClouDNS was
  live-serving a flat `300` for all of them (confirmed via `dig` against
  ClouDNS's own authoritative server). Cross-checked against GCore's
  (stale but untouched-since-creation) copy of the same zone, which
  matched the *committed config's* TTLs exactly — strongly suggesting a
  past TTL-only change to `deflect.me` was applied via octoDNS at some
  point, reported success, and silently never took effect on ClouDNS.
  Nobody noticed because the record *values* were still correct — only
  the TTL was stuck.
- We confirmed the no-op empirically with `--debug`: applying a TTL-only
  change to `ingo-struck.de.`'s root NS record made exactly 2 HTTP
  requests, both `GET` (list records, get-zone-info) — no `POST`/`PUT`/
  `DELETE` — yet `octodns-sync` printed `1 total changes` and exited 0.

### What we explicitly want to avoid in the fix

Per team preference: **do not** "fix" this by routing TTL-only changes
through the existing delete+recreate (`replace_all`) machinery just
because it's already there and would technically work. ClouDNS's API has
a real edit/PATCH-equivalent endpoint (`dns/mod-record`); the fix should
use it, not paper over the missing update path with more brute-force
delete/recreate.

### Roadmap

1. **Add `record_update` to `ClouDNSClient`**, wrapping
   `dns/mod-record` (per ClouDNS's public API docs — needs a fresh check
   against current docs for exact required/optional params: domain-name,
   record-id, host, record, ttl, etc.). This is the missing primitive;
   everything else builds on it.
2. **Fix `_apply_update`'s diff to consider TTL, not just value:**
   - When the value set is unchanged but TTL differs: call the new
     `record_update` once per existing record id with the new TTL,
     instead of falling through the empty-set no-op path.
   - When both value and TTL differ: prefer `record_update` per matched
     record (edit in place) over delete+create where the API allows it,
     reserving `record_delete`+`record_create` for records that are
     genuinely being added or removed. This directly addresses the
     "don't brute-force when PATCH exists" concern, not just for TTL.
   - Leave the structured-value types (CAA/NAPTR/SSHFP/SRV,
     `replace_all` branch) as a separate investigation — check whether
     they have the same TTL-blind-spot under a different symptom (they
     currently always delete+recreate, so they likely don't no-op, but
     should be confirmed with a targeted test rather than assumed safe).
3. **Regression test**: a TTL-only update (value set unchanged) must
   result in at least one `record_update`/`dns/mod-record` API call and a
   changed TTL in the mocked response, for at least one type from each of
   the two existing comparison branches (hashable-value and
   `replace_all`).
4. **Loudness, as defense in depth**: independent of the real fix,
   consider whether `octodns-sync`'s reported "N total changes" should be
   based on confirmed API calls actually made rather than
   `len(plan.changes)` — so that a future regression of this shape fails
   loudly (wrong count / exception) instead of silently. This is more of
   an octoDNS-core discussion than `octodns-cloudns` specifically, but
   worth raising given how easy this was to miss.
5. **Audit for existing drift**: before/alongside the fix, worth a
   one-time sweep (`octodns-compare` config vs. cloudns, or a live `dig`
   sweep of TTLs vs. config) across all ClouDNS-managed zones in this
   repo, since `deflect.me` is concrete proof this bug has already caused
   silent config drift at least once, undetected until now.
6. **Upstream** once fixed and tested, since — like the GCore bug — this
   silently affects any `octodns-cloudns` user doing a TTL-only change,
   which is about as routine an operation as exists.

### Workaround used in this repo (interim, not a fix)

While migrating NS records (commit `5a4234a`), TTL-only changes were applied
directly via ClouDNS's dashboard/API instead of through `octodns-sync`,
then verified against both the ClouDNS primary and the rcode0 AXFR
secondary via live `dig` before committing the corresponding config
changes. This is not scalable and shouldn't be treated as an acceptable
long-term process — it's exactly the kind of manual step octoDNS exists
to remove.

---

## Summary table

| | Package | Symptom | Danger | Status here |
|---|---|---|---|---|
| 1 | `octodns-gcore` 1.0.0 | Root-NS *update* sends empty `resource_records`, API rejects it | High if GCore's own validation were looser | GCore fully disabled as a target (`5a4234a`); being decommissioned |
| 2 | `octodns-cloudns` 0.0.17 | TTL-only update silently no-ops (0 API calls), reports success | High — undetectable without manual verification; already caused real drift (`deflect.me`) | Worked around manually per-change; needs the real fix upstream |

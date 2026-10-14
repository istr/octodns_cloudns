# Issue and PR plan for [ROADMAP.md](ROADMAP.md)

Status: v2, 2026-10-14. Supersedes v1 (2026-09-20, preserved in tag
[`roadmap-v1`](https://github.com/istr/octodns_cloudns/blob/roadmap-v1/ISSUE-PLAN.md)), which
proposed one tracking issue, 41 bug and task issues, four fork issues and 24 PRs.

Baseline: upstream `ClouDNS/octodns_cloudns` `main` at `2c45212` (`0.1.0.dev0`). Fork
`istr/octodns_cloudns` `main` is that commit plus the three planning documents. Every PR branches
from `upstream/main`, not from the fork's `main`, and is opened against `ClouDNS/octodns_cloudns`.

## 1. What changed, and what it means for filing

The maintainer (boyanpeychev) answered in
[#25](https://github.com/ClouDNS/octodns_cloudns/issues/25#issuecomment-6100465234) on 2026-10-10 and then did
a large part of v1 himself (details in [ROADMAP.md](ROADMAP.md) section 0):

- **0.0.18 is released** (#28, 2026-10-11): the TTL hotfix, safe request encoding, redaction, error
  propagation, desired-plan immutability, credential removal, live validation. v1's phase 0 (repo
  side) and phase 2 are complete, and so are v1 issues B1, B2, B8, B9, T10 and T11 and PR-0.1 and
  PR-2.1 to PR-2.4.
- **D1, D5 and DS are done** (#26, #41). #23, #24 and #14 are closed.
- **Filing:** "keep #25 as the umbrella tracker and use a small number of focused issues: TTL
  hotfix, tooling/compatibility, API client, codecs, and reconciliation/provider integration.
  Reuse #23 and #14." The TTL hotfix shipped as #28 without a separate issue; #23 and #14 closed
  with it and with #41. That leaves **four new upstream issues (A to D)**. v1's B\*/T\* findings
  become checklist items inside them and in PR descriptions; none is filed on its own.
- **Milestones:** `0.1.0` exists and is empty. We cannot assign milestones or labels (no triage
  permission), so the #25 comment asks the maintainer to put A to D on `0.1.0`.
- **Requirements** that each issue must carry, quoted from #25:
  - small, independently tested PRs; formatting kept separate;
  - fail safely on unsupported GeoDNS and unparseable managed rows;
  - preserve intentionally inactive records by default, explicit opt-in reactivation;
  - reject unsupported TTLs at planning time by default, rounding opt-in;
  - rate limiting accounts for shared egress IPs, not only shared credentials;
  - no blind retries after ambiguous write outcomes;
  - uncertain API behaviour validated in an isolated test zone, captured fixtures sanitised;
  - release checks include read-back verification and an empty second plan after apply;
  - AI-assisted development is acceptable; contributors stay responsible for correctness, evidence
    and human review;
  - release ownership stays with ClouDNS; milestones imply no dates.

### 1.1 Fork issues

Only one fork issue remains: **F1**, running probes P3, P4, P6, P8 to P14 against our own isolated
zone and sanitising the fixtures (account details stay out of upstream). v1's other fork issues are
dropped: F2 (fork tags) because ClouDNS releases promptly and owns releases; F3 (drift sweep)
belongs in the octoDNS operations repository; F4 (contingency if upstream stalls) because upstream is
anything but stalled.

### 1.2 Prerequisites

1. Everything on GitHub (comments, issues, PRs) is posted by hand from the payloads in this file;
   no `gh` and no API tokens are used. Local work uses plain git; fork pushes go over SSH.
2. Fork `main` carries ROADMAP v2 and this file, so the links below resolve. Done when this is
   pushed.
3. Issues A to D are filed before the #25 comment, so the comment can name their numbers.

## 2. Comment on #25 (ready to paste)

````markdown
@boyanpeychev thanks for the decisions, and for shipping 0.0.18 and DS yourself. That covers a
large part of what I had planned for the first weeks, and `ds.py` is exactly the shape I had in mind
for the rewrite (canonical keys, ids kept for equal values, leftovers paired into `mod-record`, adds
before mods before deletes), so I will generalise it rather than build something parallel.

I re-baselined the roadmap on current `main` (2c45212):
https://github.com/istr/octodns_cloudns/blob/main/ROADMAP.md (v1 is kept as tag `roadmap-v1`, so
the snapshot above still resolves). Section 0 lists what changed; section 2 has the status of every
v1 finding on current `main` (9 fixed, 8 partly fixed, 12 open); your requirements are applied in
section 5 (inactive rows preserved by default, rate limiting per egress IP, no retry of ambiguous
writes, fail-safe GeoDNS and unparseable rows, TTL rejection at plan time with opt-in rounding).

As you asked, I am filing four focused issues for the remaining 0.1.0 work, since the TTL hotfix is
done:
- A: tooling and compatibility
- B: API client
- C: codecs
- D: reconciliation and provider integration (includes the live read-back / empty-second-plan
  release check)

Could you put them on milestone `0.1.0`? I cannot set milestones or labels.

Four smaller questions (details in roadmap section 7.2):
- D7: do you want a 0.0.19 that only makes the remaining legacy-path failures loud? A Delete that
  matches no API row (CAA, LOC) and an Update whose removals match nothing (NS, MX, SRV, ...) still
  succeed silently in 0.0.18, and TLSA updates and deletes crash. Guards only, no new behaviour.
- D8: when a whole name/type is deleted, should its inactive rows stay (my suggestion, consistent
  with "preserve by default") or go?
- D9: GeoDNS zones whose managed rows are all at DEFAULT: fail the zone in 0.1.0 (my suggestion), or
  manage them as static records while always sending `geodns-location`?
- D10: the formatting-only PR touches every file. When is a quiet window on `main` for it, so it
  does not collide with your own work?
````

## 3. Issue bodies (ready to paste)

Each ends with `Refs #25`. Links are pinned to `2c45212` so line anchors stay valid.

### 3.1 Issue A — tooling and compatibility

**Title:** `[0.1.0] Tooling and compatibility: pyproject.toml, lint, formatting, script/cibuild in CI`

````markdown
Part of the 0.1.0 work tracked in #25 (roadmap section 6, track A). The octoDNS / Python floor
bump is already on `main` (#41); this issue covers the rest of the repository tooling, so later PRs
can be checked by `script/cibuild` like other octoDNS providers.

### Current state (2c45212)

- `script/cibuild` fails all four checks: black (10 files at `-S -l 80`), isort (5 test files),
  pyflakes (4 findings: unused `zone` in `_apply`, two unused test imports, a shadowed `call` in
  `tests/test_ttl_updates.py`), and the 100 % coverage gate (86 % today).
- CI runs bare `pytest`, not `script/cibuild`.
- `setup.py`: `url` points to `github.com/octodns/octodns-cloudns` (404, also shown on PyPI);
  deprecated `tests_require`.
- No `pyproject.toml`; `script/bootstrap` links a missing `.git_hooks_pre-commit`; `script/format`
  ignores `VENV_NAME`; `script/cibuild-setup-py` uses `setup.py install`.
- `tests/config.yaml` (wrong class path) and `tests/config/recordsfortest.bg.yaml` are unused.
- No secret scanning, although a credential was committed once (#23).

### Plan (three PRs, formatting separate)

- [ ] PR-A1 packaging and lint, no reformatting: `pyproject.toml` with the octodns-org black, isort and
      pytest configuration; `url` fix; `tests_require` dropped; `.git_hooks_pre-commit` shipped;
      script fixes; the four pyflakes findings; the two unused test configs removed; a secret
      scanner (gitleaks) in CI and the pre-commit hook.
- [ ] PR-A2 formatting only: `script/format` output and nothing else, its commit hash in
      `.git-blame-ignore-revs`. Landed in a window agreed in #25, because it touches every file.
- [ ] PR-A3 CI runs `script/cibuild` and `script/cibuild-setup-py`. The minimum-version legs stay
      (Python 3.10 and 3.12 at octoDNS 1.22.0), next to the octodns-org `.ci-config.json` matrix at
      the latest octoDNS. The coverage gate stays off until the legacy code is removed (issue D).

### Acceptance

`script/lint` and `script/format --check` pass in CI on every leg; the test suite is unchanged
and green; `python -m build` and `twine check` pass.

Refs #25
Suggested labels: enhancement
````

### 3.2 Issue B — API client

**Title:** `[0.1.0] API client: client module, per-IP rate limiting, safe retries`

````markdown
Part of the 0.1.0 work tracked in #25 (roadmap sections 3.1, 5.1, track B).

### Current state (2c45212)

0.0.18 fixed the transport: form-encoded POST, timeouts, credential redaction, no retry of
ambiguous writes. Open:

- The limiter is per client instance at 19 req/s
  (https://github.com/ClouDNS/octodns_cloudns/blob/2c45212/octodns_cloudns/__init__.py#L53-L56).
  ClouDNS limits per IP (20/s, 600/min, 36,000/h, wiki article 41), so two provider entries in one
  process, or one long apply, can exceed it.
- The HTTP call runs while holding the limiter lock (L111-L127), which serialises all threads for
  the full request time.
- A `Request blocked.` reply, which means the request was not processed, is fatal mid-apply.
- Every API failure is one exception class; callers match on message strings.
- The client lives in `__init__.py` together with the provider.

### Plan (two PRs)

- [ ] PR-B1 move `ClouDNSClient` and its exceptions to `octodns_cloudns/client.py` unchanged in
      behaviour, re-exported from `octodns_cloudns`, so existing imports keep working.
- [ ] PR-B2 extend it:
  - an ordered table from `statusDescription` to typed exceptions (missing domain, zone exists,
    auth, rate limited, invalid TTL, invalid record id, plan limit); existing classes stay;
  - **one process-wide limiter**, because the budget belongs to the egress IP, not to a credential:
    default 10 req/s (fits all three tiers), configurable `calls_per_second` for operators running
    several processes behind one address (documented, since that cannot be coordinated
    in-process); sleep and HTTP call outside the lock;
  - **retries only where the outcome is known**: `Request blocked.` on any endpoint and
    `ConnectTimeout` (nothing sent), with backoff; reads may also retry other transport errors.
    Ambiguous write outcomes (read timeout, reset after send, 5xx) are **never** retried: the client
    raises, the provider clears its cache, and the message says to re-plan, which reads back the
    real state;
  - wrappers needed by issue D: `available_ttls`, `records_count`, `record_set_status`,
    `zone_create(..., ns=[...])`.

### Acceptance

`requests_mock` tests assert: form body, credentials absent from the URL and from every log record,
`record-id` used, the exception for each known `statusDescription`, the retry matrix (endpoint x
failure kind) including no retry of ambiguous writes, and limiter spacing shared across two client
instances, with an injected clock. Behaviour that is still uncertain (the 600/min reply text, the
sub-user permission error) is probed in an isolated test zone first and committed as sanitised
fixtures.

Refs #25
Suggested labels: enhancement
````

### 3.3 Issue C — codecs

**Title:** `[0.1.0] Codecs: one table for write fields, read parsing and match keys`

````markdown
Part of the 0.1.0 work tracked in #25 (roadmap sections 3.2, 5.1, track C).

### Why

Today every record type is described up to four times: `_VALUE_FIELDS` (write and read field
names), `_data_for_*` (read parsing), `_records_are_same` (matching against API rows) and the dead
`_params_for_*` formatters. They disagree, which is what is left of the value-matching bugs on
`main` (verified with a mocked client at 2c45212):

- TLSA Update and Delete crash with `AttributeError: '_equality_tuple'`
  (https://github.com/ClouDNS/octodns_cloudns/blob/2c45212/octodns_cloudns/__init__.py#L850-L868).
- CAA and LOC rows never match (ints vs API strings): a CAA Delete makes zero API calls and reports
  success; a CAA Update adds a duplicate (L790-L825).
- Removals are skipped for NS, MX, SRV and other structured types: root NS `[a,b] -> [c,d]` adds c
  and d and leaves a and b live (L737-L748).
- Two NAPTR siblings produce four `record_delete` calls (L762-L770).
- A TXT value ending in `.` reads back without it (L310).
- Hostname values are written with a trailing dot except for MX and SRV (L224-L236).

`ds.py` (#41) already solves this for DS: one canonical primitive key per value, cast and
normalised the same way on both sides. This issue generalises that to every supported type.

### Plan (two PRs, pure additions first)

- [ ] PR-C1 `octodns_cloudns/codec.py`: a `TYPES` table built from today's `_VALUE_FIELDS` triples
      (write field, octoDNS attribute, read aliases) with `to_fields`, `from_row` and `key` per type;
      a frozen `Row` (id, type, host, ttl, status, location, note, failover, key, raw); an
      `UnparseableRow` error naming the row id, never a partial value. Not wired into the provider
      yet.
- [ ] PR-C2 DS moved onto the table; `DsCodec` stays importable; all DS tests unchanged.

Wiring the codec into populate and apply is issue D. Record types #33-#40 become one table entry
each afterwards, where octoDNS has a value class.

### Acceptance

A table-driven test per supported type: value -> fields -> fixture row -> value round-trips, `key()`
is equal on both sides, read aliases are accepted, TXT `\;` and trailing dots survive, hex and case
normalisation work, hostnames compare with or without the dot, and malformed rows raise
`UnparseableRow`. Fixtures are sanitised captures from an isolated test zone where the shape is
uncertain (TXT over 255 bytes, IDN, trailing dots on hostname writes), synthetic otherwise and
marked as such.

Refs #25
Suggested labels: enhancement, bug
````

### 3.4 Issue D — reconciliation and provider integration

**Title:** `[0.1.0] Per-record-id reconciliation and provider integration`

````markdown
Part of the 0.1.0 work tracked in #25 (roadmap sections 5.2 to 5.7, track D). Depends on C;
uses B's wrappers for TTL validation and zone creation.

### Why

0.0.18 made TTL-only updates correct. Value changes still go through the legacy path: deletes
before creates (an empty window), removals that match nothing (issue C), combined TTL and value
changes rejected, Deletes that silently do nothing. GeoDNS rows are flattened on read while
`SUPPORTS_GEO = True`, inactive rows are read as live values for every type except DS, and mixed
TTLs within a name/type are reduced to one row's TTL.

`ds.operations` (#41) is the reconcile for one type. This issue generalises it and moves every type
onto it.

### Plan

- [ ] PR-D1 `octodns_cloudns/reconcile.py`, pure (no I/O, no octoDNS imports): keep equal values with
      their ids, pair leftovers into `mod-record`, adds -> mods -> activates -> deletes; inactive
      rows outside the managed set unless reactivation is enabled; location buckets so pairing never
      crosses locations. `ds.operations` moved onto it.
- [ ] PR-D2 provider rewired:
  - populate keeps record ids;
  - **fails safe** on unparseable active rows and on rows with a non-DEFAULT GeoDNS location,
    naming the row ids, also under `lenient`;
  - inactive rows excluded from values and preserved;
  - every `_apply_*` goes through the reconcile with a preflight before the first write;
  - TTL and value changes allowed together;
  - zero-op Updates and Deletes collected and raised;
  - `SUPPORTS_GEO = False`, `SUPPORTS_MULTIVALUE_PTR = True`.
- [ ] PR-D3 TTL policy and drift repair:
  - `dns/get-available-ttl` once per run;
  - a TTL outside the account's list **fails at plan time** by default, `ttl_policy: round_up` is
    the opt-in;
  - `_extra_changes` repairs mixed TTLs and duplicate active rows (as DS does), and inactive rows
    only with `inactive_records: reactivate`.
- [ ] PR-D4 zone auto-create via `dns/register` with `ns[]` from the desired root NS; re-list when
      registering without; `has been already added.` treated as existing.
- [ ] PR-D5 legacy code (`_records_are_same`, `_params_for_*`, `_data_for_*`) and the tests pinning it
      removed; `--cov-fail-under=100` on in CI.
- [ ] PR-D6 gated live integration class, the **release check**: runs the scenario table on an
      isolated test zone, **reads back** every row after apply, asserts an **empty second plan**,
      unique names per run, cleanup in `finally`, baseline compared by id. Sanitised fixtures for
      anything newly learned.
- [ ] PR-D7 README and CHANGELOG: supported types, TTL policy and plan requirement, inactive and
      duplicate semantics, GeoDNS fail-safe, config options, class path, migration notes for
      every behaviour change.

New options, defaults per #25: `inactive_records` (`preserve` | `reactivate`), `ttl_policy`
(`reject` | `round_up`), `validate_ttl` (`api` | `static`).

Value changes on rows with notes or failover stay refused, as in 0.0.18, until a live probe shows
`mod-record` preserves them on a value change (roadmap P13).

### Acceptance

- Exact ordered-op tests for the roadmap 5.3 scenario table, plus: inactive preserved and
  reactivated, duplicates, unparseable rows, 20 -> 25 and 20 -> 15 values, root NS never empty, no
  cross-location pairing, Create against matching stray rows, and two providers sharing one plan
  without mutation.
- A Manager-level plan -> apply -> plan test ending in an empty plan.
- Coverage 100 %.
- The live run in PR-D6 passes before ClouDNS tags 0.1.0.

Refs #25
Suggested labels: enhancement, bug
````

## 4. PR series

All PRs: branch from `upstream/main`, open against `ClouDNS/octodns_cloudns:main`, tests in the
same PR, body ends with `Part of #<issue>` and `Refs #25` (the last PR of an issue says
`Closes #<issue>`). Sizes S < M < L.

| PR | branch | size | issue | depends on | contents |
|---|---|---|---|---|---|
| PR-A1 | `chore/pyproject-and-lint` | S | A | — | pyproject, `setup.py` fixes, scripts, pre-commit, pyflakes, stale configs, secret scan |
| PR-A2 | `chore/format-only` | S | A | PR-A1, D10 window | `script/format` output only, `.git-blame-ignore-revs` |
| PR-A3 | `ci/script-cibuild` | S | A | PR-A2 | CI runs `script/cibuild`; minimum-version legs kept |
| PR-B1 | `refactor/client-module` | S | B | PR-A2 | move the client to `client.py`, re-export, no behaviour change |
| PR-B2 | `feat/client-limiter-retries` | M | B | PR-B1 | typed errors, process-wide limiter, safe retries, new wrappers |
| PR-C1 | `feat/codec-module` | L | C | PR-A2 | `codec.py` with table-driven round-trip tests, not wired |
| PR-C2 | `refactor/ds-on-codec` | S | C | PR-C1 | DS on the table, DS tests unchanged |
| PR-D1 | `feat/reconcile-module` | M | D | PR-C1 | pure reconcile, DS moved onto it |
| PR-D2 | `feat/provider-on-reconcile` | L | D | PR-D1, PR-B1 | populate and apply through codec and reconcile, fail-safe, geo off |
| PR-D3 | `feat/ttl-policy-drift-repair` | M | D | PR-D2, PR-B2 | plan-time TTL validation, `_extra_changes` repair |
| PR-D4 | `feat/zone-auto-create-ns` | S | D | PR-D2, PR-B2 | `dns/register` with `ns[]`, re-list |
| PR-D5 | `chore/remove-legacy-coverage-gate` | M | D | PR-D3, PR-D4, PR-A3 | legacy removed, 100 % gate on |
| PR-D6 | `test/live-integration-release-check` | M | D | PR-D5 | gated live run, read-back, empty second plan |
| PR-D7 | `docs/readme-changelog-0.1.0` | S | D | PR-D5 | README and CHANGELOG, migration notes |

Optional, only if decision D7 is answered yes: `fix/0.0.19-fail-loud` on the maintainer's
0.0.19 branch: zero-op guard on `_apply_delete`, unmatched removals in `_apply_update` raise before
writes, TLSA crash turned into a clear `ProviderException`, with regression tests.

## 5. Order of work

1. **Now:** push ROADMAP v2 and this file to the fork; file issues A to D (section 3); fill their
   numbers into the #25 comment (section 2) and post it.
2. **PR-A1** right away; it is independent and small.
3. **PR-A2** in the window the maintainer names (D10). Everything after it branches from the formatted
   tree, so it goes before the large PRs to avoid rebasing them across a reformat.
4. **PR-B1, PR-C1 and PR-A3** in parallel after PR-A2; then **PR-B2** and **PR-C2**.
5. **PR-D1** once PR-C1 is merged; **PR-D2** once PR-D1 and PR-B1 are; **PR-D3** and **PR-D4** in either order; then
   **PR-D5**, then **PR-D6** and **PR-D7**.
6. F1 probes (P3, P4, P6, P8 to P14) run alongside, each before the PR that needs it (roadmap 7.1).

Critical path: PR-A2 -> PR-C1 -> PR-D1 -> PR-D2 -> PR-D3 -> PR-D5 -> PR-D6. Track B runs in parallel and joins at PR-D3.

## 6. Coverage check

### 6.1 Maintainer requirements from #25

| requirement | where |
|---|---|
| #25 umbrella, few focused issues, reuse #23 and #14 | sections 1 and 3; #23 and #14 already closed upstream |
| small, independently tested PRs; formatting separate | section 4; PR-A2 |
| fail safely on unsupported GeoDNS | PR-D2 (populate refuses non-DEFAULT rows, `SUPPORTS_GEO = False`); D9 |
| fail safely on unparseable managed rows | PR-C1 (`UnparseableRow`), PR-D2 |
| preserve inactive records by default, opt-in reactivation | PR-D1, PR-D2, PR-D3 (`inactive_records`); D8 |
| reject unsupported TTLs at plan time by default, rounding opt-in | PR-D3 (`ttl_policy`) |
| rate limiting accounts for shared egress IPs | PR-B2 |
| no blind retries after ambiguous writes | PR-B2 |
| isolated test zone, sanitised fixtures | B, C and D acceptance; F1 |
| read-back and an empty second plan as release checks | PR-D2 (Manager test), PR-D6 (live) |
| octoDNS >= 1.22, Python >= 3.10 for 0.1.0 only | done in #41; PR-A3 keeps the minimum-version legs |
| document behaviour changes | PR-D7 |
| release ownership with ClouDNS | no fork tags; releases not in any PR |

### 6.2 v1 findings

| findings | where |
|---|---|
| W1, W2, W5, W7, W13, R1, R3, R4, R5 | fixed upstream (#28); regression tests exist |
| W3, W4, W6, W8, W11, W14 | PR-C1 (codec) and PR-D2 (wired in); PR-D5 removes the legacy code |
| W9, R2 | PR-D2 (GeoDNS fail-safe) |
| W10 | PR-D1 (adds before deletes, paired mods) |
| W12, R10 | PR-D2 |
| R6, R9 | PR-B2 |
| R7 | PR-D1, PR-D2 |
| R8 | PR-D2, PR-D3 |
| S1 (scanner) | PR-A1 |
| S2 | PR-D5 (100 % gate) |
| S3 | PR-A3 |
| S4 | PR-A1 |
| S5 | PR-A1 |

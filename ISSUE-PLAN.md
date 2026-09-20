# Issue and PR plan for [ROADMAP.md](ROADMAP.md)

Status: v1, 2026-09-20. Proposal only — nothing in this file has been filed yet.

Covers every item of the roadmap (phases 0 to 5), the 41 verified findings of
[BUG-REPORT.md](BUG-REPORT.md) / roadmap section 2, the 12 live probes (P1 to P12) and the six
maintainer decisions (D1 to D6). One overarching tracking issue (E1), 14 bug issues (B\*), 27 task
issues (T\*), 4 fork-only issues (F\*), and 24 PRs that close them.

Baseline: fork `istr/octodns_cloudns` `main` == `upstream/main` == `c29be64` (0 commits either way),
so every PR branches off `upstream/main` and is opened against `ClouDNS/octodns_cloudns`.

## 1. Where things are filed, and why

| | upstream `ClouDNS/octodns_cloudns` | fork `istr/octodns_cloudns` |
|---|---|---|
| what goes there | everything about the published package: bugs, code tasks, tests, CI, packaging, docs, releases, the tracking issue | only what upstream cannot act on: our credentials, our zones, our git tags, our contingency planning |
| why | other `octodns-cloudns` users hit these bugs and search upstream; upstream owns PyPI and the API; PRs need an issue number to reference and `Closes #N` only works within the target repo | credentials and throwaway-zone work is account-specific; fork tags exist only because the fork has no PyPI rights |
| count below | E1 + B1-B14 + T1-T27 | F1-F4 |

Recommendation: **file 42 upstream (T26 goes to `octodns/octodns` instead), 4 in the fork.** The
fork is not a place to hide bug
reports — upstream's closed issues #2, #3, #4, #11, #16 and #18 are all symptoms of the same
value-string matching design, so the new bug issues belong next to them.

### 1.1 Prerequisites before filing

1. **Enable Issues on the fork.** `istr/octodns_cloudns` currently has `has_issues: false` (the
   GitHub default for forks). Settings → General → Features → Issues. Without this, F1-F4 have
   nowhere to live.
2. **Commit `ROADMAP.md`, `BUG-REPORT.md` and this file to the fork's `main`** (they are untracked
   today). Every upstream issue below links to them as the evidence trail; a link to an uncommitted
   file 404s. Do not PR them upstream — they are working documents naming upstream's own mistakes,
   and E1 carries the content upstream needs.
3. **Accept that you cannot label or milestone upstream issues.** Applying labels needs triage
   permission; upstream's labels are the nine GitHub defaults (`bug`, `documentation`, `duplicate`,
   `enhancement`, `good first issue`, `help wanted`, `invalid`, `question`, `wontfix`) and there are
   no milestones. Therefore: encode the phase in the **title prefix** (`[0.0.18]`, `[0.1.0]`,
   `[chore]`, `[probe]`) and put a `Suggested labels:` line at the bottom of each body for a
   maintainer to apply. Milestones `0.0.18` and `0.1.0` are worth asking for in E1.
4. **Establish standing in E1, then still file in waves.** 42 issues appearing overnight on a repo
   with 24 lifetime issues reads as a hostile takeover from a stranger — but you are not a stranger
   to octoDNS, and E1 now says so up front (octoDNS org and `octodns/review` member, 13 merged PRs
   across seven octodns-org repos, octodns-gcore root-NS support), together with an explicit "this is
   help with your repo, not a fork announcement" and an offer to collapse the whole thing into fewer
   PRs. That reframes the volume as review capacity rather than pressure, which is exactly what it
   is. It does not make the volume itself disappear: still file E1 first, wait for a maintainer
   reply, then file the rest in waves (section 5). If upstream prefers fewer issues, E1's checklist
   alone is enough and the B/T issues collapse into PR descriptions.

   On verifiability: the octoDNS org membership is public as of 2026-09-20 and was confirmed
   (`GET /orgs/octodns/public_members/istr` → 204; the org currently lists exactly one public member,
   `istr`), so the affiliation in E1 can be checked from the profile badge and needs no further
   evidence. Team membership (`octodns/review`) is never publicly visible on GitHub, so that half of
   the sentence stays on trust — which is why E1 also names the public record behind it: 32 public
   issues and PRs across `octodns`, `octodns-constellix`, `octodns-digitalocean`, `octodns-gcore`,
   `octodns-ns1`, `octodns-ovh` and `octodns-rackspace`, 13 of them merged, with octodns-gcore#48/#49
   linked inline. Keep that link in the body even though the badge is now live: it shows a provider
   feature requested, implemented and merged, which is the specific thing being offered here.

### 1.2 Already filed (do not duplicate)

| # | title | maps to | still needed |
|---|---|---|---|
| upstream #23 | Remove hardcoded credentials from integration tests | phase 0.1 + 0.3, S1 | PR-0.1 does the repo-side half; rotation stays with ClouDNS / the PR #20 author |
| upstream #24 | Make license information consistent | D1, part of S4 | nothing from us; PR-1.1 deliberately leaves the `license=` line alone and records the discrepancy in `CHANGELOG.md` |
| upstream #14 | Support for DNSSEC records | phase 5 | T22 comments on it rather than opening a new issue |

## 2. E1 — the overarching tracking issue

**Repo:** upstream · **Kind:** tracking/epic · **Suggested labels:** `enhancement`, `help wanted` ·
**Blocked by:** — · **Closed by:** PR-3.10 (or manually, once every box is ticked)

**Title:** `Tracking: fix the silent TTL no-op and move to update-based reconciliation (roadmap to 0.1.0)`

**Body:** the framing below, then the **full text of `ROADMAP.md`**, then the checklist. It fits:
`ROADMAP.md` is 46,768 characters against GitHub's 65,536-character body limit, leaving ~15 KB for
the framing and checklist.

Three edits are required when pasting the roadmap into the issue body, because relative links do
not resolve in an issue:

- `octodns_cloudns/__init__.py#L577-L605` and friends →
  `https://github.com/ClouDNS/octodns_cloudns/blob/c29be64/octodns_cloudns/__init__.py#L577-L605`
  (pin the commit, not `main`, or the line anchors drift with every merge).
- `BUG-REPORT.md` → its committed fork URL (prerequisite 1.1.2).
- `../octodns-cloudflare/octodns_cloudflare/__init__.py#L806-L934` →
  `https://github.com/octodns/octodns-cloudflare/blob/main/octodns_cloudflare/__init__.py#L806-L934`.

Keep `ROADMAP.md` in the fork as the source of truth and re-sync the issue body when it changes;
say so in the framing so readers know which copy wins.

````markdown
`octodns-cloudns` 0.0.17 applies a TTL-only change by making zero API calls, while
`octodns-sync --doit` reports success and exits 0. This has already caused undetected production
drift in one of our zones (`deflect.me`: config said 600/3600/86400, ClouDNS served a flat 300).

The TTL bug is one symptom of a structural issue: the provider matches API records to octoDNS values
by re-deriving strings in three places that disagree with each other, and it discards the record ids
that `dns/records` returns. That same root cause produced the already-closed #2, #3, #4, #11, #16 and
#18. The fix is to keep the ids and reconcile per record id with `dns/mod-record`, the way
octodns-cloudflare does for a backend with the same shape.

This issue is the tracking issue for that work. The full roadmap follows: what is broken and how it
was verified (section 2), the ClouDNS API and octoDNS contracts the rewrite codes against (3, 4), the
target design (5), the phased plan (6), the live probes and the decisions we need from you (7).

Where this comes from: I run a handful of zones on ClouDNS through octoDNS, and I work on octoDNS
providers generally — I'm a member of the octoDNS organisation and of its `octodns/review` team, and
have contributed to octoDNS core and to the constellix, gcore, ns1, ovh, digitalocean and rackspace
providers (13 merged PRs; most recently root-NS support in octodns-gcore, octodns/octodns-gcore#48
and #49). That is also why the tooling PRs below follow the octodns-org conventions —
`script/cibuild`, the shared `.ci-config.json` Python matrix, black/isort config in `pyproject.toml`,
the `tests/config/unit.tests.yaml` fixture layout — rather than inventing new ones, and why the
design in section 5 is modelled on octodns-cloudflare, the one octodns-org provider whose backend has
the same per-record-id shape as ClouDNS.

This is offered as help with your repository, not as a fork announcement. The work is written and
tested in <fork URL> and submitted here as small self-contained PRs in the order below; nothing lands
without tests, and every PR is yours to take, amend or decline. If the number of issues and PRs is
more than you want in your tracker, say so and I will collapse them into fewer, larger PRs, or hold
the rewrite and send only the TTL fix. I am also happy to take review load on this repo if that would
help.

Decisions marked D1 to D6 in section 7.2 are yours; probes P1 to P12 need one throwaway zone on a
paid plan and I am happy to run them if you would rather not.

Three asks up front:
- milestones `0.0.18` and `0.1.0`, so the two releases can be tracked;
- a maintainer opinion on D5 (raise the floor to octoDNS 1.22 / Python 3.10 in 0.1.0) and D6 (ship
  0.0.18 as a small TTL-only fix first, or go straight to 0.1.0) before phase 2 starts;
- a say on how you want the rest filed: one issue per finding as listed in the checklist, or just
  this issue plus the PRs.

The canonical copy of this roadmap lives at <fork ROADMAP.md URL> and is updated as probes land;
this body is a snapshot.

---

<full ROADMAP.md text, links rewritten as described>

---

### Checklist

**Phase 0 — stop the bleeding**
- [ ] #23 rotate sub-user 85055 (ClouDNS / PR #20 author)
- [ ] #23 remove the credential and `argl.net` from the test module (PR-0.1)
- [ ] T1 `.gitignore` + secret scanning in CI and pre-commit (PR-0.2)
- [ ] #24 license decision (D1)

**Phase 1 — safety net, tooling, probes**
- [ ] T2 packaging metadata + `pyproject.toml` + `CHANGELOG.md` (PR-1.1)
- [ ] T3 floor bump to octoDNS 1.22 / Python 3.10 (D5) (PR-1.2)
- [ ] T4 formatting pass and lint fixes (PR-1.3, PR-1.4)
- [ ] T5 CI on the octodns-org matrix via `script/cibuild` (PR-1.5)
- [ ] T6 `script/probe-api` + captured fixtures (PR-1.6)
- [ ] T7 API contract questions P1, P4, P5, P8-P12
- [ ] T8 test suite restructure (PR-1.7)
- [ ] T9 characterisation tests for W1-W8, R1, R5 (PR-1.8)

**Phase 2 — 0.0.18, the bug fix**
- [ ] T10 client: POST body, timeout, `record_mod` (PR-2.1)
- [ ] B8 credentials out of the URL and the logs (PR-2.1)
- [ ] B1 TTL-only updates use `dns/mod-record` (PR-2.2)
- [ ] B2 CNAME/ALIAS/DNAME updates stop crashing (PR-2.2)
- [ ] B9 populate stops swallowing API errors, `[]` handled (PR-2.3)
- [ ] T11 release 0.0.18 (PR-2.4)

**Phase 3 — 0.1.0, the rewrite**
- [ ] T12 `client.py` (PR-3.1), B12 transport and rate limiting
- [ ] T13 `codec.py` (PR-3.2), B4 TLSA, B5 CAA/LOC, B6 TXT, B14 SSHFP/LOC spellings
- [ ] T14 `reconcile.py` (PR-3.3), B3 value removals, B7 update-path damage
- [ ] T15 `provider.py` (PR-3.4), B10 geo, B11 inactive rows and TTL drift, B13 cleanup
- [ ] T16 TTL policy and drift repair (D3, D4) (PR-3.5)
- [ ] T17 zone auto-create with `ns[]` (PR-3.6)
- [ ] T18 legacy code removed, 100 % coverage gate on (PR-3.7)
- [ ] T19 gated live integration tests (PR-3.8)
- [ ] T20 README rewrite (PR-3.9)
- [ ] T21 release 0.1.0 (PR-3.10)

**Phase 4 — docs and infrastructure on your side**
- [ ] T27 wiki article 509 class path, articles 59/60 `id` vs `record-id`, PyPI home page, git tags

**Phase 5 — follow-ups**
- [ ] T22 DS, HTTPS/SVCB, OPENPGPKEY (#14)
- [ ] T23 GeoDNS via `dynamic` records (after P7)
- [ ] T24 pagination guard (only if P4 shows truncation)
- [ ] T25 sliding-window rate limiter
- [ ] T26 "reported change count vs. real operations" in octoDNS core

Suggested labels: enhancement, help wanted
````

## 3. Issue catalogue

### 3.1 Bug issues (upstream, user-visible symptoms)

Each body: **Symptom** (what a user sees) · **Root cause** (with a pinned blob link) · **Evidence**
(how it was reproduced) · **Expected** · **Acceptance** (the test that must exist) ·
`Suggested labels:`. Keep them short — the depth lives in E1.

| id | title | findings | severity | closed by | notes |
|---|---|---|---|---|---|
| B1 | `TTL-only updates silently no-op: 0 API calls, sync reports success` | W1 | critical | PR-2.2 | the headline bug; full body in 3.2 |
| B2 | `Updating a CNAME, ALIAS or DNAME crashes with AttributeError: no attribute 'values'` | W2 | critical | PR-2.2 | regression from #21, related to closed #4/#5; leaves plans half-applied |
| B3 | `Removing a value from NS, MX, SRV, SSHFP, NAPTR, LOC or a quoted TXT does nothing` | W4 | critical | PR-3.3 | same family as closed #11/#16/#18; non-convergent, re-plans forever |
| B4 | `TLSA updates and deletes crash in _records_are_same` | W3 | critical | PR-3.2 | no TLSA branch; the generic branch raises |
| B5 | `CAA and LOC records are never matched: updates create duplicates, deletes no-op` | W6 | high | PR-3.2 | int/float vs API strings; also notes that #22's `except TypeError` is dead on octoDNS 1.22+ |
| B6 | `A TXT value ending in a dot never round-trips: perpetual update plus one duplicate per run` | W8 | medium | PR-3.2 | `rstrip('.')`; related to closed #3 |
| B7 | `Update path mutates plan.desired, opens an outage window, and double-deletes NAPTR rows` | W5, W10, W11 | high | PR-2.2 (W5), PR-3.3 (W10, W11) | three distinct failures of one code path; splittable if a maintainer prefers |
| B8 | `Credentials travel in the URL, parameters are unencoded, and the full URL is logged at DEBUG` | W7, R4 | high | PR-2.1 | security; `&`/`#`/`%XX` in TXT, CAA, NAPTR values mangle the request and ClouDNS answers Success |
| B9 | `populate() swallows every API error and reports the zone as non-existent; an empty zone crashes it` | R1, R5 | critical | PR-2.3 | worst case: a source-mode misread makes every other target plan a full delete |
| B10 | `GeoDNS records are flattened on read and written with the wrong values` | R2, W9 | high | PR-3.4 | proposes `SUPPORTS_GEO = False` for 0.1.0 (D2) and asks for P7 |
| B11 | `Inactive rows read as live values; per-record TTLs collapsed to one row's TTL` | R7, R8 | medium | PR-3.4, PR-3.5 | makes drift left by a partial apply invisible |
| B12 | `No timeout, no retries, mutating calls over GET, and only the 20 req/s tier is throttled` | R6, R9 | medium | PR-3.1 | a `Request blocked.` reply becomes a fatal exception mid-apply |
| B13 | `Cleanup: redundant get-zone-info, stale record cache, dead value formatters, wrong log count` | W12, W13, W14, R10 | low | PR-1.4, PR-3.4 | one `good first issue`-shaped housekeeping issue |
| B14 | `SSHFP and LOC use different parameter spellings on read and write (fp_type vs fptype, lat_deg vs lat-deg)` | R3 | high | PR-3.2 | the wiki documents the underscore form, both ClouDNS SDKs send the dashed form; ask which is authoritative (P1) and offer the send-both fallback |

### 3.2 B1 in full (ready to paste)

**Title:** `TTL-only updates silently no-op: 0 API calls, sync reports success`

````markdown
### Symptom

With 0.0.17, a change that only alters a record's TTL makes **zero** HTTP requests to the ClouDNS
API, while `octodns-sync --doit` prints `N total changes` and exits 0. There is no error, no warning
and no other signal. The next run plans the same change again.

Affected types: A, AAAA, TXT, SPF, NS, PTR, MX, SRV, SSHFP, NAPTR, CAA and LOC.

### Root cause

`ClouDNSProvider._apply_update` diffs on value only, never on TTL
(https://github.com/ClouDNS/octodns_cloudns/blob/c29be64/octodns_cloudns/__init__.py#L577-L605):

```python
to_delete = set(existing.values).difference(change.new.values)
to_create = set(change.new.values).difference(existing.values)
```

When only the TTL changed, both sets are empty, `_apply_create` iterates zero times and nothing is
sent. Structurally, `ClouDNSClient` has no `dns/mod-record` wrapper at all — the provider was built
around delete-then-recreate, and a TTL is not a value, so it has no path to express this change.

### Evidence

- Reproduced on this checkout with octoDNS 1.22.0 and `provider._client = Mock()`: a TTL-only Update
  on A, MX and a root NS record yields an empty call list.
- Observed live with `--debug` on `ingo-struck.de.`: exactly two requests, both GET (`dns/records`,
  `dns/get-zone-info`), no write of any kind, and `1 total changes` printed.
- Confirmed production drift: `deflect.me` was configured with TTLs 600/3600/86400 and ClouDNS
  served a flat 300 on its own authoritative servers (`dig`), while a second provider's untouched
  copy of the same zone matched the committed config exactly.

### Expected

A TTL-only Update issues one `dns/mod-record` per matching API record, resending the full record
with the new TTL, and no `dns/add-record` or `dns/delete-record` at all. Delete-then-recreate is not
an acceptable fix here: `dns/mod-record` exists, and delete+add opens a window in which the name has
no records.

An Update that produces zero API operations must fail the sync rather than report success.

### Acceptance

Regression tests asserting the exact client call list for a TTL-only Update on A (multi-value), MX,
a root NS set and CNAME: one `mod-record` per row, zero adds, zero deletes; plus one test that a
zero-operation Update raises.

Context, design and the rest of the findings: #<E1>
Suggested labels: bug
````

### 3.3 Task issues (upstream)

| id | title | phase | kind | blocked by | closed by |
|---|---|---|---|---|---|
| T1 | `[chore] Add .gitignore and secret scanning to CI and the pre-commit hook` | 0 | chore | — | PR-0.2 |
| T2 | `[chore] Fix packaging metadata and add pyproject.toml, CHANGELOG.md` | 1 | chore | — | PR-1.1 |
| T3 | `[chore] Raise the floor to octoDNS >= 1.22 and Python >= 3.10 (D5)` | 1 | decision + chore | T2, maintainer answer on D5 | PR-1.2 |
| T4 | `[chore] One formatting-only commit, then fix the pyflakes findings and dead code` | 1 | chore | T2 | PR-1.3, PR-1.4 |
| T5 | `[chore] Run script/cibuild on the octodns-org Python matrix in CI` | 1 | chore | T2, T4 | PR-1.5 |
| T6 | `[probe] Add script/probe-api and commit the captured API fixtures` | 1 | test | T7 answers or an own throwaway zone | PR-1.6 |
| T7 | `[probe] API contract questions: parameter spellings, pagination, inactive rows, rate-limit text` | 1 | question | — | no PR (answers land in T6's fixtures) |
| T8 | `[test] Restructure the test suite and add tests/config/unit.tests.yaml` | 1 | test | PR-0.1 | PR-1.7 |
| T9 | `[test] Characterisation tests for W1-W8, R1 and R5` | 1 | test | T8, T6 | PR-1.8 |
| T10 | `[0.0.18] Client: POST form body, timeout, and a record_mod wrapper for dns/mod-record` | 2 | feature | T9 | PR-2.1 |
| T11 | `[0.0.18] Release 0.0.18` | 2 | release | B1, B2, B8, B9 | PR-2.4 |
| T12 | `[0.1.0] Extract client.py: typed exceptions, shared limiter, idempotency-aware retries` | 3 | feature | T10 | PR-3.1 |
| T13 | `[0.1.0] Add codec.py: one table for write params, read parsing and match keys` | 3 | feature | T6 (P1), T12 | PR-3.2 |
| T14 | `[0.1.0] Add reconcile.py: a pure per-record-id reconcile` | 3 | feature | T13 | PR-3.3 |
| T15 | `[0.1.0] Rewire the provider onto codec and reconcile` | 3 | feature | T14 | PR-3.4 |
| T16 | `[0.1.0] TTL policy at plan time and drift repair via _extra_changes (D3, D4)` | 3 | feature | T15 | PR-3.5 |
| T17 | `[0.1.0] Zone auto-create: register with ns[] and re-list before applying` | 3 | feature | T15, T6 (P9, P10) | PR-3.6 |
| T18 | `[0.1.0] Delete the legacy code paths and turn on the 100 % coverage gate` | 3 | chore | T15, T16, T17 | PR-3.7 |
| T19 | `[0.1.0] Gated live integration tests against a throwaway zone` | 3 | test | T18 | PR-3.8 |
| T20 | `[0.1.0] Rewrite the README: supported types, TTL policy, ownership semantics, config keys` | 3 | docs | T18 | PR-3.9 |
| T21 | `[0.1.0] Release 0.1.0` | 3 | release | T18, T19, T20 | PR-3.10 |
| T22 | `Support DS, HTTPS/SVCB and OPENPGPKEY` | 5 | enhancement | T21 | — (comment on #14 instead of filing, then file when scheduled) |
| T23 | `GeoDNS via dynamic records with geo-only rules` | 5 | enhancement | T21, P7 | — |
| T24 | `Pagination guard: cross-check dns/get-records-count` | 5 | enhancement | P4 | — (only if P4 shows truncation) |
| T25 | `Sliding-window rate limiter for large applies` | 5 | enhancement | T21 | — |
| T26 | `octoDNS core: "N total changes" should reflect operations actually performed` | 5 | question | — | — (filed in `octodns/octodns`, not here) |
| T27 | `[docs] Wiki article 509 class path, articles 59/60 record-id, PyPI home page, release tags` | 4 | docs | — | no PR (ClouDNS-side) |

Body sketches for the ones that are not self-evident:

- **T3** — state both sides: octoDNS 1.22 is needed for `TxtValue.normalize_raw_text`, hashable
  CAA/TLSA values and current `Record.new` semantics, and octoDNS 1.22 itself requires Python 3.10+,
  so the current 3.8 CI leg silently tests octoDNS 1.10. Offer the fallback from roadmap phase 4:
  the reconcile keys on strings, so only the TXT helpers need 1.22 and a `replace('\\;', ';')`
  fallback keeps an older floor installable if upstream objects.
- **T7** — one issue, one table, twelve rows: P1 (`fptype`/`lat-deg` vs `fp_type`/`lat_deg`, write
  and read), P2 (does `mod-record` preserve status, notes, failover, location?), P3 (`data.id`
  number or string), P4 (pagination beyond 100 rows), P5 (are `status=0` rows listed, does
  `mod-record` reset status?), P6 (TXT over 255 bytes in the listing), P8 (duplicate add-record),
  P9 (default NS rows deletable, `mod-record` on a monitored row), P10 (`ns[]` suppresses defaults),
  P11 (sub-user permission error text, unknown extra parameters), P12 (600/min reply text, punycode
  vs Unicode for IDN). Say explicitly that each answer becomes a committed fixture and that we will
  run the probes ourselves if they prefer — the questions are cheaper for them to answer than for us
  to measure.
- **T11 / T21** — checklist: version bump, `CHANGELOG.md` entry naming every behaviour change,
  `script/release`, PyPI upload (ClouDNS only), a GitHub tag and release. For 0.1.0 the CHANGELOG
  must name: `mod-record` instead of delete+add, errors no longer swallowed by `populate`, TTL
  validation at plan time, drift repair as visible plan lines, geo dropped, floor bump.
- **T16** — carries D3 (`inactive_records` default `reactivate` vs `ignore`) and D4 (`ttl_policy`
  default: fail at plan time vs round up) as a question to maintainers, with the roadmap's
  recommendation and the reasoning.
- **T19** — note that this needs `CLOUDNS_INTEGRATION_TEST`, `CLOUDNS_SUB_AUTH_ID` /
  `CLOUDNS_AUTH_ID`, `CLOUDNS_AUTH_PASSWORD` and `CLOUDNS_TEST_DOMAIN` as repository secrets, which
  only a maintainer can add; the tests skip without them, so CI stays green either way.

### 3.4 Fork issues

Needs prerequisite 1.1.1 (enable Issues).

| id | title | why not upstream |
|---|---|---|
| F1 | `Run the API probes P1-P12 against a throwaway zone and commit the fixtures` | needs our paid plan, our API user and a throwaway zone; the account details and raw responses are ours. Feeds T6/T7 upstream. |
| F2 | `Tag v0.0.18 and v0.1.0 in the fork for pip install from git` | the fork has no PyPI rights for `octodns-cloudns`; these tags exist so our own deployments can consume the fix before upstream releases. `script/release` signs with GPG and pushes to `origin`. |
| F3 | `Sweep all ClouDNS-managed zones for TTL drift after the first 0.1.0 sync` | operational, about our zones. Belongs in the octodns config repo; mirror here only if that repo has no issue tracker. |
| F4 | `Upstream PR queue: track review status, cherry-pick plan if upstream stalls` | a contingency about upstream's responsiveness — filing that upstream would be tactless and useless. Holds the fallback: keep PRs small and self-contained so they can be cherry-picked, and ship fork tags meanwhile. |

## 4. PR series

All PRs: branch from `upstream/main` in the fork, open against `ClouDNS/octodns_cloudns:main`, body
ends with `Closes #<issue>` and `Part of #<E1>`, tests included in the same PR (upstream merges
quickly with light review, so nothing may depend on a follow-up). Sizes S < M < L.

| PR | branch | size | closes | contents |
|---|---|---|---|---|
| PR-0.1 | `fix/remove-committed-credentials` | S | #23 (repo side) | drop the password, the `85055` default and `argl.net`; require `CLOUDNS_INTEGRATION_TEST` + env credentials, skip otherwise |
| PR-0.2 | `chore/gitignore-and-secret-scan` | S | T1 | `.gitignore` from octodns-cloudflare, gitleaks step in CI, same check in `.git_hooks_pre-commit` |
| PR-1.1 | `chore/packaging-and-pyproject` | M | T2, part of B13 | `url` fix, `pyproject.toml` with the octodns-org black/isort/pytest config, `CHANGELOG.md` (recording the D1 license discrepancy without resolving it), regenerated `requirements*.txt`, `.git_hooks_pre-commit` shipped, `script/format` honours `VENV_NAME`, `script/cibuild-setup-py` uses `pip install .`, `tests_require` dropped |
| PR-1.2 | `chore/floor-octodns-1.22` | S | T3 | `install_requires=octodns>=1.22.0`, `python_requires>=3.10`; **hold until D5 is answered** |
| PR-1.3 | `chore/format-only` | S | T4 (part) | `script/format` output only, no logic changes, hash added to `.git-blame-ignore-revs` |
| PR-1.4 | `chore/lint-and-dead-code` | S | T4 (part), part of B13 | four pyflakes findings, unused imports, the two never-raised exception classes, the `found -N records` log (R10) |
| PR-1.5 | `ci/octodns-org-cibuild` | M | T5, part of S3 | replace `ci.yml` with the octodns-cloudflare pattern reading `.ci-config.json`, run `script/cibuild` and `script/cibuild-setup-py`; coverage gate stays off until PR-3.7 |
| PR-1.6 | `test/probe-api-and-fixtures` | M | T6 | `script/probe-api` (registers `octodns-it-<uuid>.test`, cleans up in `finally`), raw responses as `tests/fixtures/cloudns-*.json`, probe results appended to `CHANGELOG.md` |
| PR-1.7 | `test/restructure-suite` | M | T8 | rename to `tests/test_octodns_provider_cloudns.py`, add `tests/config/unit.tests.yaml` with one record per supported type, delete the three tautological tests, the octodns-core-only test, `tests/config.yaml` and `tests/config/recordsfortest.bg.yaml` |
| PR-1.8 | `test/characterisation` | M | T9 | one test per W1-W8, R1, R5 driving `_apply` with a mocked client and asserting the exact call list; the ones asserting phase 2/3 behaviour marked `xfail` |
| PR-2.1 | `fix/client-post-timeout-mod-record` | M | T10, B8 | `request()` switched to a POST form body with a timeout; `record_mod()` wrapping `dns/mod-record` with `record-id` and the full row; drop `logging.basicConfig`, the Bearer header and URL logging; redacted DEBUG logging |
| PR-2.2 | `fix/ttl-only-updates-use-mod-record` | M | B1, B2, W5 of B7 | in `_apply_update`: single-value normalisation, one `record_mod` per matched row whose TTL differs, never assign to `change.new`, raise `ProviderException` on a zero-operation Update; flips the matching `xfail`s from PR-1.8 |
| PR-2.3 | `fix/populate-error-propagation` | S | B9 | `zone_records` coerces `[]` to `{}` and propagates everything but `Missing domain-name`; `exists` is False only for that one case |
| PR-2.4 | `release/0.0.18` | S | T11 | version bump, CHANGELOG entry naming the known remaining limits (B3, B4, B5, B6 are phase 3) |
| PR-3.1 | `feat/client-module` | L | T12, B12 | `octodns_cloudns/client.py` per roadmap 5.1: typed exception table, shared limiter keyed by account with an injected clock, retry on `Request blocked.` and on transport errors for idempotent endpoints only; `requests_mock` tests asserting the POST body and that no log record contains the password |
| PR-3.2 | `feat/codec-module` | L | T13, B4, B5, B6, B14 | `codec.py`: `TYPES` table with `to_fields`/`from_row`/`key`, `FIELD_ALIASES`, frozen `Row`; table-driven round-trip test per type against the fixtures |
| PR-3.3 | `feat/reconcile-module` | M | T14, B3, W10+W11 of B7 | `reconcile.py`: pure function, `Add`/`Mod`/`Activate`/`Delete`, location-bucketed pairing, ordering adds → mods → activates → deletes; exact ordered-op tests for the whole scenario table plus duplicates, inactive, unparsable, 20→25, 20→15, root NS never empty |
| PR-3.4 | `feat/provider-on-reconcile` | L | T15, B1 (fully), B10, B11, rest of B13 | `provider.py`: populate on the side table keeping ids, `_apply_*` through `reconcile`, write-through cache cleared in `finally`, collected zero-op failure, `SUPPORTS_GEO=False`, `SUPPORTS_MULTIVALUE_PTR=True`; two-instance mutation-guard test; Manager-level end-to-end test |
| PR-3.5 | `feat/ttl-policy-and-drift-repair` | M | T16, B11 | `_process_desired_zone` TTL validation against `dns/get-available-ttl` with `supports_warn_or_except`, `_extra_changes` repair Updates for mixed TTLs, inactive, duplicate and unparsable rows, with INFO lines naming the row ids |
| PR-3.6 | `feat/zone-auto-create` | M | T17 | `dns/register` with `ns[]` from the desired root NS, re-list when registering without, `has been already added.` treated as exists, GeoDNS plan error naming the `zone_type` option |
| PR-3.7 | `chore/remove-legacy-and-coverage-gate` | M | T18 | delete the old module code and the tests pinning delete+recreate, turn on `--cov-fail-under=100` |
| PR-3.8 | `test/live-integration-gated` | M | T19 | gated integration class running the scenario table against a throwaway zone, unique label per run, cleanup in `finally` |
| PR-3.9 | `docs/readme-rewrite` | M | T20 | supported types as verified, TTL policy and the Premium/DDoS/GeoDNS plan requirement, inactive/duplicate ownership semantics, geo dropped, config keys, the correct class path |
| PR-3.10 | `release/0.1.0` | S | T21, E1 | version bump, full CHANGELOG entry, upgrade notes |

## 5. Order of work

Waves, each leaving the suite green and releasable:

1. **Now, independent:** PR-0.1, PR-0.2. File E1 first, then B1 and B8 (the two a user can be
   bitten by today), then T7 — the probe answers have the longest lead time.
2. **After a maintainer reply:** PR-1.1, PR-1.3 → PR-1.4 → PR-1.5. PR-1.2 only once D5 is answered.
   PR-1.6 runs in parallel as soon as a throwaway zone exists (F1).
3. **Safety net:** PR-1.7 → PR-1.8. Nothing in phase 2 lands before the characterisation tests do.
4. **0.0.18:** PR-2.1 → PR-2.2 → PR-2.3 → PR-2.4. Decision D6 gates this wave; if upstream wants to
   skip 0.0.18, PR-2.1 still lands (phase 3 builds on it) and PR-2.2/2.3 fold into PR-3.4.
5. **0.1.0:** PR-3.1 → PR-3.2 → PR-3.3 → PR-3.4, then PR-3.5, PR-3.6 in either order, then PR-3.7 →
   PR-3.8, PR-3.9 → PR-3.10. PR-3.2 needs P1 answered, or ships both parameter spellings.
6. **After the release:** T27 on the ClouDNS side, F3 in the ops repo, then phase 5 (T22-T26).

Critical path: E1 → D5/D6 answers → PR-1.7/1.8 → PR-2.x → PR-3.1..3.4 → PR-3.7 → PR-3.10. Everything
in phase 0 and 1 except PR-1.2 can proceed without a maintainer reply.

## 6. Coverage check against the roadmap

| roadmap item | where |
|---|---|
| phase 0.1, 0.3 (rotate, notify) | upstream #23 |
| phase 0.2 (remove credential) | PR-0.1 |
| phase 0.4 (gitignore, secret scan) | T1 / PR-0.2 |
| phase 1.1 to 1.6 | T2-T9 / PR-1.1 to PR-1.8 |
| phase 2.1 to 2.5 | T10, B1, B2, B8, B9 / PR-2.1 to PR-2.4 |
| phase 3.1 to 3.7 | T12-T21 / PR-3.1 to PR-3.10 |
| phase 4.1 (tracking, PR order) | E1, section 5 |
| phase 4.2 (wiki, PyPI, tags) | T27 |
| phase 4.3 (drift sweep) | F3 |
| phase 5 | T22-T26 |
| P1-P6, P8-P12 | T7 (questions), T6 / PR-1.6 (fixtures), F1 (execution) |
| P7 | T23 |
| D1 | upstream #24, recorded by PR-1.1 |
| D2 | B10, decided in PR-3.4 |
| D3, D4 | T16 / PR-3.5 |
| D5 | T3 / PR-1.2 |
| D6 | E1 opening ask, gates wave 4 |
| W1-W14, R1-R10 | B1-B14 (all 24 findings mapped; see 3.1) |
| S1-S5 | #23 + PR-0.1 (S1), T18/PR-3.7 (S2), T5/PR-1.5 (S3), T2/PR-1.1 + #24 (S4), T8/PR-1.7 (S5) |

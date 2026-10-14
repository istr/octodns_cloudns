# Roadmap: fix the silent TTL no-op and move octodns-cloudns to update-based reconciliation

Status: v2, 2026-10-14. Re-baselined on upstream `main` at
[`2c45212`](https://github.com/ClouDNS/octodns_cloudns/commit/2c45212) (`0.1.0.dev0`), after
maintainer approval in [#25](https://github.com/ClouDNS/octodns_cloudns/issues/25). v1
(2026-09-20, against 0.0.17 at `c29be64`) is preserved as tag
[`roadmap-v1`](https://github.com/istr/octodns_cloudns/blob/roadmap-v1/ROADMAP.md). Upstream is
[ClouDNS/octodns_cloudns](https://github.com/ClouDNS/octodns_cloudns), which owns releases and the
PyPI package `octodns-cloudns`.

How to read this document: section 0 is what changed since v1; section 2 is what is broken, with
its status today; sections 3 and 4 are the two contracts the rewrite must respect (ClouDNS API,
octoDNS core); section 5 is the target design; section 6 is the remaining plan; section 7 lists
what still needs a live probe and what needs a maintainer decision. **Verified** means reproduced
with octoDNS 1.22.0 and a mocked HTTP client on the commit named, or read from the raw ClouDNS wiki
pages and ClouDNS's own SDKs. **Live** means validated against a real ClouDNS zone (by upstream in
[docs/live-validation-0.0.18.md](docs/live-validation-0.0.18.md) unless stated otherwise). **Probe**
means it still needs a live call against an isolated test zone before code may rely on it.

## 0. What changed since v1

Upstream acted on the roadmap faster than planned and did a large part of it themselves:

| upstream | content | roadmap items |
|---|---|---|
| [#26](https://github.com/ClouDNS/octodns_cloudns/pull/26), [#27](https://github.com/ClouDNS/octodns_cloudns/pull/27) | MIT in both `LICENSE` and `setup.py` | D1, S4 (part), closes #24 |
| [#28](https://github.com/ClouDNS/octodns_cloudns/pull/28) | **0.0.18**, released 2026-10-11 to PyPI with tag `v0.0.18`: TTL-only updates via `dns/mod-record` for every row, form POST with timeouts, credential redaction, error propagation in populate, `[]` handled, desired records never mutated, CNAME/ALIAS/DNAME value updates via `mod-record`, zero-op guard on Updates, credential removed from tests, live validation of all 16 static types | phase 0 (repo side), phase 2 complete; W1, W2, W5, W7, W13, R1, R3, R4, R5, S1; P1 answered; closes #23 |
| [#32](https://github.com/ClouDNS/octodns_cloudns/pull/32) | escaped TXT (`\;`) deletions | W4 (TXT part only) |
| [#41](https://github.com/ClouDNS/octodns_cloudns/pull/41) | DS support on `main` (0.1.0.dev0) through a per-id codec and reconcile in [`ds.py`](octodns_cloudns/ds.py); floor raised to octoDNS >= 1.22.0, Python >= 3.10; CI on 3.10/3.12 at the minimum and 3.14 at latest octoDNS | D5 implemented, phase 5 DS, closes #14 |
| issues [#33](https://github.com/ClouDNS/octodns_cloudns/issues/33)-[#40](https://github.com/ClouDNS/octodns_cloudns/issues/40) | HTTPS, SVCB, OPENPGPKEY, SMIMEA, CERT, RP, HINFO, Web Redirect | new; served by the codec table (5.1) |
| milestones | `0.0.18` (closed), `0.1.0` (open, no issues yet) | |

The [maintainer's answers in #25](https://github.com/ClouDNS/octodns_cloudns/issues/25#issuecomment-6100465234) (2026-10-10) change four design points; each is applied below:

- **Inactive rows are preserved by default**; reactivation is an explicit opt-in (D3 reversed, 5.3).
- **Rate limiting accounts for the shared egress IP**, not only shared credentials (5.1).
- **No blind retries after ambiguous write outcomes**; recovery is read-back through a fresh plan
  (5.1, 5.6).
- **Fail safely** on unsupported GeoDNS and on unparseable managed rows, instead of flattening,
  ignoring or deleting them (5.2, 5.7).

Plus: TTLs outside the account's list are rejected at plan time by default, rounding is opt-in (D4
as recommended); live fixtures must be sanitised; release checks include read-back and an empty
second plan; filing is #25 plus a few focused issues, not one per finding (see
[ISSUE-PLAN.md](ISSUE-PLAN.md)).

The most useful new fact for the design: `ds.py` already is the codec plus reconcile of section 5
for one record type (canonical primitive key, ids retained for equal values, leftovers paired into
`mod-record`, adds before mods before deletes, inactive rows outside the managed bucket, duplicate
and TTL drift repair via `_extra_changes`). The rewrite generalises it rather than introducing a
parallel design.

## 1. Goals and non-goals

Goals, in priority order:

1. ~~A TTL-only change must produce real API calls, using `dns/mod-record`, never
   delete-then-recreate.~~ Done in 0.0.18.
2. Replace the legacy value-update path (`_records_are_same` plus set difference, deletes before
   creates) with a per-record-id reconcile for every supported type that uses `dns/mod-record` for
   everything that changes in place and reserves `dns/add-record` and `dns/delete-record` for
   values that are genuinely added or removed. Combined TTL and value changes work in one run
   (0.0.18 rejects them).
3. Make failures loud everywhere: a change that yields zero API operations, or leaves a desired
   removal unmatched, fails the sync. 0.0.18 guards Updates only; Deletes still succeed silently
   (W6).
4. Fail safely: rows the provider cannot represent (GeoDNS locations, unparseable fields) stop the
   plan for that zone before any write; they are never flattened, ignored or deleted.
5. Leave the provider in a state where the 100 % coverage gate in `script/coverage` can be enabled
   in CI and stays enabled.

Non-goals for 0.1.0: GeoDNS support (fail-safe only, see 5.7 and D2), new record types beyond DS
(#33-#40, served by the codec table afterwards), octoDNS core changes.

## 2. What is broken

v1 found 29 issues (W1-W14 write path, R1-R10 read path, S1-S5 repository). The "where" links
point to 0.0.17 at `c29be64`; the "status" column is verified on `2c45212` with a mocked client
(octoDNS 1.22.0) and links there. Totals today: **9 fixed, 8 partly fixed, 12 open**.

### 2.1 Write path

| id | severity | symptom (0.0.17) | where (c29be64) | status at 2c45212 |
|---|---|---|---|---|
| W1 | critical | TTL-only Update makes zero API calls; `octodns-sync` reports success and re-plans the same change every run. | [L577-605](https://github.com/ClouDNS/octodns_cloudns/blob/c29be64/octodns_cloudns/__init__.py#L577-L605) | **fixed** (#28, [L666-695](https://github.com/ClouDNS/octodns_cloudns/blob/2c45212/octodns_cloudns/__init__.py#L666-L695)): one `record_mod` per row, live-validated for 16 types. |
| W2 | critical | Any Update of a CNAME, ALIAS or DNAME crashes with `AttributeError: ... no attribute 'values'`, leaving the plan half-applied. | [L584-587](https://github.com/ClouDNS/octodns_cloudns/blob/c29be64/octodns_cloudns/__init__.py#L584-L587) | **fixed** (#28, [L703-716](https://github.com/ClouDNS/octodns_cloudns/blob/2c45212/octodns_cloudns/__init__.py#L703-L716)) via `mod-record`. Residual: the target is sent with its trailing dot ([L224-236](https://github.com/ClouDNS/octodns_cloudns/blob/2c45212/octodns_cloudns/__init__.py#L224-L236) strips it only for MX/SRV); P14. |
| W3 | critical | TLSA Update and Delete crash: no TLSA branch in `_records_are_same`; the generic branch compares `TlsaValue == str` and raises. | [L714-721](https://github.com/ClouDNS/octodns_cloudns/blob/c29be64/octodns_cloudns/__init__.py#L714-L721) | **open** ([L850-868](https://github.com/ClouDNS/octodns_cloudns/blob/2c45212/octodns_cloudns/__init__.py#L850-L868)): both raise `AttributeError: '_equality_tuple'`. |
| W4 | critical | Value removal silently skipped for NS, MX, SRV, SSHFP, NAPTR, LOC and TXT with `;`: raw API strings are compared with value objects or dotted names. Non-convergent. | [L599-601](https://github.com/ClouDNS/octodns_cloudns/blob/c29be64/octodns_cloudns/__init__.py#L599-L601) | **partly fixed**: TXT `;` fixed (#32, [L733-735](https://github.com/ClouDNS/octodns_cloudns/blob/2c45212/octodns_cloudns/__init__.py#L733-L735)). Root NS `[a,b] -> [c,d]` still gives 2 creates and 0 deletes; an MX swap gives 1 create, 0 deletes ([L737-748](https://github.com/ClouDNS/octodns_cloudns/blob/2c45212/octodns_cloudns/__init__.py#L737-L748)). The zero-op guard does not fire because the call count is non-zero. |
| W5 | high | `change.new.values` is overwritten inside `plan.desired`; a second target sees truncated data. | [L603-604](https://github.com/ClouDNS/octodns_cloudns/blob/c29be64/octodns_cloudns/__init__.py#L603-L604) | **fixed** (#28, `change.new.copy()`, [L743-747](https://github.com/ClouDNS/octodns_cloudns/blob/2c45212/octodns_cloudns/__init__.py#L743-L747)). |
| W6 | high | CAA and LOC matching compares ints/floats with API strings, so those rows are never matched: Delete no-ops, Update creates duplicates. | [L643-678](https://github.com/ClouDNS/octodns_cloudns/blob/c29be64/octodns_cloudns/__init__.py#L643-L678) | **open** ([L790-825](https://github.com/ClouDNS/octodns_cloudns/blob/2c45212/octodns_cloudns/__init__.py#L790-L825)): CAA Delete makes 0 calls and succeeds, because `_apply_delete` has no zero-op guard ([L871-879](https://github.com/ClouDNS/octodns_cloudns/blob/2c45212/octodns_cloudns/__init__.py#L871-L879)); CAA Update adds a duplicate row. |
| W7 | high | Parameters string-formatted into a GET URL without encoding. | [L75-146](https://github.com/ClouDNS/octodns_cloudns/blob/c29be64/octodns_cloudns/__init__.py#L75-L146) | **fixed** (#28, form POST, [L99-129](https://github.com/ClouDNS/octodns_cloudns/blob/2c45212/octodns_cloudns/__init__.py#L99-L129)); live: `&`, `+`, `%`, `#`, `;` survive. |
| W8 | medium | `_data_for_TXT` does `rstrip('.')`: a TXT ending in `.` never round-trips (perpetual Update plus a duplicate per run). | [L290](https://github.com/ClouDNS/octodns_cloudns/blob/c29be64/octodns_cloudns/__init__.py#L290) | **open** ([L310](https://github.com/ClouDNS/octodns_cloudns/blob/2c45212/octodns_cloudns/__init__.py#L310)): `v=1.` reads back as `v=1`. |
| W9 | medium | Geo create sends the default values for every location, reduces `NA-US-CA` to `CA`, and accumulates `geodns-code` across the loop. | [L148-152](https://github.com/ClouDNS/octodns_cloudns/blob/c29be64/octodns_cloudns/__init__.py#L148-L152), [L437-461](https://github.com/ClouDNS/octodns_cloudns/blob/c29be64/octodns_cloudns/__init__.py#L437-L461) | **partly fixed**: no more accumulation ([L246-250](https://github.com/ClouDNS/octodns_cloudns/blob/2c45212/octodns_cloudns/__init__.py#L246-L250)); default values per location, subdivision-only codes and `locations = 0` remain ([L502-526](https://github.com/ClouDNS/octodns_cloudns/blob/2c45212/octodns_cloudns/__init__.py#L502-L526), [L628-642](https://github.com/ClouDNS/octodns_cloudns/blob/2c45212/octodns_cloudns/__init__.py#L628-L642)). |
| W10 | medium | Delete-before-create inside one Update leaves a window with zero records. | [L591-605](https://github.com/ClouDNS/octodns_cloudns/blob/c29be64/octodns_cloudns/__init__.py#L591-L605) | **open for value changes** ([L727-748](https://github.com/ClouDNS/octodns_cloudns/blob/2c45212/octodns_cloudns/__init__.py#L727-L748)); TTL-only no longer goes this way; DS orders adds first. |
| W11 | medium | NAPTR matching ignores service/regexp/replacement; sibling rows are returned twice and deleted twice. | [L615-623](https://github.com/ClouDNS/octodns_cloudns/blob/c29be64/octodns_cloudns/__init__.py#L615-L623) | **open** ([L762-770](https://github.com/ClouDNS/octodns_cloudns/blob/2c45212/octodns_cloudns/__init__.py#L762-L770)): two sibling rows produce four `record_delete` calls. |
| W12 | low | `_apply` calls `dns/get-zone-info` on every apply; `isGeoDNS` can never be True at its call site. | [L739-748](https://github.com/ClouDNS/octodns_cloudns/blob/c29be64/octodns_cloudns/__init__.py#L739-L748) | **open** ([L899-908](https://github.com/ClouDNS/octodns_cloudns/blob/2c45212/octodns_cloudns/__init__.py#L899-L908)); `zone` there is also an unused variable (pyflakes). |
| W13 | low | Record cache never updated inside an apply and kept after a failed apply. | [L374-380](https://github.com/ClouDNS/octodns_cloudns/blob/c29be64/octodns_cloudns/__init__.py#L374-L380), [L755](https://github.com/ClouDNS/octodns_cloudns/blob/c29be64/octodns_cloudns/__init__.py#L755) | **fixed** (#28): popped in `finally` ([L914-915](https://github.com/ClouDNS/octodns_cloudns/blob/2c45212/octodns_cloudns/__init__.py#L914-L915)); write-through on the TTL path only ([L694](https://github.com/ClouDNS/octodns_cloudns/blob/2c45212/octodns_cloudns/__init__.py#L694)). |
| W14 | info | Every `_params_for_*` value-formatting branch is dead code. | [L563-575](https://github.com/ClouDNS/octodns_cloudns/blob/c29be64/octodns_cloudns/__init__.py#L563-L575) | **open** ([L491-642](https://github.com/ClouDNS/octodns_cloudns/blob/2c45212/octodns_cloudns/__init__.py#L491-L642)); `record_create` uses `fields_from_value` instead. |

### 2.2 Read path

| id | severity | symptom (0.0.17) | where (c29be64) | status at 2c45212 |
|---|---|---|---|---|
| R1 | critical | `zone_records` swallows every API error; populate reports the zone as non-existent. | [L374-380](https://github.com/ClouDNS/octodns_cloudns/blob/c29be64/octodns_cloudns/__init__.py#L374-L380), [L418](https://github.com/ClouDNS/octodns_cloudns/blob/c29be64/octodns_cloudns/__init__.py#L418) | **fixed** (#28, [L429-443](https://github.com/ClouDNS/octodns_cloudns/blob/2c45212/octodns_cloudns/__init__.py#L429-L443)). |
| R2 | high | GeoDNS rows flattened into plain values while `SUPPORTS_GEO=True`. | [L396-417](https://github.com/ClouDNS/octodns_cloudns/blob/c29be64/octodns_cloudns/__init__.py#L396-L417) | **open** ([L462-482](https://github.com/ClouDNS/octodns_cloudns/blob/2c45212/octodns_cloudns/__init__.py#L462-L482), [L265](https://github.com/ClouDNS/octodns_cloudns/blob/2c45212/octodns_cloudns/__init__.py#L265)): two geo rows read as one plain A with both values. Write side: TTL, single-value and DS paths reject geo rows before any write ([L653-669](https://github.com/ClouDNS/octodns_cloudns/blob/2c45212/octodns_cloudns/__init__.py#L653-L669)); the multi-value and delete paths do not. |
| R3 | high | SSHFP and LOC spellings differ between read and write. | [L169](https://github.com/ClouDNS/octodns_cloudns/blob/c29be64/octodns_cloudns/__init__.py#L169), [L207-208](https://github.com/ClouDNS/octodns_cloudns/blob/c29be64/octodns_cloudns/__init__.py#L207-L208), [L349-357](https://github.com/ClouDNS/octodns_cloudns/blob/c29be64/octodns_cloudns/__init__.py#L349-L357) | **fixed and live** (#28, `_VALUE_FIELDS` [L166-203](https://github.com/ClouDNS/octodns_cloudns/blob/2c45212/octodns_cloudns/__init__.py#L166-L203)): write `fptype`/`lat-deg`, read `fp_type`/`lat_deg`. |
| R4 | high | Credentials in the URL, a Bearer header, full URL logged, `basicConfig(DEBUG)` at import. | [L11](https://github.com/ClouDNS/octodns_cloudns/blob/c29be64/octodns_cloudns/__init__.py#L11), [L57-91](https://github.com/ClouDNS/octodns_cloudns/blob/c29be64/octodns_cloudns/__init__.py#L57-L91) | **fixed** (#28, `_redact` [L78-85](https://github.com/ClouDNS/octodns_cloudns/blob/2c45212/octodns_cloudns/__init__.py#L78-L85), endpoint-only logging [L109](https://github.com/ClouDNS/octodns_cloudns/blob/2c45212/octodns_cloudns/__init__.py#L109)). |
| R5 | medium | An empty zone (`[]`) crashes populate. | [L397-399](https://github.com/ClouDNS/octodns_cloudns/blob/c29be64/octodns_cloudns/__init__.py#L397-L399) | **fixed** (#28, [L438-439](https://github.com/ClouDNS/octodns_cloudns/blob/2c45212/octodns_cloudns/__init__.py#L438-L439)). |
| R6 | medium | No timeout, no retry, GET for writes, HTTP call made while holding the throttle lock. | [L89-107](https://github.com/ClouDNS/octodns_cloudns/blob/c29be64/octodns_cloudns/__init__.py#L89-L107) | **partly fixed**: timeout and POST done ([L76](https://github.com/ClouDNS/octodns_cloudns/blob/2c45212/octodns_cloudns/__init__.py#L76), [L118-127](https://github.com/ClouDNS/octodns_cloudns/blob/2c45212/octodns_cloudns/__init__.py#L118-L127)); the request still runs inside the lock ([L111-127](https://github.com/ClouDNS/octodns_cloudns/blob/2c45212/octodns_cloudns/__init__.py#L111-L127)); no retry at all, including the safe `Request blocked.` case. |
| R7 | medium | Inactive rows (`status` 0) are read as live values. | [L399-405](https://github.com/ClouDNS/octodns_cloudns/blob/c29be64/octodns_cloudns/__init__.py#L399-L405) | **partly fixed**: DS excludes them ([L468-469](https://github.com/ClouDNS/octodns_cloudns/blob/2c45212/octodns_cloudns/__init__.py#L468-L469)); every other type still reads an inactive A as live. |
| R8 | medium | Per-record TTLs collapsed to one row's TTL (SRV and LOC use the leaked loop variable). | [L280-372](https://github.com/ClouDNS/octodns_cloudns/blob/c29be64/octodns_cloudns/__init__.py#L280-L372) | **open** ([L298-427](https://github.com/ClouDNS/octodns_cloudns/blob/2c45212/octodns_cloudns/__init__.py#L298-L427), [L399](https://github.com/ClouDNS/octodns_cloudns/blob/2c45212/octodns_cloudns/__init__.py#L399), [L407](https://github.com/ClouDNS/octodns_cloudns/blob/2c45212/octodns_cloudns/__init__.py#L407)); DS repairs mixed TTLs. |
| R9 | low | Only the 20 req/s tier is throttled, per client instance; a `Request blocked.` reply is fatal. | [L52-55](https://github.com/ClouDNS/octodns_cloudns/blob/c29be64/octodns_cloudns/__init__.py#L52-L55), [L93-104](https://github.com/ClouDNS/octodns_cloudns/blob/c29be64/octodns_cloudns/__init__.py#L93-L104) | **open** ([L53-56](https://github.com/ClouDNS/octodns_cloudns/blob/2c45212/octodns_cloudns/__init__.py#L53-L56)): 19 req/s per instance; nothing shared per IP. |
| R10 | low | `populate` logs `found -N records`. | [L406](https://github.com/ClouDNS/octodns_cloudns/blob/c29be64/octodns_cloudns/__init__.py#L406) | **open** ([L471](https://github.com/ClouDNS/octodns_cloudns/blob/2c45212/octodns_cloudns/__init__.py#L471), [L486](https://github.com/ClouDNS/octodns_cloudns/blob/2c45212/octodns_cloudns/__init__.py#L486)). |

### 2.3 Tests, packaging, security

| id | severity | finding (0.0.17) | status at 2c45212 |
|---|---|---|---|
| S1 | critical | A ClouDNS sub-user id and password committed in the test module since PR #20. | **partly fixed**: removed, live tests need explicit opt-in and environment credentials (#28); rotation confirmed by the credential's owner in #23. No secret scanner in CI or pre-commit. |
| S2 | high | Coverage 44 %; tautological tests; `script/coverage` demands 100 %. | **partly fixed**: 208 passed, 24 live checks skipped; coverage 86 % (`ds.py` 100 %, `__init__.py` 84 %). `_records_are_same` and `_apply_delete` are uncovered. |
| S3 | high | CI runs bare `pytest`; `script/cibuild` never runs and fails all four checks. | **partly fixed**: CI tests Python 3.10 and 3.12 at octoDNS 1.22.0 and 3.14 at latest, with `--disable-network`. Still bare `pytest`; `script/cibuild` fails black (10 files at `-S -l 80`), isort (5 test files), pyflakes (4 findings) and the coverage gate. |
| S4 | medium | Packaging metadata wrong; no `pyproject.toml`, `CHANGELOG.md`, `.gitignore`; dangling pre-commit symlink; script defects. | **partly fixed**: floors, MIT, `CHANGELOG.md`, `.gitignore`, `requirements.txt` done. Open: `url` points to a 404, deprecated `tests_require`, no `pyproject.toml`, `.git_hooks_pre-commit` missing, `script/format` ignores `VENV_NAME`, `cibuild-setup-py` uses `setup.py install`, and a second script directory `scripts/` (`publish_release.py`) next to octodns-org's `script/`. |
| S5 | low | `tests/config.yaml` (wrong class path) and `tests/config/recordsfortest.bg.yaml` unused. | **open**. |

## 3. ClouDNS API contract the rewrite codes against

Sources: wiki articles 41, 42, 43, 48, 57, 58, 59, 60, 66, 134, 153, 183, 184, 188, 434,
ClouDNS's [PHP SDK](https://github.com/ClouDNS/cloudns-php-sdk) and
[Go client](https://github.com/ClouDNS/cloudns-go), go-acme/lego, dnscontrol and libdns, and since
v2 upstream's live validation of 2026-10-11.

### 3.1 Endpoints and envelope

- All endpoints accept GET or POST. The client POSTs a form body via `requests` `data=` (in place
  since 0.0.18), which URL-encodes every value.
- Auth is `auth-id` | `sub-auth-id` | `sub-auth-user` plus `auth-password` as ordinary parameters.
  API users may be IP-restricted at creation (article 42).
- The HTTP API is a paid-plan feature, and TTLs can only be changed on Premium, DDoS-protected or
  GeoDNS subscriptions (article 188).
- Every failure is HTTP 200 with `{"status":"Failed","statusDescription":"..."}`. Known texts:
  `Missing domain-name`, `Invalid record-id param.`,
  `Invalid TTL. Choose from the list of the values we support.`,
  `This record type is not supported.`,
  `Invalid authentication, incorrect auth-id or auth-password.`,
  `Request blocked. <ip> is sending more than 20 requests per second.`,
  `There is another record for the same host.` (live, CNAME), and for `dns/register`:
  `<domain> has been already added.`, `<domain> is invalid domain name.`,
  `Your plan does not support GeoDNS zones.`
- `dns/records` returns an object keyed by record id (string); each row also carries `id`. Scalars
  are strings except `status` (int). An empty zone returns `[]`. The apex host is `""`. Hostname
  targets come back without a trailing dot. ClouDNS's default apex NS rows are ordinary rows with
  ids. With `include-notes=1` the note comes back under the singular key `note` (live).
- `dns/add-record` returns `data.id` (number or string: P3).
- `dns/mod-record` requires `domain-name`, `record-id`, `host`, `ttl`, plus the type-specific
  parameters. It cannot change the type and has no `status` parameter. **Live:** resending the full
  value with a new TTL keeps the id, every value field, inactive status, notes and failover
  settings. Omitting `status`, `note` and failover parameters preserves them. On a GeoDNS zone,
  omitting `geodns-location` moves regional rows to DEFAULT (A, AAAA, ALIAS, SRV, NAPTR) or is
  rejected (CNAME); sending the row's numeric `geodns-location` preserves everything. Whether a
  *value* change via `mod-record` also preserves notes and failover is not established (P13).
- `dns/delete-record` takes `domain-name`, `record-id`. `dns/change-record-status` takes
  `domain-name`, `record-id`, `status` (1/0; omitting it toggles).
- `dns/register` takes `domain-name`, `zone-type` (`master|slave|parked|geodns`) and optional
  `ns[]`. Without `ns[]` ClouDNS seeds its default NS rows at the apex; with `ns[]` "default NS
  records will be not added" (article 48; P10).
- Rate limits **per IP**: 20 req/s, 600 req/min, 36,000 req/h (article 41). Sustained rate is
  therefore 10 req/s, shared by every process and credential behind one egress address.
- Allowed TTLs come from `dns/get-available-ttl` per account. Live: 120 and 7200 were rejected on
  the test account, 300 and 3600 accepted. The wiki list is a fallback, not a contract.

### 3.2 Per-type parameter map (add and mod are identical)

| type | write params | list fields read | canonical key (value only, no ttl) |
|---|---|---|---|
| A, AAAA | `record` | `record` | normalised IP |
| NS, PTR | `record` | `record` (no dot) | fqdn with dot |
| CNAME, ALIAS, DNAME | `record` | `record` (no dot) | fqdn with dot |
| TXT, SPF | `record` = raw text (`\;` unescaped) | `record` -> `TxtValue.normalize_raw_text()` | raw text, dots kept |
| MX | `record` (exchange, no dot), `priority` | `record`, `priority` | `(pref, exchange.)` |
| SRV | `record` (target, no dot), `priority`, `weight`, `port` | same | `(prio, weight, port, target.)` |
| CAA | `caa_flag`, `caa_type`, `caa_value` | same (strings) | `(flags, tag, value)` |
| SSHFP | `record`, `algorithm`, `fptype` (live) | `fp_type` (live) | `(alg, fptype, fp lowercased)` |
| TLSA | `record`, `tlsa_usage`, `tlsa_selector`, `tlsa_matching_type` | same | `(usage, sel, match, data lowercased)` |
| NAPTR | `order`, `pref`, `flag`, `params`, `regexp`, `replace` | same | all six fields |
| LOC | `lat-deg` ... `v-precision` (dashed, live) | `lat_deg` ... `v_precision` (underscored, live) | all twelve fields |
| DS | `key_tag`, `algorithm`, `digest_type`, `record` | same | `(key_tag, alg, digest_type, digest lowercased)` (as `DsCodec.key`) |

Today NS, PTR, CNAME, ALIAS and DNAME are written *with* the trailing dot that octoDNS carries
(only MX and SRV are stripped). ClouDNS evidently accepts that (live TTL validation reads them back
without it), but whether it normalises or stores it differently is not established (P14). The codec
strips it uniformly, matching what both SDKs do.

## 4. octoDNS contract constraints (octodns 1.22.0, verified from source)

- `change.new` **is** the record object in `plan.desired`; `change.existing` **is** the record that
  `populate` built. Never assign to their attributes. Use `record.copy()` or local structures.
- An `Update` is emitted when values differ, or TTL differs, or (with `SUPPORTS_GEO`) geo differs.
  Record identity is `(name, type)` only, so a type change is always Delete + Create.
- `Plan.changes` is sorted Delete < Create < Update. Keep that order so CNAME <-> A swaps delete
  first.
- `BaseProvider.apply()` returns `len(plan.changes)` regardless of what `_apply` did. The
  in-contract way to be loud is to raise `ProviderException` from `_apply`.
- `populate(zone, target, lenient)` returns `True` if the zone exists (even with zero records) and
  `False` only if it does not.
- Provider limitations belong in `_process_desired_zone` (copy, adjust, `add_record(replace=True)`,
  `supports_warn_or_except`), `_include_change` and `_extra_changes`. `strict_supports` defaults
  to True.
- TXT round trip: `TxtValue.normalize_raw_text(raw)` when reading, `value.to_raw_text()` when
  writing. Never strip dots.
- `SUPPORTS_GEO` is a mandatory class attribute; geo records are deprecated (removal in 2.0).
- Every structured value class is hashable in 1.22 and exposes typed attributes (ints/floats), so
  API strings must be cast before comparison; `DsCodec.key` shows the pattern.
- The manager creates one provider instance per config entry and shares it across zone threads when
  `max_workers > 1`. Several provider entries (and therefore several clients) can run in one
  process. Caches, sessions and the rate limiter are shared state.

## 5. Target design

The core is unchanged from v1 and now proven in upstream's own `ds.py`: one codec table, a
per-record-id reconcile that keeps equal values, pairs leftovers into `mod-record`, orders adds
before mods before deletes, and fails loudly. It follows octodns-cloudflare's `_apply_Update`
([octodns_cloudflare/__init__.py:806-934](https://github.com/octodns/octodns-cloudflare/blob/main/octodns_cloudflare/__init__.py#L806-L934)),
the only octodns-org provider whose backend has the same per-record-id shape.

### 5.1 Module layout

- `octodns_cloudns/client.py`: `ClouDNSClient`, moved from `__init__.py` with its current
  behaviour (form POST, timeout, `_redact`, `record_mod`, `include-notes=1`), then extended:
  - `zone_info`, `zone_create(domain, zone_type, ns=None)`, `records`, `record_add`, `record_mod`,
    `record_delete`, `record_set_status`, `available_ttls`, `records_count`.
  - One ordered table mapping `statusDescription` to typed exceptions: `ClouDNSMissingDomain`,
    `ClouDNSZoneExists`, `ClouDNSAuthError`, `ClouDNSRateLimited`, `ClouDNSInvalidTtl`,
    `ClouDNSInvalidRecordId`, `ClouDNSPlanLimit`, else `ClouDNSClientException`. The existing
    public exception classes stay importable.
  - **Rate limiter shared per egress IP:** one process-wide limiter (module-level, not per client or
    per account), because ClouDNS counts per source address and every provider entry in a process
    shares it. Fixed minimum interval, default 10 req/s (satisfies 20/s, 600/min and 36,000/h), the
    sleep taken outside the lock and the HTTP call made outside it too, `penalise()` after a
    throttle reply. `calls_per_second` lets operators divide the budget when several processes share
    one address; that cannot be coordinated in-process and is documented as such.
  - **Retries only where the outcome is unambiguous:** `Request blocked.` on any endpoint (the
    request was not processed), and `ConnectTimeout` (nothing was sent). Reads may also retry on
    other transport errors. A write whose outcome is ambiguous (read timeout, connection reset after
    send, 5xx) is **never** retried: the client raises, the provider clears its cache, and the
    error says to re-plan, which reads back the real state. This keeps 0.0.18's "do not retry
    ambiguous writes" and adds only the safe cases.
- `octodns_cloudns/codec.py`: one table `TYPES` generalising `_VALUE_FIELDS` and `DsCodec`. Per
  type: write fields, octoDNS attribute, read aliases (the current `_VALUE_FIELDS` triples),
  `to_fields(value)`, `from_row(row)` and `key()`. The key is a canonical primitive tuple derived
  from the same fields that become the payload, with ints cast, hex lowercased and hostnames
  normalised to one dot convention, as `DsCodec.key` does today. `from_row` raises a dedicated
  `UnparseableRow` naming the row id for anything it cannot parse; it never returns a partial
  value. A frozen `Row` dataclass carries `id, type, host, ttl, status, location, note, failover,
  key, raw`. DS becomes one table entry with unchanged behaviour. Each of #33-#40 that octoDNS can
  model becomes one entry plus its tests. Retires `_records_are_same`, `_params_for_*` and
  `_data_for_*` (fixes W3, W4, W6, W8, W11, W14).
- `octodns_cloudns/reconcile.py`: a pure function `reconcile(rows, wants, options) -> [Op]` with
  `Op = Add | Mod | Activate | Delete`, no I/O and no octoDNS imports, generalising
  `ds.operations`. Every scenario is an exact ordered-op table test.
- `octodns_cloudns/provider.py`: `ClouDNSProvider` with `populate`, `_process_desired_zone`,
  `_extra_changes`, `_apply` and thin `_apply_create/update/delete` that call `reconcile` and
  execute ops through the client with a write-through cache.
- `octodns_cloudns/__init__.py`: re-exports `ClouDNSProvider`, `ClouDNSClient`, the exception
  classes and `__version__`, so the documented class path and existing imports keep working.

Public constructor stays `ClouDNSProvider(id, auth_id, auth_password, sub_auth=False)`. New
keyword-only options, defaults chosen per the maintainer's answers:

| option | default | opt-in |
|---|---|---|
| `inactive_records` | `preserve`: inactive rows are left alone and outside the managed set | `reactivate`: an inactive row equal to a desired value is reactivated instead of adding a duplicate |
| `ttl_policy` | `reject`: a TTL outside the account's list fails at plan time | `round_up`: the plan shows and writes the next allowed TTL, with a WARNING |
| `validate_ttl` | `api`: `dns/get-available-ttl` once per run, documented list on error | `static` |
| `calls_per_second` | 10 (process-wide) | lower when several processes share one IP |
| `timeout` | `(10, 30)` as today | |
| `zone_type` | `master` for auto-created zones | `geodns` only once GeoDNS is supported |

### 5.2 Populate keeps the ids and fails safe

`zone_rows(zone)` calls `records()` (which coerces `[]` to `{}`), returns `None` on
`ClouDNSMissingDomain`, and lets every other exception propagate (as 0.0.18 already does). Each row
of a supported type is parsed through the codec into a `Row` and stored in
`self._zone_rows[zone.name]`; unsupported types are skipped with one INFO line per zone.

- **Unparseable active rows of a supported type fail safe:** populate raises naming the row id,
  host and type, also under `lenient` (as `ds.py` does for DS today). The row is never ignored,
  silently dropped from the values, or deleted by repair. Unparseable inactive rows are left alone
  under `inactive_records: preserve`.
- **Rows carrying a GeoDNS location fail safe** (5.7).
- Inactive rows (`status` 0) are kept in the side table but excluded from the octoDNS values (R7).
- Duplicate active rows with the same key: lowest id is the live one; the rest are repaired by
  `_extra_changes` (as DS does).
- Mixed TTLs inside one name/type: populate reports `min(ttl)` and `_extra_changes` emits a repair
  Update listing the row ids (R8).
- `exists` is `True` whenever the listing succeeded, `False` only on `ClouDNSMissingDomain`.
  `before` is captured before the loop (R10).

### 5.3 Reconcile

```
rows  = side-table rows for (host, type)
wants = one Want(key, ttl, fields) per desired value (from change.new, read-only)

active   = rows with status 1, grouped by key; lowest id first
inactive = rows with status 0                      # outside the managed set by default

for key in wants:
    if key in active:      keep the first row; Mod it if its ttl != want.ttl; rest -> spare
    elif reactivate and key in inactive:
                           Activate (and Mod first if ttl differs)
    else:                  -> new
spare += every active row whose key is not wanted

pair new with spare -> Mod on the spare's id        (a value swap is one call; a single-value
                                                     type or the root NS set never goes empty)
ops = Adds(unpaired new) -> Mods -> Activates -> Deletes(unpaired spare)
```

This is `ds.operations` with three additions: the `reactivate` branch (opt-in), a location bucket
so pairing never crosses locations once GeoDNS returns, and a collected zero-op check.

- `_apply_update` raises if `ops` is empty. To avoid aborting a half-applied plan on one false
  positive, zero-op changes are collected and one `ProviderException` naming them is raised after
  the loop.
- `_apply_create` runs the same function (normally all Adds; matching stray active rows are kept).
- `_apply_delete` deletes every **active** row of that host/type, sorted by id, and raises if there
  is none (closes W6's silent delete). Inactive rows are preserved by default; whether a Delete of
  the whole name/type should also remove them is D8.
- TTL and value changes are allowed in the same Update (0.0.18 rejects them).
- `_apply` preflights every change through `reconcile` before the first write (as 0.0.18 and DS
  already do), creates the zone from `plan.exists` without an extra `get-zone-info` (W12), writes
  new ids from `record_add` through to the side table, and clears the cache in a `finally`.

Call counts (0.0.18 behaviour in parentheses):

| scenario | calls |
|---|---|
| TTL-only change on a 3-value A record | 3 mod (3 mod) |
| one value swapped in a 3-value TXT | 1 mod (1 delete + 1 add) |
| CNAME target change | 1 mod (1 mod) |
| add one CAA value | 1 add (1 add; the existing value is never matched, so later runs duplicate) |
| delete a CAA record | 1 delete per row (0 calls, reports success) |
| replace all 20 values of a TXT | 20 mod (20 delete + 20 add, with an empty window) |
| root NS `[a,b] -> [c,d]` | 2 mod (2 add; a and b left live) |
| TTL and one value change together | 1 mod per row (rejected before any write) |
| desired value present only as an inactive row | 1 add, inactive row kept (default); 1 activate with `reactivate` |

### 5.4 Zone auto-create

When `plan.exists` is False: if the desired zone has a root NS record, pass its values as `ns[]` to
`dns/register` so ClouDNS seeds no default NS rows; otherwise register without `ns[]` and re-list
the zone before applying, so the seeded default NS rows are reconciled instead of added beside.
`has been already added.` means the zone exists: re-list. `Your plan does not support GeoDNS
zones.` is a hard error.

### 5.5 TTL policy and drift repair

In `_process_desired_zone`: fetch the allowed list (`dns/get-available-ttl` once per run, documented
list on error). A record with a TTL outside the list raises at plan time under the default
`ttl_policy: reject`, naming the record, the TTL and the allowed values. This is independent of
`strict_supports`, so rounding never happens without the explicit `ttl_policy: round_up`; then the
record is copied with the next allowed TTL, `add_record(replace=True)`, and a WARNING is logged.

`_extra_changes` emits a repair Update for any managed name/type with mixed TTLs, duplicate active
rows, or (only with `inactive_records: reactivate`) inactive rows equal to a desired value. Each
logs an INFO line `drift at <name> <type>` with the row ids, because the plan line itself shows no
visible difference. This generalises the DS repair in `__init__.py` today. Repair Updates count
toward `Plan.raise_if_unsafe()` thresholds; document that a first run on a drifted zone may need
`--force`.

### 5.6 Error handling and loudness

- Any `Failed` status on a write raises with the redacted `statusDescription`.
- Ambiguous write outcomes are never retried (5.1); the cache is cleared and the error tells the
  operator to re-plan. Because the reconcile is idempotent, the next plan converges from the
  read-back state.
- Zero-op Updates and Deletes fail the sync (5.3).
- `ClouDNSInvalidTtl` at apply time (account list changed between plan and apply) names the
  record, the TTL and the allowed list.
- Value changes on rows with notes or failover: 0.0.18 refuses these before any write. 0.1.0 keeps
  refusing (fail safe, naming the row ids) until P13 shows `mod-record` preserves them on a value
  change. Add plus Delete is no alternative: deleting the row destroys its note and failover check.
  TTL-only Mods on such rows stay allowed (live-validated).
- Thread safety: side tables keyed per zone, cache writes under a lock, process-wide limiter, one
  `requests.Session` per client guarded by the limiter.

### 5.7 GeoDNS

Maintainer requirement: **fail safely on unsupported GeoDNS.** Proposal (D2):

- `SUPPORTS_GEO = False` in 0.1.0. octoDNS deprecates geo records and the current read and write
  mappings are wrong (W9, R2).
- populate raises for any managed name/type that has a row with a non-DEFAULT location, naming the
  rows and pointing to the release notes. Nothing is flattened.
- DEFAULT-location rows on a GeoDNS zone are representable as plain records. They may be managed as
  static records **only** if every write sends the row's numeric `geodns-location` explicitly (the
  live probe showed omission is only harmless for DEFAULT rows, and CNAME regional rows reject it).
  Whether to support that in 0.1.0 or fail the whole GeoDNS zone is D9.
- The codec `Row` carries the location and the reconcile pairs inside location buckets, so geo
  support can return later (legacy `geo` or `dynamic` with geo-only rules, after P7) without
  touching the reconcile.

## 6. Remaining plan

Phases 0 and 2 of v1 are complete upstream; phase 1 is partly done (floor bump, CHANGELOG,
`.gitignore`, CI matrix). The rest is grouped into the four focused issues the maintainer asked for;
[ISSUE-PLAN.md](ISSUE-PLAN.md) has the issue bodies and the PR series. Every PR branches from
`upstream/main`, is small and independently tested, and keeps formatting separate. Releases and
version bumps are ClouDNS's. PR ids follow the numbering inside each track (track D item 2 is
PR-D2); the `PR-` prefix keeps them apart from the decisions D1 to D10.

### Track A: tooling and compatibility

1. Packaging and lint (no reformatting): `pyproject.toml` with the octodns-org black/isort/pytest
   configuration, `url` fix, `tests_require` dropped, `.git_hooks_pre-commit` shipped, `script/format`
   honours `VENV_NAME`, `cibuild-setup-py` uses `pip install .`, the four pyflakes findings, stale
   `tests/config.yaml` and `tests/config/recordsfortest.bg.yaml` removed (S5), a secret scanner in CI
   and pre-commit (S1).
2. One formatting-only PR (`script/format`), its hash in `.git-blame-ignore-revs`. Timed with the
   maintainer, because it touches every file and upstream commits actively.
3. CI runs `script/cibuild` and `script/cibuild-setup-py`, keeping upstream's minimum-version legs
   (Python 3.10/3.12 at octoDNS 1.22.0) next to the `.ci-config.json` matrix at the latest octoDNS.
   The coverage gate stays off until track D removes the legacy code.

### Track B: API client

1. Extract `client.py` unchanged in behaviour, with re-exports (pure move, existing tests green).
2. Typed exception table, process-wide rate limiter per egress IP with the call made outside the
   lock, retries only on `Request blocked.` and `ConnectTimeout`, ambiguous writes never retried,
   `records_count`, `available_ttls`, `record_set_status`, `zone_create(ns=...)`. `requests_mock`
   tests assert the form body, credentials absent from the URL and every log record, and the retry
   matrix per endpoint and failure kind, with an injected clock.

### Track C: codecs

1. `codec.py` as a pure addition: the `TYPES` table, `Row`, `UnparseableRow`, and a table-driven
   round-trip test per type (value -> fields -> sanitised fixture row -> value; `key()` equal on both
   sides; read aliases; TXT `\;` and trailing dots; hex case; hostname dots).
2. Move DS onto the table; `DsCodec` becomes a thin alias, all 49 DS tests unchanged and green.

### Track D: reconciliation and provider integration

1. `reconcile.py` as a pure addition, with exact ordered-op tests for the scenario table in 5.3 plus
   inactive (preserve and reactivate), duplicates, 20 -> 25, 20 -> 15, root NS never empty, no
   cross-bucket pairing, Create against matching stray rows. Move `ds.operations` onto it.
2. Provider rewired: populate on the side table with fail-safe parsing (5.2), `_apply_*` through
   `reconcile` with preflight, write-through cache and collected zero-op failure (5.3),
   `SUPPORTS_GEO = False` with the GeoDNS fail-safe (5.7), `SUPPORTS_MULTIVALUE_PTR = True`.
   Mutation guard with two provider instances sharing one plan; a Manager-level plan -> apply ->
   plan test that ends with an empty plan.
3. TTL policy and drift repair (5.5).
4. Zone auto-create with `ns[]` (5.4).
5. Legacy code and the tests that pin it removed; `--cov-fail-under=100` on in CI.
6. Gated live integration class on an isolated test zone: runs the scenario table, **reads back**
   every row after apply, asserts the **second plan is empty**, unique names per run, cleanup in
   `finally`, baseline snapshot compared by id. This is the release gate the maintainer asked for.
7. README and CHANGELOG: supported types, TTL policy and plan requirement, inactive and duplicate
   semantics, GeoDNS fail-safe, config keys, the correct class path, migration notes.

Done when: every row of section 2 has a green regression test, coverage is 100 %, the live
integration run passes with read-back and an empty second plan, and the CHANGELOG names every
behaviour change. ClouDNS then releases 0.1.0.

### Optional: a 0.0.19 fail-loud patch (D7)

The maintainer reserved a separate branch for 0.0.19. Three legacy-path defects still fail
silently or crash in 0.0.18 and are cheap to make *loud* without the rewrite: a Delete that matches
no API row (W6 CAA/LOC), an Update whose removals match no API row (W4 NS/MX/SRV and friends), and
the TLSA crash (W3) turned into a clear `ProviderException`. Only if the maintainer wants it.

### After 0.1.0

- ClouDNS-side: wiki article 509 class path, the `id` vs `record-id` rows in articles 59/60, the
  PyPI home page `url`.
- Operations (outside this repository): a TTL drift sweep across all ClouDNS-managed zones after the
  first 0.1.0 sync; `deflect.me` proves drift exists.
- Follow-ups: record types #33-#40 as codec entries; GeoDNS via `dynamic` after P7; pagination guard
  only if P4 shows truncation; the "reported change count vs. real operations" question in octoDNS
  core.

## 7. Open questions

### 7.1 Live probes

Run against an isolated test zone with a dedicated sub-user; every captured response is sanitised
(zone names, record ids, notes, IP addresses, any account identifier replaced by stable
placeholders) before it is committed as `tests/fixtures/cloudns-*.json`.

| # | question | status | needed by |
|---|---|---|---|
| P1 | SSHFP/LOC parameter spellings on write and read | **answered (live):** write `fptype`, `lat-deg`...; read `fp_type`, `lat_deg`... | done |
| P2 | Does `mod-record` with only the TTL changed keep everything else? | **answered (live):** yes for value fields, status, notes and failover on static rows; GeoDNS needs the explicit `geodns-location` | done |
| P3 | Is `data.id` from add-record a number or a string? | open | PR-D2 (write-through cache) |
| P4 | Does `dns/records` return zones with more than 100 rows completely? | open | PR-D2 |
| P5 | Are `status=0` rows listed, and does `mod-record` reset status? | **answered (live):** listed; a TTL mod preserves status | done |
| P6 | How do TXT values over 255 bytes look in the listing? | open | PR-C1 |
| P7 | GeoDNS location field and code vocabulary | **partly answered (live):** created via `geodns-code`, modified via numeric `geodns-location`; vocabulary and read mapping open | after 0.1.0 |
| P8 | Does add-record reject an identical duplicate or create it? | open | PR-D1 |
| P9 | Can the default apex NS rows be deleted or modified, and does `mod-record` work on a monitored row? | open (TTL mod on monitored rows: live yes) | PR-D4 |
| P10 | Does `dns/register` with `ns[]` seed no default NS rows? | open | PR-D4 |
| P11 | Sub-user error text for a zone outside its permissions; are unknown extra parameters ignored? | open | PR-B2 |
| P12 | Reply when the 600/min tier is exceeded; punycode or Unicode for IDN names? | open | PR-B2, PR-C1 |
| P13 | Does a *value* change via `mod-record` preserve notes, failover and inactive status? | **new**, open | PR-D2 (5.6) |
| P14 | Are hostname values written with a trailing dot stored and listed identically to dot-less ones? | **new**, open | PR-C1 |

### 7.2 Maintainer decisions

| # | decision | status |
|---|---|---|
| D1 | License | **resolved:** MIT in `LICENSE` and `setup.py` (#26). |
| D2 | GeoDNS in 0.1.0 | **decided:** fail safely. Proposal in 5.7 (`SUPPORTS_GEO = False`, populate refuses non-DEFAULT rows); details are D9. |
| D3 | Inactive rows | **decided:** preserve by default; reactivation opt-in (`inactive_records: reactivate`). |
| D4 | Unsupported TTLs | **decided:** reject at plan time by default; rounding opt-in (`ttl_policy: round_up`). |
| D5 | Floor octoDNS 1.22 / Python 3.10 | **decided and implemented** on `main` for 0.1.0 (#41). |
| D6 | 0.0.18 first | **decided and released** 2026-10-11. |
| D7 | A 0.0.19 fail-loud patch for the legacy path (section 6)? | **open**; recommendation: yes, small, guards only. |
| D8 | Should deleting a whole name/type also remove its inactive rows? | **open**; recommendation: no by default (preserve), removed only with `inactive_records: reactivate`, which signals that octoDNS owns them. |
| D9 | GeoDNS zones whose managed rows are all DEFAULT: manage them (sending `geodns-location`) or fail the zone? | **open**; recommendation: fail the zone in 0.1.0, add DEFAULT-only support with P7. |
| D10 | When to land the formatting-only PR | **open**; needs a quiet window on `main`. |

## 8. Risks and mitigations

| risk | mitigation |
|---|---|
| `mod-record` on a value change loses notes or failover | P13; until answered, value changes on such rows are refused before any write, as in 0.0.18 (5.6). |
| Ambiguous write (timeout after send) leaves unknown state | never retried; cache cleared; the error says to re-plan; the reconcile is idempotent, so the next plan converges from read-back. |
| Rate limit hit mid-apply, or several processes behind one IP | process-wide limiter at 10 req/s; `calls_per_second` to divide the budget across processes; `Request blocked.` is retried with backoff because it is unambiguous. |
| Fail-safe parsing blocks a zone that 0.0.18 synced | the error names the row ids and the fix; CHANGELOG lists it as a behaviour change; no silent fallback. |
| Drift repair surprises operators | INFO lines list the row ids; repair Updates count toward safety thresholds; inactive rows are never touched without `reactivate`. |
| Account TTL list differs from the documented one | fetched per run; the static list is only a fallback. |
| Formatting PR conflicts with upstream's own work | separate PR, timed with the maintainer (D10). |
| Refactor breaks DS | DS moves onto the shared codec and reconcile last within each track, with its 49 tests unchanged. |

## 9. Appendix: evidence trail

- v1 reproductions (0.0.17, mocked client, octoDNS 1.22.0) are in tag `roadmap-v1`.
- v2 reproductions (`2c45212`, mocked client, octoDNS 1.22.0): TLSA Update/Delete ->
  `AttributeError: '_equality_tuple'`; NS `[a,b] -> [c,d]` -> 2 creates, 0 deletes; MX swap -> 1
  create, 0 deletes; CAA Delete -> 0 calls, no error; CAA Update -> 1 create, old row kept; TXT `v=1.`
  reads back as `v=1`; two NAPTR siblings -> 4 deletes; two geo rows -> one A with both values;
  inactive A read as live; mixed-TTL rows -> one TTL; TTL 123 plans without error. Fixed cases
  re-checked: TTL-only A/CAA -> one `record_mod` per row; CNAME update -> one `record_mod`;
  `change.new.values` untouched; populate propagates an auth error; `[]` -> `exists=True`.
- Test suite at `2c45212`: 208 passed, 24 live checks skipped (Python 3.13, octoDNS 1.22.0);
  coverage 86 %.
- Live evidence: [docs/live-validation-0.0.18.md](docs/live-validation-0.0.18.md) (16 static
  types, metadata matrix, GeoDNS location behaviour); [docs/ds.md](docs/ds.md) (DS semantics,
  synthetic fixtures only).
- ClouDNS SDK evidence: PHP SDK `dnsModifyRecord` sends `record-id`, `fptype`, `lat-deg`,
  `h-precision`; Go client `updaterec` uses the same names. Article 48: `ns[]` suppresses default
  NS rows. Article 188: TTL changes need Premium/DDoS/GeoDNS subscriptions. Article 41: limits per
  IP.
- Rate-limit text: dnscontrol `providers/cloudns/api.go` retries on `Request blocked.`.
- Empty-zone `[]`: go-acme/lego and libdns/cloudns clients.
- Upstream: maintainer boyanpeychev; 0.0.18 on PyPI and as GitHub release `v0.0.18`
  (2026-10-11); `main` at 0.1.0.dev0; milestones `0.0.18` (closed) and `0.1.0` (open).

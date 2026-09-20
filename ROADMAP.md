# Roadmap: fix the silent TTL no-op and move octodns-cloudns to update-based reconciliation

Status: v1, 2026-09-20. Applies to `octodns_cloudns` 0.0.17 in this fork (`istr/octodns_cloudns`);
upstream is [ClouDNS/octodns_cloudns](https://github.com/ClouDNS/octodns_cloudns), which also owns the
PyPI package `octodns-cloudns`.

How to read this document: section 2 is what is broken and how we know; sections 3 and 4 are the
two contracts the rewrite must respect (ClouDNS API, octoDNS core); section 5 is the target design;
section 6 is the phased plan; section 7 lists what needs a live probe and what needs a maintainer
decision. Everything marked **verified** was reproduced on this checkout with octoDNS 1.22.0 and a
mocked HTTP client, or read from the raw ClouDNS wiki pages and ClouDNS's own SDKs. Items marked
**probe** need one live call against a throwaway zone before code may rely on them.

## 1. Goals and non-goals

Goals, in priority order:

1. Fix [BUG-REPORT.md](BUG-REPORT.md) section 2: a TTL-only change must produce real API calls, and
   it must use `dns/mod-record`, never delete-then-recreate.
2. Replace the brute-force update path (`_apply_update` deleting and re-adding API records) with a
   per-record-id reconcile that uses `dns/mod-record` for everything that changes in place and
   reserves `dns/add-record` and `dns/delete-record` for values that are genuinely added or removed.
3. Make failures loud: a change that yields zero API operations must fail the sync, never report
   success.
4. Leave the provider in a state where the 100 % coverage gate already present in
   `script/coverage` can be enabled in CI and stays enabled.

Non-goals: GeoDNS (explicitly deferred, see 5.7 and D2), new record types (DS, HTTPS/SVCB; tracked
as follow-ups), and octoDNS core changes.

## 2. What is broken (verified)

The TTL bug is one instance of a structural problem: the provider matches API records to octoDNS
values by re-deriving strings in three places (`_data_for_*`, `_records_are_same`, `_apply_update`)
that disagree with each other, and it forgets the record ids that `dns/records` hands it. The cases
below were reproduced with `provider._client = Mock()` and a realistic `dns/records` cache; they
become the regression tests of phase 1. 44 findings were produced and each was challenged by three
independent reviewers; the 41 that survived are listed here, the 3 refuted ones (apex host `@`,
implicit pagination, verbatim host matching) were hypotheticals contradicted by evidence.

### 2.1 Write path

| id | severity | symptom | where |
|---|---|---|---|
| W1 | critical | TTL-only Update makes zero API calls for A, AAAA, TXT, SPF, NS, PTR, MX, SRV, SSHFP, NAPTR, CAA and LOC. `apply()` still returns `len(plan.changes)`, so `octodns-sync` reports success and re-plans the same change every run. | [__init__.py:577-605](octodns_cloudns/__init__.py#L577-L605) |
| W2 | critical | Any Update of a CNAME, ALIAS or DNAME record crashes with `AttributeError: 'CnameRecord' object has no attribute 'values'` (only `TypeError` is caught). Deletes and Creates sort before Updates, so the plan is left half-applied. Present since PR #21 on every octoDNS version. | [__init__.py:584-587](octodns_cloudns/__init__.py#L584-L587) |
| W3 | critical | TLSA Update and Delete crash: no TLSA branch in `_records_are_same`, the generic branch evaluates `TlsaValue == str` and octoDNS's equality mixin raises `AttributeError`. | [__init__.py:714-721](octodns_cloudns/__init__.py#L714-L721) |
| W4 | critical | Value removal is silently skipped for NS, MX, SRV, SSHFP, NAPTR, LOC and any TXT containing `;` or ending in `.`: `record['record'] in to_delete` compares a raw API string with octoDNS value objects or dotted names. A root NS change `[a,b] -> [c,d]` adds c and d and leaves a and b live. Non-convergent: the next run plans the same Update and again deletes nothing. | [__init__.py:599-601](octodns_cloudns/__init__.py#L599-L601) |
| W5 | high | `stripped_change.new.values = to_create` assigns a `set` onto `change.new`, which *is* the record inside `plan.desired` (`Zone.copy()` is shallow). A second target for the same zone in the same run sees a truncated desired record; `record.data` raises `TypeError` afterwards. | [__init__.py:603-604](octodns_cloudns/__init__.py#L603-L604) |
| W6 | high | CAA and LOC matching compares octoDNS ints/floats with the API's string fields without casting, so those records are never matched: Delete no-ops, Update creates duplicates. CAA and TLSA values only became hashable in octoDNS 1.22.0, which makes PR #22's `except TypeError` branch dead code on 1.22+. | [__init__.py:643-678](octodns_cloudns/__init__.py#L643-L678) |
| W7 | high | Parameters are string-formatted into a GET URL with no encoding. `&` in a TXT, CAA or NAPTR value splits the request, `#` truncates it, valid `%XX` sequences are decoded server-side. ClouDNS accepts the mangled request and answers Success. | [__init__.py:75-146](octodns_cloudns/__init__.py#L75-L146) |
| W8 | medium | `_data_for_TXT` does `rstrip('.')`, so a TXT value ending in `.` never round-trips: perpetual Update plus one duplicate API record per `--doit` run. | [__init__.py:290](octodns_cloudns/__init__.py#L290) |
| W9 | medium | Geo create sends the default values for every location (never `record.geo[code].values`), reduces `NA-US-CA` to `CA`, and accumulates `geodns-code` parameters across the loop. | [__init__.py:148-152](octodns_cloudns/__init__.py#L148-L152), [437-461](octodns_cloudns/__init__.py#L437-L461) |
| W10 | medium | Delete-before-create inside one Update leaves a window with zero records and no rollback if the add fails (for example on `Invalid TTL`). | [__init__.py:591-605](octodns_cloudns/__init__.py#L591-L605) |
| W11 | medium | NAPTR matching ignores service/regexp/replacement, so sibling rows are returned twice and `delete-record` is called twice for the same id; the second call fails and aborts the apply. | [__init__.py:615-623](octodns_cloudns/__init__.py#L615-L623) |
| W12 | low | `_apply` calls `dns/get-zone-info` on every apply although `plan.exists` is already known; `isGeoDNS` can never return True at its call site, and its string does not match the documented register error. | [__init__.py:739-748](octodns_cloudns/__init__.py#L739-L748) |
| W13 | low | `_zone_records` is never updated inside an apply and is kept after a failed apply (`pop` is not in a `finally`). Harmless today, a hazard once ids drive `mod-record`. | [__init__.py:374-380](octodns_cloudns/__init__.py#L374-L380), [755](octodns_cloudns/__init__.py#L755) |
| W14 | info | Every `_params_for_*` value-formatting branch is dead: `_apply_create` overwrites `rrset_values` with the raw octoDNS value object and `record_create` depends on that. | [__init__.py:563-575](octodns_cloudns/__init__.py#L563-L575) |

### 2.2 Read path

| id | severity | symptom | where |
|---|---|---|---|
| R1 | critical | `zone_records` swallows every `ClouDNSClientException` (invalid credentials, rate limit, HTTP 4xx) and `populate` returns `exists=False` with zero records. Target mode: the dry run shows "Create zone + create everything". Source mode: every other target plans a full delete. Only `Missing domain-name` means "zone does not exist". | [__init__.py:374-380](octodns_cloudns/__init__.py#L374-L380), [418](octodns_cloudns/__init__.py#L418) |
| R2 | high | GeoDNS rows are flattened into plain values while `SUPPORTS_GEO=True`, so every geo record is an Update on every run that then applies with zero calls. | [__init__.py:396-417](octodns_cloudns/__init__.py#L396-L417) |
| R3 | high | SSHFP and LOC parameter spellings differ between read (`fp_type`, `lat_deg`, `h_precision`) and write (`fptype`, `lat-deg`, `h-precision`); the wiki documents the underscore form, both ClouDNS SDKs send the dashed form. Needs probe P1. | [__init__.py:169](octodns_cloudns/__init__.py#L169), [207-208](octodns_cloudns/__init__.py#L207-L208), [349-357](octodns_cloudns/__init__.py#L349-L357) |
| R4 | high | Credentials travel in the URL query string, are duplicated in a meaningless `Authorization: Bearer` header, and the full URL is logged at DEBUG. `logging.basicConfig(level=DEBUG)` runs at import time. | [__init__.py:11](octodns_cloudns/__init__.py#L11), [57-62](octodns_cloudns/__init__.py#L57-L62), [75-76](octodns_cloudns/__init__.py#L75-L76), [91](octodns_cloudns/__init__.py#L91) |
| R5 | medium | An empty zone comes back as the JSON array `[]` (confirmed by go-acme/lego and libdns source comments); `records_data.items()` then raises. This is exactly the state right after `_apply` auto-creates a zone. | [__init__.py:397-399](octodns_cloudns/__init__.py#L397-L399) |
| R6 | medium | No timeout, no retry, GET for mutating calls, HTTP call made while holding the throttle lock. | [__init__.py:89-107](octodns_cloudns/__init__.py#L89-L107) |
| R7 | medium | Inactive records (`status` 0) are read as live values; the provider can neither detect nor fix them. | [__init__.py:399-405](octodns_cloudns/__init__.py#L399-L405) |
| R8 | medium | Per-record TTLs are collapsed to `records[0]['ttl']` (SRV and LOC even use the leaked loop variable, so the *last* row). Drift left by a partial apply is invisible. | [__init__.py:280-372](octodns_cloudns/__init__.py#L280-L372) |
| R9 | low | The throttle honours only the 20 req/s tier; ClouDNS also enforces 600 req/min and 36,000 req/h per IP, and answers a throttle with HTTP 200 `{"status":"Failed","statusDescription":"Request blocked. <ip> is sending more than 20 requests per second."}`, which the client turns into a fatal exception mid-apply. | [__init__.py:52-55](octodns_cloudns/__init__.py#L52-L55), [93-104](octodns_cloudns/__init__.py#L93-L104) |
| R10 | low | `populate` logs `found -N records` (wrong `before` computation). | [__init__.py:406](octodns_cloudns/__init__.py#L406) |

### 2.3 Tests, packaging, security

| id | severity | finding |
|---|---|---|
| S1 | critical | A ClouDNS sub-user id (85055) and password are committed in [test_octodns_cloudns_provider.py:471-478](tests/test_octodns_cloudns_provider.py#L471-L478) since commit 0a7245c (PR #20, 2026-02-15). Upstream `main` carries the password as the code default for `CLOUDNS_AUTH_PASSWORD`. Both repositories are public. The tests create and delete records in the real zone `argl.net`. The sub-user and zone belong to the PR #20 author's account, so this fork's maintainer may not be able to rotate the password. |
| S2 | high | Coverage is 44 %. `populate`'s parsing loop, every `_data_for_*`, all of `_records_are_same`, `_apply_delete` and the diff branch of `_apply_update` have zero coverage. Three tests mock the code under test; the only `_apply_update` test mocks `_records_are_same` and thereby hides W6. `script/coverage` demands 100 % and would fail. |
| S3 | high | CI runs bare `pytest` on Python 3.8/3.10/3.12. octoDNS 1.22 needs Python 3.10+, so the 3.8 leg silently tests octoDNS 1.10. `script/cibuild` (pyflakes, black, isort, coverage) is never run and fails today on all four checks. |
| S4 | medium | `setup.py` says `octodns>=0.9.17`, `python_requires>=3.6`, `license='MIT'` (the LICENSE file is GPL-3.0) and `url=https://github.com/octodns/octodns-cloudns` (404, also on PyPI). `requirements.txt` pins `octodns==1.3.0`. No `pyproject.toml`, no `CHANGELOG.md`, no `.gitignore`, dangling pre-commit symlink, `script/format` ignores `VENV_NAME`, `cibuild-setup-py` uses `setup.py install`. |
| S5 | low | [tests/config.yaml](tests/config.yaml) and [tests/config/recordsfortest.bg.yaml](tests/config/recordsfortest.bg.yaml) are unused, and the former references the non-existent class path `octodns.provider.cloudns.ClouDNSProvider` (ClouDNS's own wiki article 509 shows the same wrong path). |

## 3. ClouDNS API contract the rewrite codes against

Sources: wiki articles 41, 42, 43, 48, 57, 58, 59, 60, 66, 134, 153, 183, 184, 188, 434 (raw HTML),
ClouDNS's official [PHP SDK](https://github.com/ClouDNS/cloudns-php-sdk) and
[Go client](https://github.com/ClouDNS/cloudns-go) (used by their Terraform provider), plus
go-acme/lego, dnscontrol and libdns for observed behaviour.

### 3.1 Endpoints and envelope

- All endpoints accept GET or POST; both ClouDNS SDKs POST (PHP: form body, Go: JSON body). The
  rewrite POSTs a form body via `requests` `data=`, which also URL-encodes every value.
- Auth is `auth-id` | `sub-auth-id` | `sub-auth-user` plus `auth-password` as ordinary parameters.
  There is no header auth. API users may be IP-restricted at creation (article 42).
- The HTTP API is a paid-plan feature, and TTLs can only be changed on Premium, DDoS-protected or
  GeoDNS subscriptions (article 188). On other plans every TTL path in this roadmap is moot.
- Every failure is HTTP 200 with `{"status":"Failed","statusDescription":"..."}`. Known texts:
  `Missing domain-name` (zone does not exist, or malformed name; the same text on articles 57 to 66
  and 134), `Invalid record-id param.`, `Invalid TTL. Choose from the list of the values we support.`,
  `This record type is not supported.`, `Invalid authentication, incorrect auth-id or auth-password.`,
  `Request blocked. <ip> is sending more than 20 requests per second.` (rate limit, observed by
  dnscontrol), and for `dns/register`: `<domain> has been already added.`,
  `<domain> is invalid domain name.`, `Your plan does not support GeoDNS zones.`
- `dns/records` returns an object keyed by record id (string). Scalars are strings (`id`, `ttl`,
  `priority`, `weight`, `port`, `failover`, `caa_flag`, ...) except `status` (int). An empty zone
  returns `[]`. The apex host is `""`. Hostname targets come back without a trailing dot. ClouDNS's
  default apex NS rows are ordinary rows with ids.
- `dns/add-record` returns `{"status":"Success","statusDescription":"The record was added successfully.","data":{"id":N}}`.
- `dns/mod-record` requires `domain-name`, `record-id` (the wiki table says `id`; every example, the
  error text and both SDKs use `record-id`), `host`, `ttl`, plus the same type-specific parameters as
  add-record. It cannot change the record type and has no `status` parameter. Whether omitted
  parameters are preserved is undocumented; every production client resends the full record, so
  will we.
- `dns/delete-record` takes `domain-name`, `record-id`. `dns/change-record-status` takes
  `domain-name`, `record-id`, `status` (1/0; omitting it toggles).
- `dns/register` takes `domain-name`, `zone-type` (`master|slave|parked|geodns`) and optional `ns[]`.
  Without `ns[]` ClouDNS seeds its default NS rows at the apex; with `ns[]` "default NS records will
  be not added" (article 48).
- Rate limits per IP: 20 req/s, 600 req/min, 36,000 req/h (article 41). Sustained rate is therefore
  10 req/s.
- Allowed TTLs per the wiki: 60, 300, 900, 1800, 3600, 21600, 43200, 86400, 172800, 259200, 604800,
  1209600, 2592000. `dns/get-available-ttl` returns the account's actual list (some accounts have
  600; dnscontrol documents 2419200 instead of 2592000), so the list must be fetched, not assumed.

### 3.2 Per-type parameter map (add and mod are identical)

| type | write params | list fields read | canonical key (value only, no ttl) |
|---|---|---|---|
| A, AAAA | `record` | `record` | normalised IP |
| NS, PTR | `record` (no trailing dot) | `record` (+ `.`) | fqdn with dot |
| CNAME, ALIAS, DNAME | `record` (no trailing dot) | `record` (+ `.`) | fqdn with dot |
| TXT, SPF | `record` = `value.to_raw_text()` | `record` -> `TxtValue.normalize_raw_text()` | raw text |
| MX | `record` (exchange, no dot), `priority` | `record`, `priority` | `pref exchange.` |
| SRV | `record` (target), `priority`, `weight`, `port` | same | `prio weight port target.` |
| CAA | `caa_flag`, `caa_type`, `caa_value` | same (strings) | `flags tag value` |
| SSHFP | `record` (fingerprint), `algorithm`, `fptype` **or** `fp_type` (P1) | `fp_type` **or** `fptype` (P1) | `alg fptype fp` |
| TLSA | `record` (cert data), `tlsa_usage`, `tlsa_selector`, `tlsa_matching_type` | same | `usage sel match data` |
| NAPTR | `order`, `pref`, `flag`, `params`, `regexp`, `replace` | same | all six fields |
| LOC | `lat_deg`... `v_precision` **or** `lat-deg`... `h-precision` (P1) | `lat_deg`... **or** `lat-deg`... (P1) | all twelve fields |

Attribute mapping between octoDNS value classes and ClouDNS: `preference -> priority` (MX),
`fingerprint_type -> fptype/fp_type`, `preference -> pref`, `flags -> flag`, `service -> params`,
`replacement -> replace` (NAPTR), `certificate_association_data -> record` (TLSA),
`lat_degrees -> lat_deg` and so on (LOC). Both SDKs send every parameter on every call, so the API
tolerates extraneous parameters; that makes "send both spellings" a viable fallback if P1 is
inconclusive.

## 4. octoDNS contract constraints (octodns 1.22.0, verified from source)

- `change.new` **is** the record object in `plan.desired`; `change.existing` **is** the record that
  `populate` built. Never assign to their attributes. Use `record.copy()` or local structures.
- An `Update` is emitted when values differ, or TTL differs, or (with `SUPPORTS_GEO`) geo differs.
  Record identity is `(name, type)` only, so a type change is always Delete + Create and
  `mod-record`'s inability to change type is never hit by an Update.
- `Plan.changes` is sorted Delete < Create < Update. `_apply` may reorder; keep that order so
  CNAME <-> A swaps delete first.
- `BaseProvider.apply()` returns `len(plan.changes)` regardless of what `_apply` did, and the manager
  sums it into "N total changes". The in-contract way to be loud is to raise `ProviderException`
  from `_apply` (goal 3). Overriding `apply()` is possible but non-standard; the "count real calls"
  idea from the bug report belongs in an octoDNS core discussion.
- `populate(zone, target, lenient)` must return `True` if the zone exists (even with zero records)
  and `False` only if it does not. `plan.exists` drives the "Create zone" output and the root-NS
  safety check in `Plan.raise_if_unsafe()`.
- Provider limitations belong in `_process_desired_zone` (copy the record, adjust,
  `add_record(replace=True)`, `supports_warn_or_except`), `_include_change` and `_extra_changes`.
  `strict_supports` defaults to True, so unsupported things raise at plan time unless the user opts
  out. `Zone.get(name, type)` exists for lookups.
- TXT round trip: `TxtValue.normalize_raw_text(raw)` when reading, `value.to_raw_text()` when
  writing. Never strip dots.
- `SUPPORTS_GEO` is a mandatory class attribute; geo records are deprecated (removal in 2.0).
- Every structured value class is hashable in 1.22 and exposes typed attributes (ints/floats), so
  API strings must be cast before comparison. The reconcile below keys on strings, so it does not
  itself need that hashability; only the TXT helpers need 1.22.
- The manager creates one provider instance per config entry and shares it across zone threads when
  `max_workers > 1` (default 1). Per-instance caches, the `requests.Session` and the rate limiter are
  therefore shared state and need a lock or per-zone keying.

## 5. Target design

Three independent designs were produced and scored by three reviewers with different priorities
(octoDNS contract compliance, production SRE, long-term maintainer). The modular design below won
two of three verdicts and was second by one point in the third; the grafts from the other two are
folded in. All three agreed on the core: one codec table, a per-record-id reconcile with
create/delete pairing, adds before mods before deletes, and a hard failure on a zero-operation
Update. It follows octodns-cloudflare's `_apply_Update`
([octodns_cloudflare/__init__.py:806-934](../octodns-cloudflare/octodns_cloudflare/__init__.py#L806-L934)),
the only surveyed provider whose backend has the same shape as ClouDNS (one API record per value,
add/modify/delete by id, no rrset or batch endpoint). OVH and Constellix use delete+recreate and are
not a model; GCore, NS1 and Route53 have rrset semantics ClouDNS lacks.

### 5.1 Module layout

- `octodns_cloudns/client.py`: `ClouDNSClient` with `zone_info`, `zone_create(domain, zone_type, ns=None)`,
  `records`, `record_add`, `record_mod`, `record_delete`, `record_set_status`, `available_ttls`,
  `records_count`. One `request(endpoint, **params)` that POSTs a form body (auth included), has a
  timeout, never logs credentials (DEBUG logs endpoint plus a redacted parameter dict), and maps
  `statusDescription` to typed exceptions through one ordered table: `ClouDNSMissingDomain`
  (`Missing domain-name` and get-record's `Missing domain name.`), `ClouDNSZoneExists`
  (`has been already added.`), `ClouDNSAuthError`, `ClouDNSRateLimited` (`Request blocked.`),
  `ClouDNSInvalidTtl`, `ClouDNSInvalidRecordId`, `ClouDNSPlanLimit` (`Your plan does not support GeoDNS zones.`),
  and `ClouDNSClientException` for everything else. The existing HTTP-status subclasses stay.
  Retries: `Request blocked.` on every endpoint (the request was not processed), transport errors
  and 5xx only on reads and idempotent writes (`mod-record`, `change-record-status`), never on
  `add-record` or `delete-record`.
  Rate limiter: a fixed minimum interval (default 10 req/s, configurable) shared through a
  module-level registry keyed by `(auth_type, auth_id)` so two provider entries for one account share
  a budget, sleeping outside the lock, with a `penalise()` hook called after a throttle reply. A
  sliding-window limiter that allows 18 req/s bursts under the per-minute cap is an optional later
  refinement; the fixed interval satisfies all three documented caps in ten lines.
- `octodns_cloudns/codec.py`: one table `TYPES` with, per record type, `to_fields(value) -> send params`,
  `from_row(row) -> octoDNS value data`, and `key(value)`. The key is derived from the same dict
  that becomes the payload (`tuple(sorted(to_fields(value).items()))`), so key and payload cannot
  drift; both sides first pass through the octoDNS value class (row -> value data -> `Record.new`
  value -> `to_fields`), which gives identical normalisation of case, idna, ints and floats. A
  `FIELD_ALIASES` table holds the read-side alternatives (`fp_type`|`fptype`, `lat_deg`|`lat-deg`,
  and the three observed GeoDNS location spellings) and a separate send column, so P1's answer is a
  one-row change. A frozen `Row` dataclass carries `id, type, host, ttl, status, location, key, raw`.
  `populate`, the reconcile and the tests all go through this table; that retires
  `_records_are_same`, `_params_for_*` and `_data_for_*` (fixes W3, W4, W6, W8, W11, R3, W14).
- `octodns_cloudns/reconcile.py`: a pure function `reconcile(rows, wants) -> [Op]` with
  `Op = Add | Mod | Activate | Delete`, no I/O and no octoDNS imports, so every scenario is an exact
  ordered-op table test.
- `octodns_cloudns/provider.py`: `ClouDNSProvider` with `populate`, `_process_desired_zone`,
  `_extra_changes`, `_apply` and thin `_apply_Create/Update/Delete` that call `reconcile` and execute
  ops through the client with a write-through cache.
- `octodns_cloudns/__init__.py`: re-exports `ClouDNSProvider`, `ClouDNSClient`, the exception
  classes and `__version__ = __VERSION__`, so the documented class path and existing imports keep
  working. No `logging.basicConfig`.

Public constructor stays `ClouDNSProvider(id, auth_id, auth_password, sub_auth=False)`. New
keyword-only options with backwards-compatible defaults: `zone_type` (`master` default, `geodns`),
`ttl_policy` (`strict` default: unsupported TTL raises at plan time; `round_up`), `validate_ttl`
(`api` default: fetch `dns/get-available-ttl` once per zone, fall back to the documented list on
error; `static`), `inactive_records` (`reactivate` default | `ignore`), `timeout`, `calls_per_second`.

### 5.2 Populate keeps the ids

`zone_rows(zone)` calls `records()` (which coerces `[]` to `{}`), returns `None` on
`ClouDNSMissingDomain`, and lets every other exception propagate (fixes R1, R5). It parses each row
through the codec into `Row` objects stored in `self._zone_rows[zone.name]`, skipping unsupported
types with one INFO line per zone. `populate` groups rows by `(host, type)`, excludes `status` 0 rows
from the octoDNS values but keeps them in the side table, dedupes identical keys (lowest id wins,
WARNING), reports `min(ttl)` for mixed-TTL groups (WARNING listing the ids), warns once per zone
about rows with `failover == "1"`, and builds records with `Record.new(..., lenient=lenient)`.
`exists` is `True` whenever the listing succeeded (an empty zone included), `False` only on
`ClouDNSMissingDomain`. `before = len(zone.records)` is captured before the loop (R10).

### 5.3 Reconcile

```
rows  = side-table rows for (host, type)           # active, inactive, unparsable, geo
wants = one Want(key, location, ttl, fields) per desired value (from change.new, read-only)

existing = {(location, key): row}   lowest id wins; duplicates and unparsable rows -> junk
desired  = {(location, key): want}

adds  = wants whose (location, key) is not in existing
mods  = matched rows whose ttl != want.ttl            # the TTL-only case: one Mod per row
acts  = matched rows with status 0                    # reactivate instead of adding a duplicate
left  = existing rows whose (location, key) is not desired

pair adds with left ONLY inside the same location bucket -> Mod on the leftover's id
   (single-value types and the root NS set never pass through an empty state; a value swap is one call)

ops = Adds -> Mods -> Activates (after the Mod for paired rows) -> Deletes(left) -> Deletes(junk)
```

`_apply_Update` raises if `ops` is empty; to avoid aborting a half-applied plan on one false
positive, zero-op changes are collected and the `ProviderException` naming them is raised after the
loop (goal 3). `_apply_Create` runs the same function (normally all Adds; matched stray rows are
kept, never deleted then re-added). `_apply_Delete` deletes every row of that host/type, inactive
and junk included, sorted by id. `_apply` sorts changes Delete -> Create -> Update, creates the zone
from `plan.exists` (no extra `get-zone-info`, fixes W12), and clears the cache in a `finally`
(fixes W13). New ids from `record_add` are inserted into the side table; a missing `data.id` marks
the cache dirty instead of failing. The empty-cache check is `is None`, not falsiness, so an empty
zone is not re-listed.

Call counts (today's behaviour in parentheses):

| scenario | calls |
|---|---|
| TTL-only change on a 3-value A record | 3 mod (0, silent) |
| one value swapped in a 3-value TXT | 1 mod (1 delete + 1 add, or 1 add with the old row left live) |
| CNAME target change | 1 mod (crash) |
| add one CAA value | 1 add (never matched: duplicates) |
| replace all 20 values of a TXT | 20 mod (20 delete + 20 add with an outage window) |
| root NS `[a,b] -> [c,d]` | 2 mod (2 add, a and b left live) |
| desired value present only as an inactive row | 1 activate (+1 mod if ttl differs) |

### 5.4 Zone auto-create

When `plan.exists` is False: if the desired zone has a root NS record, pass its values as `ns[]` to
`dns/register` so ClouDNS seeds no default NS rows; otherwise register without `ns[]` and re-list
the zone before applying, so the seeded default NS rows are in the side table and octoDNS's root
NS Create reconciles against them instead of adding beside them. Classify `has been already added.`
as `ClouDNSZoneExists` (treat as exists and re-list) and `Your plan does not support GeoDNS zones.`
as a hard error naming the `zone_type` option.

### 5.5 TTL policy

In `_process_desired_zone`: obtain the allowed list (`dns/get-available-ttl` once per zone, fallback
to the documented list). For every record whose TTL is not allowed call
`supports_warn_or_except(msg, 'rounding up to <next allowed>')`, then
`desired.add_record(copy_with_new_ttl, replace=True)`. With the default `strict_supports: true` this
fails at plan time with a clear message instead of mid-apply after deletes; with
`strict_supports: false` the plan shows the TTL that will actually be written. `_extra_changes`
emits an Update for any rrset whose rows have non-uniform TTLs, inactive rows (unless
`inactive_records: ignore`), duplicates or unparsable rows, with an INFO line `drift at <name> <type>`
listing the row ids, because the plan line itself shows no visible difference. Those repair Updates
count toward `Plan.raise_if_unsafe()` thresholds; document that a first run on a drifted zone may
need `--force`.

### 5.6 Error handling and loudness

- Any `Failed` status on a write raises with the verbatim `statusDescription`; only `Request blocked.`
  and transport errors on idempotent endpoints are retried, with backoff.
- Zero-op Updates fail the sync as described in 5.3.
- `ClouDNSInvalidTtl` names the record, the TTL and the allowed list.
- Rows with `failover == "1"` get a WARNING before any write; a paired Mod that would move such a row
  to a different value is logged explicitly (behaviour of mod-record on monitored rows is
  undocumented, probe P9).
- Thread safety: side tables keyed per zone, cache writes under a lock, limiter shared per account,
  one `requests.Session` per thread or guarded.

### 5.7 GeoDNS

Decision D2 for the maintainers; recommendation: 0.1.0 ships `SUPPORTS_GEO = False`. octoDNS
deprecates geo records, the current implementation can neither read nor correctly write them (W9,
R2), and reading requires the location vocabulary from `dns/get-geodns-locations` plus a mapping
from octoDNS continent/country/subdivision codes that does not exist today (P7). The codec key and
`Row` already carry the location, and pairing is bucketed by location, so geo support can return in
a later release (as legacy `geo` or `dynamic` records with geo-only rules) without touching the
reconcile. If geo must stay in 0.1.0, add the read-side mapping, validate every desired code at plan
time via `supports_warn_or_except`, refuse cross-bucket pairing (clearing a location through
`mod-record` is undocumented), and skip rrsets that have only geo rows with a WARNING rather than
inventing default values.

## 6. Phased plan

Each phase is one or a few PRs, leaves the test suite green, and is releasable on its own. Sizes
are relative (S < M < L). Release milestones mean "merged upstream and released by ClouDNS" or
"installable from a fork tag via `pip install git+https://github.com/istr/octodns_cloudns@vX`":
the fork has no PyPI rights for `octodns-cloudns`, and `script/release` signs tags with GPG and
pushes to `origin`.

| item | owner |
|---|---|
| PyPI releases and version bumps | ClouDNS (boyanpeychev) |
| rotating sub-user 85055 | PR #20 author (constantins2001) or ClouDNS support |
| license decision (MIT metadata vs GPL-3.0 file) | upstream |
| everything else | this fork, submitted as PRs |

### Phase 0: stop the bleeding (S, immediately, independent of everything else)

1. Ask the PR #20 author and ClouDNS support to rotate or delete API sub-user 85055; if this fork's
   maintainer has access, do it directly. History rewriting cannot un-leak a value public since
   February 2026; treat it as burned regardless.
2. Remove the credential, the `85055` default and `argl.net` from the test module; require
   `CLOUDNS_INTEGRATION_TEST`, `CLOUDNS_SUB_AUTH_ID` or `CLOUDNS_AUTH_ID`, `CLOUDNS_AUTH_PASSWORD` and
   `CLOUDNS_TEST_DOMAIN`, skipping otherwise.
3. Open an issue on ClouDNS/octodns_cloudns that their `main` carries the password as a code default.
4. Add `.gitignore` (from octodns-cloudflare) and a secret scanner (gitleaks or trufflehog) to CI and
   the pre-commit hook.

Done when: no credential string in the working tree, CI has a secret-scan step, upstream notified.

### Phase 1: safety net, tooling, live probes (M)

1. Packaging: `install_requires` `octodns>=1.22.0`, `python_requires>=3.10`, fix `url`, add
   `pyproject.toml` with the octodns-org black/isort/pytest configuration, regenerate
   `requirements*.txt` with `script/update-requirements`, add `CHANGELOG.md`, ship
   `.git_hooks_pre-commit`, make `script/format` honour `VENV_NAME`, replace `setup.py install` in
   `script/cibuild-setup-py` with `pip install .`, drop the deprecated `tests_require`. Leave the
   license line for upstream to decide (D1) but record the discrepancy in the CHANGELOG.
2. One formatting-only commit (`script/format`) listed in `.git-blame-ignore-revs`; fix the four
   pyflakes findings; remove the two never-raised exception classes and the unused imports.
3. Replace `.github/workflows/ci.yml` with the octodns-cloudflare pattern: read `.ci-config.json`
   from octodns core (currently 3.10 to 3.14), run `script/cibuild` and `script/cibuild-setup-py`.
   Keep the coverage gate off until phase 3.
4. Write `script/probe-api` and run probes P1 to P6 and P8 to P12 (section 7) against a throwaway
   zone. Prerequisites: a paid ClouDNS plan with HTTP API, an API user without IP restriction (or a
   self-hosted runner), a subscription on which TTLs can be changed, and, for a sub-user, permission
   on the test zone. The script may register its own zone (`octodns-it-<uuid>.test`) and must
   delete everything it created. Commit every raw response as `tests/fixtures/cloudns-*.json`; this
   is the single most important input for phases 2 and 3.
5. Rename the test module to `tests/test_octodns_provider_cloudns.py`, add
   `tests/config/unit.tests.yaml` with one record of every supported type, delete the three
   tautological tests and the octodns-core-only test, delete [tests/config.yaml](tests/config.yaml)
   and [tests/config/recordsfortest.bg.yaml](tests/config/recordsfortest.bg.yaml) (or replace them
   with a working Manager config for the end-to-end test in phase 3).
6. Characterisation tests for W1 to W8 and R1, R5 using the captured fixtures: drive
   `provider._apply(Plan(zone, zone, [Update(existing, new)], True))` with a mocked client and assert
   the exact call list. Mark the ones that assert phase 2/3 behaviour `xfail`.

Done when: CI runs lint, format check and tests on the octodns-org Python matrix; fixtures captured
from the live API exist for every supported type; the probe results are recorded in the CHANGELOG.

### Phase 2: the bug fix as a small, reviewable release (M, 0.0.18)

This phase makes TTL changes work before the modular rewrite lands, using `mod-record` and nothing
else. It does not route anything through delete+recreate.

1. `ClouDNSClient.record_mod(domain, record_id, host, ttl, fields)` wrapping `dns/mod-record` with
   `record-id`, always sending the full row, and `ClouDNSClient.request` switched to a POST form
   body with a timeout (this also fixes W7 and R4 for every call).
2. In the existing `_apply_update`: normalise single-value records (`values = getattr(rec, 'values', None) or [rec.value]`,
   fixes W2); for every API row matched by `_records_are_same` whose `ttl` differs from `change.new.ttl`,
   issue one `record_mod` that echoes the row's own type fields back with the new TTL (for SSHFP and
   LOC send the spelling P1 established, or both); never assign to `change.new` (W5); raise
   `ProviderException` when an Update produced zero API calls (goal 3, defence in depth).
3. `zone_records`: coerce `[]` to `{}` (R5); propagate everything except `Missing domain-name` (R1).
4. Drop `logging.basicConfig`, the Bearer header and URL logging (R4).
5. Regression tests: TTL-only on A, MX, root NS and CNAME each issue exactly one `mod-record` per
   row and no add/delete; auth failure propagates from `populate`; `[]` populates zero records with
   `exists=True`; `change.new.values` is untouched after apply.

Known limits of 0.0.18, documented in the CHANGELOG: value removals for structured types (W4) and
CAA/LOC/TLSA matching (W3, W6) are only fixed by phase 3; the old value-diff path still runs for
value changes.

Done when: the four TTL-only regression tests are green against a mocked client and once against
the live throwaway zone; PR submitted upstream; fork tag `v0.0.18`.

### Phase 3: the rewrite (L, 0.1.0)

Land as pure additions first, then switch over, so upstream can review small diffs.

1. `client.py` as in 5.1, with tests using `requests_mock` asserting the POST body (form encoded,
   credentials present in the body and absent from the URL and from every log record, `record-id`
   used, `include-notes=0`), the shared limiter (injected clock), retry on `Request blocked.`, the
   typed exception table, and idempotency-aware transport retries.
2. `codec.py` with a table-driven test: for every supported type, value -> `to_fields` -> fixture-shaped
   row -> `from_row` -> `Record.new` round-trips the `unit.tests.yaml` record, `key()` is equal on
   both sides, alias tolerance works, unparsable rows yield `key=None`, TXT `\;` escaping and the
   `> 255` byte `" "` separator collapse behave.
3. `reconcile.py` with exact ordered-op tests for the scenario table in 5.3 plus: inactive matched,
   inactive unmatched, duplicates, unparsable, 20 -> 25, 20 -> 15, root NS never empty, no
   cross-bucket pairing, Create with pre-existing matching rows issues zero deletes.
4. `provider.py`: populate on the side table (5.2), `_process_desired_zone` TTL policy and
   `_extra_changes` drift repair (5.5), `_apply` with zone auto-create (5.4), ordering, write-through
   cache, collected zero-op failure (5.3), `SUPPORTS_GEO = False` (5.7) and
   `SUPPORTS_MULTIVALUE_PTR = True` (ClouDNS stores one row per PTR value). Flip the phase 1 `xfail`
   tests to real assertions; add the mutation-guard test with two provider instances sharing one
   plan (W5); add a Manager-level end-to-end test (dry run makes only `dns/records` calls, `--doit`
   sequence matches, second run plans no changes).
5. Delete the old module code and the tests that pinned delete+recreate; turn on
   `--cov-fail-under=100` in CI.
6. Gated integration test class running the scenario table against the throwaway zone with a
   unique label per run and cleanup in `finally`; run it once before tagging.
7. README rewrite: supported types as verified, TTL policy and plan requirement, inactive/duplicate
   semantics ("octoDNS owns every row of a managed name/type"), geo dropped, config keys, the
   correct class path.

Done when: every table in section 2 has a green regression test, coverage 100 %, the integration run
passes, CHANGELOG entry names the behaviour changes (mod-record instead of delete+add, errors no
longer swallowed by populate, TTL validation at plan time, drift repair, geo dropped, floor bump to
octoDNS 1.22 / Python 3.10), PR series submitted upstream, fork tag `v0.1.0`.

### Phase 4: upstream and operations (S)

1. Tracking issue on ClouDNS/octodns_cloudns linking BUG-REPORT.md and upstream issues #2, #3, #4,
   #11, #16 and #18 as symptoms of the same value-string matching design; PRs in order 0, 1, 2, 3.
   Upstream merges quickly with light review, so each PR must be self-contained with its tests. If
   upstream rejects the octoDNS 1.22 floor, the reconcile keys on strings and only the TXT helpers
   need 1.22, so a fallback `replace('\\;', ';')` keeps 0.0.18 installable on older versions.
2. Ask ClouDNS to fix wiki article 509 (wrong class path), the `id` vs `record-id` rows in articles
   59/60, the PyPI home page, and to cut GitHub tags for releases.
3. Run `octodns-compare` (or a `dig` TTL sweep) across all ClouDNS-managed zones in the operating
   repository after the first sync with 0.1.0; `deflect.me` proves drift already exists, and the
   first run will show TTL repairs and inactive/duplicate cleanups as explicit plan lines to review
   before `--doit`.

### Phase 5: follow-ups (not scheduled)

- DS, HTTPS/SVCB, OPENPGPKEY support (upstream issue #14 asks for DNSSEC records); all have hashable
  value classes in octoDNS 1.22 and documented ClouDNS parameters.
- GeoDNS via `dynamic` records with geo-only rules, after P7.
- Pagination guard (`dns/get-records-count` cross-check) only if P4 shows truncation.
- Sliding-window rate limiter if large applies turn out to be limiter-bound.
- Raise the "reported change count vs. real operations" question in octoDNS core.

## 7. Open questions

### 7.1 Live probes (phase 1, `script/probe-api`)

| # | question | why it matters | how to close |
|---|---|---|---|
| P1 | Does add/mod accept `fptype`+`lat-deg` (both SDKs, current code) or `fp_type`+`lat_deg` (wiki)? Which spelling does `dns/records` emit? | SSHFP/LOC create, update and read | Create one SSHFP and one LOC with each spelling, list, delete; record the accepted spelling in `FIELD_ALIASES`. Fallback: send both. |
| P2 | Does `mod-record` with only `ttl` changed and all other fields resent keep everything else intact (status, notes, failover, location)? | TTL-only path | Modify an A record's TTL, re-list, diff all fields. |
| P3 | Is `data.id` in the add-record response a number or a string? | cache update after create | One create. |
| P4 | Does `dns/records` without `rows-per-page` return a zone with more than 100 records completely? | silent truncation | Import 150 records, compare with `dns/get-records-count`. |
| P5 | Are `status=0` records listed, and does `mod-record` reset status? | inactive-record policy | Deactivate via `change-record-status`, list, modify, list. |
| P6 | What do TXT records over 255 bytes look like in the listing? dnscontrol reports literal `" "` chunk separators. | TXT round trip | Create a 300-byte TXT, list. |
| P7 | Which field carries the GeoDNS location (`geodns-location`, `geodns-code`, `geodns-location-code`) and what is the code vocabulary? | only if geo returns | Requires a GeoDNS zone; deferred. |
| P8 | Does add-record reject an identical duplicate row or create it? | Create against stray rows, idempotency | Add the same A twice. |
| P9 | Can the default `ns*.cloudns.net` apex rows be deleted or modified via the API, and does `mod-record` work on a row with a failover check? | root NS management, 5.6 | Register a throwaway zone without `ns[]`, delete one default row; attach a check, modify. |
| P10 | Does `dns/register` with `ns[]` really seed no default NS rows? | 5.4 | Register with `ns[]`, list. |
| P11 | What text does a sub-user get for a zone outside its permissions, and is an unknown extra parameter ignored on add/mod? | R1 classification, P1 fallback | Sub-user restricted to one zone; add with a bogus parameter. |
| P12 | What is returned when the 600/min tier is exceeded, and does ClouDNS expect punycode or Unicode for IDN zone and host names? | limiter, IDNA | 601 `get-zone-info` calls in a minute; one IDN throwaway zone. |

### 7.2 Maintainer decisions

| # | decision | recommendation |
|---|---|---|
| D1 | License: `setup.py` says MIT, the LICENSE file is GPL-3.0. | Upstream decides; this fork only records the discrepancy. |
| D2 | Keep `SUPPORTS_GEO` in 0.1.0? | No (5.7); revisit after P7. |
| D3 | `inactive_records` default: reactivate or ignore? | `reactivate`, with `ignore` documented for teams that disable rows by hand. |
| D4 | `ttl_policy` default with `strict_supports: true`: fail at plan time or round up? | Fail (octoDNS convention); `round_up` opt-in. |
| D5 | Floor bump to octoDNS 1.22 / Python 3.10 in 0.1.0? | Yes; keep the string-key fallback described in phase 4 if upstream objects. |
| D6 | Ship 0.0.18 at all, or go straight to 0.1.0? | Ship it: TTL changes are blocked in production today and the diff is small. |

## 8. Risks and mitigations

| risk | mitigation |
|---|---|
| P1 spelling probe is inconclusive | send both spellings on write (the API tolerates extraneous parameters), read either key. |
| `mod-record` resets a field we did not resend | the reconcile always sends the full record; P2 verifies status, notes, failover and location survive. |
| Account's TTL list differs from the documented one, or TTLs cannot be changed on the plan | `validate_ttl: api` fetches the list per zone; a plan without TTL changes makes every TTL path fail loudly at plan time. |
| Rate limit hit mid-apply on large zones | shared limiter at 10 req/s plus retry with backoff on `Request blocked.`; ordering add -> mod -> delete keeps names populated if a retry ultimately fails; the reconcile is idempotent, so re-running converges. |
| Drift repair deletes or reactivates rows an operator disabled on purpose | `inactive_records: ignore`; INFO lines list the row ids before `--doit`; documented behaviour change. |
| Zero-op failure aborts a sync on a codec false positive | zero-op changes are collected and raised after the loop, so the rest of the plan is applied and the message names the offending records. |
| Behaviour change surprises existing users (errors no longer swallowed, TTL rejection at plan time, geo dropped) | 0.1.0 with a CHANGELOG entry; `strict_supports: false` keeps the old permissive feel for TTLs. |
| Upstream is slow, declines, or rejects the floor bump | small PRs that can be cherry-picked; fork tags installable from git; string-key fallback for older octoDNS. |
| `max_workers > 1` shares one provider instance across threads | per-zone side tables, lock around cache writes, shared limiter, per-thread session. |

## 9. Appendix: evidence trail

- Reproductions (mocked client, octoDNS 1.22.0): TTL-only A/MX/root NS -> `[]` calls; CNAME update ->
  `AttributeError`; MX swap -> `record_create` only, old row kept; `stripped_change` mutation ->
  `desired.values == set()`; two-target run -> second target never creates the shared values.
- Version facts: `ValueMixin` has no `values` on 1.3.0, 1.10.0, 1.21.0, 1.22.0; MX/SRV/SSHFP/NAPTR/LOC
  values hashable since 1.3.0; CAA/TLSA hashable from 1.22.0 (`EqualityTupleMixin.__hash__`,
  CHANGELOG 1.22.0). `Zone.get(name, type)` at zone/base.py:357; `max_workers` default 1 at
  manager.py:211.
- ClouDNS SDK evidence: PHP SDK `dnsModifyRecord` sends `record-id`, `fptype`, `lat-deg`,
  `h-precision` and every parameter on every call; Go client `updaterec` uses `json:"record-id"`,
  `json:"fptype"`, `json:"lat-deg"`, POSTs JSON. Wiki articles 58/60 tables use `fp_type`,
  `lat_deg`, `h_precision`. Article 48: `ns[]` suppresses default NS rows; register error texts.
  Article 188: TTL changes need Premium/DDoS/GeoDNS subscriptions.
- Rate limit text and behaviour: dnscontrol `providers/cloudns/api.go` retries on
  `Request blocked.`; limits from article 41.
- Empty-zone `[]`: go-acme/lego `providers/dns/cloudns/internal/client.go`, libdns/cloudns `client.go`.
- Upstream: ClouDNS/octodns_cloudns, maintainer boyanpeychev, no tags or GitHub releases, PyPI
  `octodns-cloudns` 0.0.17 uploaded 2026-08-21 (author ClouDNS, no maintainer field), open issue
  #14 (DNSSEC), closed issues #2/#3/#4/#11/#16/#18 all symptoms of value-string matching.

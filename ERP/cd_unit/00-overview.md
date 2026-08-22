# cd_unit — API Documentation Overview

## Scope

| | |
| --- | --- |
| **Source files behind `urls.py`** | 6 (`article.py`, `cdunit_mobile.py`, `bundle_mgmt.py`, `bundle_recive.py`, `stationery_items_approval.py`, `qp_list_detailes.py`) |
| **Live routed endpoints** | 42 total |
| **Boilerplate/empty** | `models.py`, `admin.py`, `views.py`, `tests.py` (all Django-default stubs, no app code) — models live in a shared `util` app instead |

Pacing plan: smallest-file-first. Delivered so far: `article.py` (1 endpoint),
`stationery_items_approval.py` (2), `qp_list_detailes.py` (2), `cdunit_mobile.py` (7), and
`bundle_recive.py` (3). Remaining: `bundle_mgmt.py` (27, the large file — will likely need to be
split across more than one turn on its own).

## What this module does

`cd_unit` (app label `cdunit`) is the **CD (Central Distribution) Unit** module of the SLCM
system — it manages the physical bundling and distribution of examination answer sheets:

- **Bundle generation & tracking** (`bundle_mgmt.py`) — creating bundles per exam centre/QP
  code, listing them by date/exam/camp/AO, allocating bundles to evaluation camps, printing
  bundle slips, revaluation bundle handling, and bundle-list reporting.
- **Bundle receipt** (`bundle_recive.py`) — receiving/confirming bundles (including an
  encrypted variant) and fetching bundle data.
- **Article requests** (`article.py`) — colleges requesting "articles" (stationery-adjacent
  request items) against a priority/type master.
- **Stationery approval** (`stationery_items_approval.py`) — approving and dispatching
  stationery item requests raised by colleges.
- **QP (question paper) list details** (`qp_list_detailes.py`) — listing QP-related bundle
  detail views.
- **Mobile API** (`cdunit_mobile.py`) — login/logout, OTP-based password reset/change,
  profile view, and route list for a CD-unit-facing mobile app.

## Response format used across all APIs

Every endpoint in this app returns through a single shared helper, **`format_response(...)`**,
imported via `from util.serializers import *` (its actual implementation lives in the `util`
app and is not part of this upload — signature and body shape below are inferred from call-site
usage across all 6 files, not confirmed against source).

Observed call pattern (positional):

```python
format_response(success: bool, message, data={}, error_code=<optional>, status_code=..., template_name=...)
```

Example calls seen:
```python
format_response(True, FETCH_SUCCESS_MSG, data={'state': y}, status_code=status.HTTP_201_CREATED, template_name='...')
format_response(False, BAD_GATEWAY, {}, BAD_GATEWAY_ERROR_CODE, status_code=status.HTTP_400_BAD_REQUEST, template_name='...')
```

**No deviations found.** A repo-wide search across all 6 source files found zero raw
`Response(...)` calls — every single endpoint in this app goes through `format_response`. This
will be re-checked as the remaining, larger files (`bundle_mgmt.py` in particular) are read in
full, but the grep-level check already covers all 6 files.

## A note on source visibility

This upload contains only the `cd_unit` app itself. The following are imported via `import *`
and are **not included**, so any field list or constant value sourced from them is marked
"external, not confirmable" in the module docs:

- `util.models` — all Django models (`BundleMaster`, `CampMaster`, `ExaminationSchedule`,
  `StationeryRequestItem`, `ArticleRequestCollege`, etc.)
- `util.serializers` — shared serializers, and `format_response` itself
- `util.constants` — all message/status constants (`FETCH_SUCCESS_MSG`, `BAD_GATEWAY`,
  `ACCEPTED`, `DISPATCHED`, etc.)
- `util.pagination` — shared pagination classes

`cd_unit`'s own `serializers.py` (439 lines, 17 classes) **is** included, so fields for any
response built from those classes are confirmed directly from source.

## Sub-areas

| Doc file | Source file | Endpoints | Description | Status |
| --- | --- | --- | --- | --- |
| `article.md` | `article.py` | 1 | College article (stationery-adjacent) requests | ✅ Complete |
| `stationery_items_approval.md` | `stationery_items_approval.py` | 2 | Approve/dispatch stationery item requests | ✅ Complete |
| `qp_list_detailes.md` | `qp_list_detailes.py` | 2 | QP-wise bundle detail listing | ✅ Complete |
| `cdunit_mobile.md` | `cdunit_mobile.py` | 7 | Mobile app auth (login/OTP/password) & profile | ✅ Complete |
| `bundle_recive.md` | `bundle_recive.py` | 3 | Bundle receipt (incl. encrypted variant) & fetch | ✅ Complete |
| `bundle_mgmt.md` | `bundle_mgmt.py` | 27 | Bundle generation, camp allocation, revaluation, reporting | *(coming next)* |

## Known cross-cutting issues

- **No visible per-view `permission_classes`/authentication on any endpoint documented so
  far** (`article.py`, `stationery_items_approval.py`). `stationery_items_approval.py` imports
  `IsAuthenticated` but never applies it. Whether requests are actually gated depends on a
  project-wide `DEFAULT_PERMISSION_CLASSES` setting not visible in this upload — flagged per
  module, not asserted as a confirmed hole.
- **Broad `except Exception`** blocks are the standard error-handling pattern across every
  endpoint seen so far. This means real bugs (`NameError`, `AttributeError`, `UnboundLocalError`
  from missing `None` checks or unassigned variables) get silently converted into a generic
  `BAD_GATEWAY`/`INTERNAL_SERVER_ERROR` response rather than surfacing distinctly — see
  `stationery_items_approval.md` for two concrete instances of this.
- **Copy-pasted duplicate import blocks** — `stationery_items_approval.py` has its entire
  ~25-line import block repeated twice near-verbatim (lines 1–24 and 25–50). Worth checking
  whether the same pattern recurs in the larger files.
- **`logger` undefined in `qp_list_detailes.py`** — all three exception handlers in that file
  call `logger.error(...)`, but the file never imports `logging` or assigns `logger` (unlike
  `article.py` and `cdunit_mobile.py`, which both do). If `logger` isn't supplied by one of that
  file's wildcard imports, every exception in that module triggers a second, uncaught
  `NameError` inside the handler meant to catch the first one — see `qp_list_detailes.md`.
- **`cdunit_mobile.py`: logical failures report `success: true`.** Across `CDLogin`,
  `CDSendOTP`, and `CDResetPassword`, every non-exception failure path (wrong credentials, OTP
  mismatch, unknown email) returns `format_response(True, <failure message>, ...)` — only
  exception-driven failures in that file use `success: false`. Combined with most of that file's
  exception handlers returning HTTP 204 (conventionally "success, no content") for failures, a
  client that trusts the envelope's `success` flag or the HTTP status code over the message text
  would misread several failure conditions as successes. See `cdunit_mobile.md` for the full
  breakdown, including one endpoint (`CDChangePassword`) whose success path is unreachable dead
  code.
- **A confirmed unassigned-variable `NameError`** (the exact class of bug called out in the doc
  prompt) was found in `CDProfileView` (`cdunit_mobile.py`) — variables built only inside an
  `if user.groups...MESSENGER...` branch are referenced unconditionally afterward with no
  `else`. The resulting exception is also reported using a message and hardcoded string both
  copy-pasted from an unrelated password-reset context — see `cdunit_mobile.md`, endpoint 6.
- **`bundle_recive.py` breaks the app-wide "everything is wrapped in a broad `except`" pattern.**
  Its two substantive endpoints (`DecryptedBundleRecive`, `BundleFetch`) have no top-level
  `try`/`except` around the core bundle lookup — an unmatched bundle code/id, an unrecognized
  `bcode` value, or a decryption failure all raise completely unhandled exceptions (a raw 500,
  not a graceful `format_response`) rather than the generic-error pattern seen everywhere else in
  this app so far. `BundleRecive.get()` is also effectively disabled — its real logic is
  commented out, so it always returns an empty success response regardless of input.
- **Near-duplicate endpoints drifting apart with subtle bugs** — `DecryptedBundleRecive` and
  `BundleFetch` share near-identical copy-pasted lookup logic but disagree in the details: one
  has a tuple-instead-of-string typo on its `examination` field, they format the `college` field
  differently, and they derive the exam date from two different models. See `bundle_recive.md`
  for specifics. This is the same "near-duplicate endpoints, subtle differences" bug class the
  doc prompt calls out, and worth watching for again in `bundle_mgmt.py`, which has many more
  structurally similar endpoint pairs (e.g. regular vs. revaluation bundle listing).
- **An unrouted dead class with a real bug**: `bundle_recive.py` also defines `AcceptBundles`,
  which isn't wired up in `urls.py` at all. If it's ever routed, note that it filters
  `BundleMaster.objects.filter(pk=int(i))` without `.first()`/`.get()`, then tries to set
  `.status` and call `.save()` on the resulting `QuerySet` — `QuerySet` has no `.save()` method,
  so every call would raise `AttributeError`.

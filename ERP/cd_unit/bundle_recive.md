# bundle_recive

Source file: `cd_unit/bundle_recive.py` (336 lines). URL mapping is defined in
`cd_unit/urls.py`.

Status: 3 routed endpoints, plus one unrouted dead class (`AcceptBundles`). Unlike every other
file documented so far in this app, the two substantive endpoints here (`DecryptedBundleRecive`,
`BundleFetch`) have **no top-level `try`/`except` around their core bundle-lookup logic** — only
later, cosmetic field lookups are individually wrapped — so several genuine crash risks here are
completely unhandled rather than converted into a generic error response.

## 1. Bundle Receive

| Field | Value |
| --- | --- |
| **URL** | `bundle-recive` |
| **Method** | GET |
| **Auth** | No `permission_classes` set |
| **View class** | `BundleRecive` (extends `ListAPIView`) |

### Request body

None currently read (see Notes — the original implementation that read a `bundlecode` query
param is commented out).

### Response — success (200)

```json
{ "success": true, "message": "<value of FETCH_SUCCESS_MSG>", "data": {} }
```

### Notes

- **This endpoint is effectively disabled.** Roughly the entire method body is commented out —
  the original logic (read `bundlecode` from `request.GET`, look up the matching `BundleMaster`,
  serialize it via `BundleAcknowledgementSerializer`, and return it) is present only as comments.
  What actually runs today is:
  ```python
  gen_state = State.objects.get(label="GENERATED")
  dist_state = State.objects.get(label="DISTRIBUTED")
  return format_response(True, FETCH_SUCCESS_MSG, data={}, status_code=status.HTTP_200_OK, ...)
  ```
  `gen_state`/`dist_state` are fetched and never used for anything — dead code — and the
  response always returns an empty `data: {}` regardless of any request parameters. Whoever
  calls `GET bundle-recive` expecting bundle details back is not currently getting any.
- **No exception handling at all.** If either `State.objects.get(label=...)` lookup fails (no
  matching `State` row), it raises `State.DoesNotExist` completely unhandled — a 500 with no
  graceful `format_response` fallback, unlike almost every other endpoint in this app.

## 2. Decrypted Bundle Receive

| Field | Value |
| --- | --- |
| **URL** | `decrypt-bundle-recive` |
| **Method** | POST |
| **Auth** | No `permission_classes` set |
| **View class** | `DecryptedBundleRecive` (extends `ListAPIView`) |

### Request body

| Field | Type | Notes |
| --- | --- | --- |
| `bundleid` | string | for `bcode="SA"`: an encrypted/encoded payload (see Notes for the slicing scheme); for `bcode="CA"`: a plain `BundleMaster.bundle_code` |
| `bcode` | string | `"SA"` (encrypted lookup by decrypted id) or `"CA"` (plain lookup by bundle code) — any other value leaves the bundle unresolved (see Notes) |

### Response — success (200)

```json
{
  "success": true,
  "message": "<value of BUNDLE_FETCH_SUCCESS_MSG>",
  "data": {
    "bundle_lists": [{
      "bid": 0, "bundle_code": "...", "examination": "...", "examinationcode": "...",
      "programme": "...", "programmecode": "...", "course": "...", "coursecode": "...",
      "qpcode": "...", "college": "Name(Code)", "collegecode": "...",
      "total_answer_sheet": 0, "status": "...", "camp": "...", "subcamp": "...",
      "subcourse": "(...)", "dateofexamination": "dd-mm-yyyy", "district": "..."
    }]
  }
}
```
Hand-built dict, confirmed from source — not a serializer. `Fernet` is used for decryption
(`bcode="SA"` path); it's imported transitively via this file's `from .serializers import *`,
which itself does `from cryptography.fernet import Fernet` — confirmed available, not a missing
import.

### Response — failure

**None.** There is no exception-driven failure response in this method at all — see Notes.

### Notes

- **No top-level `try`/`except` around the core lookup/decryption/status-update logic** — this
  is the most consequential finding in this file. Only the later cosmetic-field lookups (exam
  name, camp, district, etc., starting partway through the method) are individually wrapped in
  small `try`/`except` blocks that fall back to `"NA"`. The following crash paths are all
  **completely unhandled** (no `format_response` fallback, a hard 500 instead):
  - **Unassigned-variable risk, exactly the pattern the doc prompt calls out**: `bundleobj` is
    only assigned inside `if bundlecd == "SA":` or `if bundlecd == "CA":` blocks — there is no
    `else`. If `bcode` is any other value (missing, mistyped, or a value the frontend doesn't
    currently send), `bundleobj` is never assigned, and the next line,
    `if bundleobj.status == gen_state or bundleobj.status == dist_state:`, raises
    `UnboundLocalError: local variable 'bundleobj' referenced before assignment` — unhandled.
  - **Missing `None` check**: both the `"SA"` and `"CA"` branches resolve `bundleobj` via
    `.filter(...).first()` with no check that a row was actually found. An unmatched id/code
    means `bundleobj` is `None`, and the same status-check line above raises `AttributeError` on
    `None` — unhandled.
  - **Decryption failure**: for `bcode="SA"`, `f.decrypt((str(myStr).encode()))` raises
    `cryptography.fernet.InvalidToken` on any malformed, tampered, or expired payload — also
    unhandled, since it's outside every `try` block in the method.
  - A commented-out `except Exception as e: logger.error(...); return format_response(False,
    BAD_GATEWAY, ...)` block sits at the very bottom of the method (after the final `return`),
    showing the original intent was to wrap the whole thing in a try/except — that wrapping is
    not currently active.
- **Fragile fixed-offset slicing on the encrypted payload**: for `bcode="SA"`,
  `myStr = bundleidval[2:len(bundleidval)-1]` strips a hardcoded 2-character prefix and 1-
  character suffix off the incoming string before decrypting — implying the QR/scanned payload
  wraps the encrypted id in some fixed framing not documented in this file. No length check
  before slicing; a `bundleidval` shorter than 3 characters would just produce an empty/odd slice
  fed to `Fernet.decrypt`, most likely surfacing as the unhandled `InvalidToken` above.
- Compare this endpoint's near-duplicate, `BundleFetch` (below) — see "Notes on this source
  file" for the copy-paste/inconsistency pattern between them.

## 3. Bundle Fetch

| Field | Value |
| --- | --- |
| **URL** | `bundle-fetch` |
| **Method** | POST |
| **Auth** | No `permission_classes` set |
| **View class** | `BundleFetch` (extends `ListAPIView`) |

### Request body

| Field | Type | Notes |
| --- | --- | --- |
| `bundlecode` | string | plain `BundleMaster.bundle_code`, no encryption/decoding involved |

### Response — success (200)

```json
{
  "success": true,
  "message": "<value of BUNDLE_FETCH_SUCCESS_MSG>",
  "data": {
    "bundle_lists": [{
      "bid": 0, "bundle_code": "...", "examination": ["..."], "examinationcode": "...",
      "programme": "...", "programmecode": "...", "course": "...", "coursecode": "...",
      "qpcode": "...", "college": "...", "collegecode": "...",
      "total_answer_sheet": 0, "status": "...", "camp": "...", "subcamp": "...",
      "subcourse": "...", "dateofexamination": "dd-mm-yyyy"
    }]
  }
}
```
`"examination"` is shown as an array deliberately — see Notes, this is very likely an
unintentional typo rather than a designed difference from `DecryptedBundleRecive`'s equivalent
field.

### Response — failure

**None** — same as `DecryptedBundleRecive`, there is no top-level exception handling here either.

### Notes

- **Same unhandled crash risks as `DecryptedBundleRecive`**: no top-level `try`/`except`;
  `gen_state`/`dist_state` fetched unhandled; `bundleobj = BundleMaster.objects.filter(bundle_code=bundlecode).first()`
  has no `None` check, so an unmatched `bundlecode` raises `AttributeError` on the very next line
  (`bundleobj.status == gen_state`) with no graceful response.
- **Likely typo — tuple instead of string**: `examname = (examinationcenter.exam.title,)` — note
  the trailing comma inside the parentheses, which makes this a 1-element **tuple**, not the
  plain string value. `DecryptedBundleRecive`'s equivalent line (`examname =
  examscheduleobj.exam.title`, no trailing comma) does this correctly. As written, this
  endpoint's `"examination"` field serializes as `["Some Exam Title"]` instead of a plain string
  — a real response-shape inconsistency between two endpoints that otherwise return
  near-identical bundle detail payloads.
- **Response-shape inconsistency vs. `DecryptedBundleRecive` for `college`**:
  `DecryptedBundleRecive` builds a combined `"Name(Code)"` string (`collegedet`) for its
  `college` field; this endpoint instead puts the raw college name directly into `college`
  (`collegename`) with the code only in the separate `collegecode` field. Two endpoints
  returning what looks like the same bundle-detail shape disagree on how `college` is formatted.
- **Different model used for the same date lookup**: `DecryptedBundleRecive` derives the exam
  date from the already-fetched `ExaminationSchedule.start_date`; this endpoint instead makes a
  separate query against a different model, `ExaminationDate.objects.filter(exam_id=...).first()`.
  Both presumably arrive at "the exam's start date," but via two different models/paths in what
  is otherwise a near-duplicate endpoint — worth confirming this isn't leftover drift from an
  earlier refactor of one endpoint that didn't make it into the other.
- Debug `print(bundlecode)`, `print(bundleobj)`, and `print(subcoursename)` left in.
- If the `examinationcenter` lookup's own inner `try` fails, `examinationcenter` is never
  assigned; the later `examdate = ExaminationDate.objects.filter(exam_id=examinationcenter.exam.id)...`
  line would then reference an undefined name — but this reference sits inside its *own*
  separate `try`/`except` block, so it's caught locally and falls back to
  `dateofexamination = "NA"` rather than crashing — same unassigned-variable shape as elsewhere
  in this app, just harmlessly contained here.

## Notes on this source file

- **`AcceptBundles` is defined in this file but never routed in `cd_unit/urls.py`** — confirmed
  by checking every `path(...)` entry; no URL points at `AcceptBundles`. Since it's unreachable,
  it's not documented as a live endpoint, but it contains a real, confirmable bug worth noting in
  case it's ever wired up: `bundleobj = BundleMaster.objects.filter(pk=int(i))` returns a
  **QuerySet** (missing `.first()`/`.get()`), and the code then does `bundleobj.status = ...`
  followed by `bundleobj.save()` — `QuerySet` objects have no `.save()` method, so this raises
  `AttributeError` on every call (caught by that method's own broad `except`, at least). The
  `approved_by = request.user` line is also assigned and never used.
- **`DecryptedBundleRecive` and `BundleFetch` are near-duplicate endpoints** (encrypted vs. plain
  bundle lookup, otherwise building the same response shape) with clear copy-paste lineage —
  the camp/subcamp/college/subcourse lookup blocks are structurally identical between them — but
  they've drifted apart in small, inconsistent ways (the tuple-vs-string typo, the
  college-field format, and the different date-lookup model) documented individually above. Any
  future fix to one of these bugs should probably be checked against the other endpoint too.
- Neither this file's two substantive endpoints has the app-wide broad-`except`/generic-error
  pattern seen everywhere else — this file is the one place in the app so far where core lookup
  logic can raise a raw, unhandled 500 rather than a `format_response(False, ...)` reply.
- No `permission_classes` set on any of the four classes in this file (including the unrouted
  one).

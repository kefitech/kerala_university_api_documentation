# stationery_items_approval

Source file: `cd_unit/stationery_items_approval.py` (153 lines). URL mapping is defined in
`cd_unit/urls.py`.

Status: 2 routed endpoints. Contains a likely `NameError` on every successful save path in
`StationeryReqList.post`, and a confirmed unassigned-variable (`UnboundLocalError`/`NameError`)
risk in `DispatchedStationeryReqList.post` matching exactly the pattern flagged in the doc
prompt. Both are silently converted into a generic error response by a broad `except Exception`.

## 1. Stationery Approval List

| Field | Value |
| --- | --- |
| **URL** | `stationery-approval` |
| **Method** | GET, POST |
| **Auth** | No `permission_classes` set. `IsAuthenticated` is imported but never applied anywhere in the file. |
| **View class** | `StationeryReqList` (extends `ListAPIView`, overrides both `get` and `post`) |

### Request body (POST)

| Field | Type | Notes |
| --- | --- | --- |
| `id` | int | id of a `StationeryRequestItem` row to approve |

No request body for `GET`.

### Response — success, GET (200)

```json
{
  "success": true,
  "message": "<value of FETCH_SUCCESS_MSG>",
  "data": {
    "item_details_list": [
      {
        "id": 0,
        "request_date": "...",
        "college": "...",
        "collegecode": "...",
        "category": "...",
        "unit": "...",
        "input_quantity": 0,
        "status": "..."
      }
    ]
  }
}
```
Fields above are hand-built from related model attributes in a plain Python loop (not a DRF
serializer) — `request.stationery_request.{request_date, college.name, college.code}`,
`request.stationery_item.{name, unit}`, `request.quantity`, `request.status`. Confirmed
directly from this file's source (not serializer-derived, so no "external" caveat needed for
the shape itself — the underlying models `StationeryRequestItem`/etc. are still external).

### Response — failure, GET (500)

```json
{ "success": false, "message": "<value of INTERNAL_SERVER_ERROR>", "data": {} }
```

### Response — success, POST (201)

```json
{ "success": true, "message": "<value of FETCH_SUCCESS_MSG>", "data": {} }
```

### Response — failure, POST (400)

```json
{ "success": false, "message": "<value of BAD_GATEWAY>", "data": {}, "error_code": "<value of BAD_GATEWAY_ERROR_CODE>" }
```
Returned both for the deliberate "already accepted" check and for any exception (see Notes).

### Notes

- **Likely `NameError` on every successful approval** — `post` does:
  ```python
  stationeryApprovalObj.updated_by = request.user
  stationeryApprovalObj.updated_on = dt.now()
  stationeryApprovalObj.save()
  ```
  `dt` is never imported anywhere in this file (checked directly — only `datetime` and
  `from datetime import datetime` are imported, never `datetime as dt` or any `dt` name). If
  `dt` doesn't come from one of the `import *` wildcards (`util.models`, `util.serializers`,
  `util.constants`, `util.pagination` — none of which are in this upload, so this can't be
  fully ruled out), every call to this endpoint that reaches the save step raises
  `NameError: name 'dt' is not defined`, which is caught by the surrounding
  `except Exception as e:` and returned as the generic 400 `BAD_GATEWAY` response above —
  meaning approvals may currently always be failing at the save step rather than actually
  persisting. Worth confirming against `util`'s wildcard-imported names.
- **Missing `None` check**: `stationeryApprovalObj = StationeryRequestItem.objects.filter(id=request.data.get("id")).first()`
  is immediately followed by `stationeryApprovalObj.status.id == ACCEPTED` with no check that
  the lookup actually found a row. An invalid/missing `id` raises `AttributeError` on `None`,
  again swallowed into the same generic 400 response rather than a distinct "not found" message.

## 2. Dispatched Stationery Request

| Field | Value |
| --- | --- |
| **URL** | `stationery-dispatched` |
| **Method** | POST (see Notes for `GET`) |
| **Auth** | No `permission_classes` set. |
| **View class** | `DispatchedStationeryReqList` (extends `ListAPIView`, overrides only `post`) |

### Request body

| Field | Type | Notes |
| --- | --- | --- |
| `id` | int | id of a `StationeryRequestItem` row to dispatch |

### Response — success (201)

```json
{ "success": true, "message": "<value of FETCH_SUCCESS_MSG>", "data": { "state": true } }
```
`state` reflects whether the object's status actually equals the dispatched state after the
save (see Notes — this can throw before returning).

### Response — failure (400)

```json
{ "success": false, "message": "<value of BAD_GATEWAY>", "data": {}, "error_code": "<value of BAD_GATEWAY_ERROR_CODE>" }
```

### Notes

- **Confirmed unassigned-variable risk (`UnboundLocalError`/`NameError`)** — this is the exact
  pattern the doc prompt calls out explicitly. The method reads:
  ```python
  if stationeryApprovalObj.status.id == DISPATCHED:
      return format_response(False, BAD_GATEWAY, ...)          # early return

  if stationeryApprovalObj.status.id == ACCEPTED:
      dispatched_state = State.objects.get(id=DISPATCHED)        # dispatched_state assigned ONLY here
      stationeryApprovalObj.status = dispatched_state
      stationeryApprovalObj.updated_by = request.user
      stationeryApprovalObj.updated_on = dt.now()

  stationeryApprovalObj.save()                                    # unconditional
  y=False
  if stationeryApprovalObj.status == dispatched_state:            # referenced unconditionally
      y=True
  ```
  `dispatched_state` is assigned only inside the `if ... == ACCEPTED:` branch. If the item's
  current status is neither `DISPATCHED` nor `ACCEPTED` (any other state), that branch is
  skipped entirely, `dispatched_state` is never created, and the unconditional
  `stationeryApprovalObj.status == dispatched_state` check below raises
  `NameError: name 'dispatched_state' is not defined`. This is caught by the same broad
  `except Exception`, so the caller just sees the generic 400 response with no indication of
  why — a request for a stationery item that's still pending (not yet accepted) fails this way
  instead of getting a clear "must be accepted before dispatch" message.
- Same `dt.now()` concern as `StationeryReqList.post` above — `dt` isn't imported in this file.
- Same missing-`None`-check pattern on `stationeryApprovalObj.status.id` as endpoint 1.
- **Dead/inconsistent base class**: this class extends `ListAPIView` but defines only `post` —
  `get` is inherited unchanged from `ListAPIView`, which calls `self.list(request)`. Since
  neither `queryset` nor `get_queryset()` is defined on this class, an actual `GET` request to
  `stationery-dispatched` would raise a DRF `AssertionError` rather than a handled response.
  Not necessarily hit in practice if the frontend only ever POSTs here, but it's an unguarded
  path.
- A commented-out earlier version of a combined `DISPATCHED and ACCEPTED` check is left in
  place (lines 147–148) — copy-paste/iteration residue, consistent with the duplicate top-of-
  file import blocks noted in `00-overview.md`.

## Notes on this source file

- The entire ~25-line import block (django/rest_framework/util imports) is duplicated
  near-verbatim twice at the top of the file (roughly lines 1–24 and 25–50) — clear copy-paste
  evidence, harmless on its own but a sign this file was assembled by pasting from another
  module rather than written fresh.
- `IsAuthenticated` is imported in both duplicated blocks but never referenced anywhere in the
  file — dead import.
- Both endpoints share the identical missing-`None`-check and `dt` concerns, and both rely
  entirely on the outer `except Exception` to turn real bugs into a generic 400 — see
  "Known cross-cutting issues" in `00-overview.md`.

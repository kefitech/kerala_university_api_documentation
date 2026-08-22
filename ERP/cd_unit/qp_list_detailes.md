# qp_list_detailes

Source file: `cd_unit/qp_list_detailes.py` (149 lines). URL mapping is defined in
`cd_unit/urls.py`.

Status: 2 routed URLs (3 HTTP methods total). All three `except` blocks in this file call
`logger.error(...)`, but `logger` is never imported or defined anywhere in the file — a likely
`NameError` inside the error handler itself on every exception path.

## 1. Programme Class List

| Field | Value |
| --- | --- |
| **URL** | `qplistview` |
| **Method** | GET, POST |
| **Auth** | No `permission_classes` set on the view |
| **View class** | `qplistview` (extends `rest_framework.views.APIView`) |

### Request body (POST)

| Field | Type | Notes |
| --- | --- | --- |
| `programme_class_id` | int | id of a `ProgrammeClassMaster` row |

No request body for `GET`.

### Response — success, GET (201)

```json
{
  "success": true,
  "message": "<value of PROGRAMME_CATEGORY_FETCH_SUCCESS_MSG>",
  "data": /* a raw ProgrammeClassMaster QuerySet — see Notes */
}
```

### Response — success, POST (200)

```json
{
  "success": true,
  "message": "<value of PROGRAMME_CATEGORY_FETCH_SUCCESS_MSG>",
  "data": { "sem_list": ["..."], "batch_id_list": ["..."] }
}
```
`sem_list`/`batch_id_list` are plain lists built by hand from `ProgrammeSemester.semester.code`
and `AdmissionYearMaster.admission_year` — not serializer-derived.

### Response — failure, GET/POST (400)

```json
{ "success": false, "message": "<value of BAD_GATEWAY>", "data": {}, "error_code": "<value of BAD_GATEWAY_ERROR_CODE>" }
```

### Notes

- **Response-shape inconsistency — raw QuerySet passed as `data`**: `GET` does
  `data=prog_class_obj` where `prog_class_obj = ProgrammeClassMaster.objects.all()` — a bare
  Django QuerySet of model instances, not run through any serializer or `.values()`/dict
  conversion. Every other endpoint documented so far in this app (`article.py`,
  `stationery_items_approval.py`) passes either serializer `.data` or a hand-built dict/list of
  primitives. Whether `format_response` (external, not confirmable) knows how to JSON-encode raw
  model instances can't be confirmed from this upload, but it stands out as the one place in the
  app that doesn't serialize before responding — worth confirming this endpoint actually returns
  usable JSON in practice.
- **Status-code mismatch**: `GET` (a pure fetch) returns `status.HTTP_201_CREATED` — 201 is
  conventionally reserved for a resource-creation response, not a list fetch.
- `POST` and `GET` share the same view/URL despite doing unrelated things (`GET` lists all
  programme classes; `POST` returns semester/batch lists for one programme class) — not a bug,
  but worth knowing when reading the frontend that calls this URL with different methods.
- No `None` check on `ProgrammeClassMaster.objects.filter(id=prog_class_id).first()` in `POST`,
  but an invalid/missing id just produces an empty `sem_list` (Django's `filter(programme_class=None)`
  returns an empty queryset rather than raising) — silently wrong/empty results rather than a
  crash or validation error.

## 2. QP List Details

| Field | Value |
| --- | --- |
| **URL** | `qpListDetailes` |
| **Method** | POST |
| **Auth** | No `permission_classes` set on the view |
| **View class** | `QpListDetailes` (extends `APIView`) |

### Request body

| Field | Type | Notes |
| --- | --- | --- |
| `program_class` | int | id of a `ProgrammeClassMaster` row |
| `semester` | string | matched against `Semester.code` |
| `year` | string/int | matched against `Examination.year` |

### Response — success (200)

```json
{
  "success": true,
  "message": "<value of PROGRAMME_CATEGORY_FETCH_SUCCESS_MSG>",
  "data": {
    "qp_list_detailes": [
      { "qp_code": "...", "course": "Name(Code)", "No_of_bundles": 0, "No_of_Answer_Script": 0 }
    ],
    "qp_detailes_csv": [
      { "qp_code": "...", "course": "Name(Code)", "bundle_code": "...", "no_of_answer_script_each_bundle": 0 }
    ]
  }
}
```
Both lists are hand-built in a nested Python loop over `BundleMaster` rows filtered down
through `ProgrammeClassMaster → Programme → ProgrammeSemester → ProgrammeCourseSemester →
BundleMaster → QuestionPaperExamMapper → Examination (filtered by year) → QuestionPaperExamMapper
→ BundleMaster` — a long chained re-filter, confirmed directly from source. Field names are
this file's own dict keys, not a serializer's — confirmed, not external.

### Response — failure (400)

```json
{ "success": false, "message": "<value of BAD_GATEWAY>", "data": {}, "error_code": "<value of BAD_GATEWAY_ERROR_CODE>" }
```

### Notes

- **Correctly guarded against the unassigned-variable pattern**: `Qp_code = None` and
  `course = None` are initialized before the inner loop, and the outer `if Qp_code and course:`
  check gates their use afterward — unlike some of the bugs found elsewhere in this app, this
  loop does NOT have an unassigned-variable risk; it's a defensive pattern done correctly.
- The two-pass `QuestionPaperExamMapper` → `Examination` → `QuestionPaperExamMapper` re-filter
  (lines 92–96) re-derives `qp_exm_obj` from a **different** query the second time (all mappers
  for the year-filtered exams, replacing the qp_code-filtered set entirely) — dense but
  internally consistent; flagged only as complex/worth double-checking against intended
  behavior, not asserted as a bug.
- Uses `list(set(...))` twice to de-duplicate qp codes / exam ids by looping and casting to a
  Python `set` — functionally fine, just an unindexed in-Python dedup rather than a DB-level
  `.distinct()`.

## Notes on this source file

- **Likely `NameError` inside every exception handler**: all three `except Exception as e:`
  blocks in this file call `logger.error(e, exc_info=True)`, but `logger` is never imported or
  assigned anywhere in `qp_list_detailes.py` (checked directly — no `import logging`, no
  `logger = logging.getLogger(...)`, unlike `cdunit_mobile.py` and `article.py`, which both
  define their own module-level `logger`). If `logger` isn't supplied by one of this file's
  wildcard imports (`util.models`, `util.serializers`, `util.constants`, `util.urls` — none
  included in this upload, so this can't be fully ruled out), then **any exception raised inside
  any of these three view methods triggers a second, unhandled `NameError` while trying to log
  the first exception** — and since that second error occurs inside the `except` block itself,
  it is not caught by anything, so it propagates up as an unhandled 500 error instead of the
  intended graceful `format_response(False, BAD_GATEWAY, ...)` reply. This is a more serious
  variant of the `dt`-not-imported issue found in `stationery_items_approval.py`: there, the
  broad `except` still caught the secondary `NameError` and returned *some* response; here, the
  `NameError` happens *inside the except handler that would have caught it*, so nothing catches
  it at all.
- `from util.urls import *` is an unusual wildcard import for a views file (importing from a
  `urls` module) — not seen in any other file in this app. Contents not confirmable (external).

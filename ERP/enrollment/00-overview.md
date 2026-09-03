# enrollment — Overview

## What this module does

The `enrollment` app covers two related areas of the University of Kerala
system, based on the routed views and their linked templates:

1. **Student enrollment lifecycle** (`enroll.py`) — district/college/programme
   lookup data for the enrollment forms, bulk student enrollment upload,
   per-student enrollment view/upload, and admin-side windows to *enable* and
   *extend* enrollment windows for a programme/college/batch (`Enrollment
   Enable`, `Enrollment Extend`, `Enrolment Schedule List` templates).
2. **Candidate Code Generation Schedule** (`c_code_gen.py`) — a cascading
   Programme Type → Programme Group → Programme Class → Admission Year →
   College picker (`Candidate Code Generation Schedule` template) used to
   scope which programmes/colleges a candidate-code generation run applies
   to.

Both areas scope their programme lists through the logged-in staff member's
section assignment: `StaffBase` → `StaffSection` → `SubsectionProgramme`,
rather than exposing all programmes to all staff.

## Response format used across all APIs

Every routed view in both `enroll.py` and `c_code_gen.py` returns through a
single shared helper, `format_response(...)`, called consistently as:

- **Success:** `format_response(True, <message>, data=<dict>,
  status_code=status.HTTP_200_OK|201_CREATED, template_name='<template>.html')`
- **Failure:** `format_response(False, BAD_GATEWAY, {}, BAD_GATEWAY_ERROR_CODE,
  status_code=status.HTTP_400_BAD_REQUEST|500, template_name='<template>.html')`

No routed view in either file falls back to a raw `Response(...)` call — the
envelope is used uniformly, which is good hygiene. However:

- `format_response()` itself is **not defined in this upload** (it's not in
  `enroll.py`, `c_code_gen.py`, `serializers.py`, or the boilerplate files).
  It's imported implicitly via one of the `util.*` wildcard imports. The
  exact JSON key names it produces (e.g. whether the success flag comes back
  as `"success"`, `"status"`, etc.) are **not confirmable** from this
  upload — the module docs show only the arguments passed *into* the
  helper, not its output shape.
- Success calls consistently pass a **boolean** `True`/`False` as the first
  argument (not a numeric status).
- Many purely read-only "fetch a list" endpoints return
  `status.HTTP_201_CREATED` on success instead of `200_OK` (e.g. district
  list, college list, programme list, admission-year list). `201 Created`
  is semantically meant for resource-creation responses; using it for GET/
  POST-as-fetch endpoints that create nothing is inconsistent, though
  functionally harmless since the frontend presumably just checks the
  envelope's success flag.

## A note on source visibility

This upload contains only the `enrollment` Django app. It imports heavily
from a separate shared package via wildcard imports:

- `from util.constants import *` — status/message constants (e.g.
  `BAD_GATEWAY`, `BAD_GATEWAY_ERROR_CODE`, `FETCH_SUCCESS_MSG`,
  `EXAM_CNTR_FETCH_SUCCESS_MSG`) — **not included**, so literal values of
  these constants are shown as `<value of CONSTANT_NAME>`, not guessed.
- `from util.models import *` — the real ORM models (`StaffBase`,
  `StaffSection`, `SubsectionProgramme`, `AdmissionYearMaster`,
  `ProgrammeTypeMaster`, `ProgrammeClassMaster`, `EnrollmentScheduleDate`,
  etc.) — **not included**. `enrollment/models.py` itself is empty
  Django boilerplate.
- `from util.serializers import *` — a second, larger serializer set is
  pulled in from here (e.g. `ProgrammeGroupSerializer`, used in
  `c_code_gen.py` but never defined locally) — **not included**.

Only `enrollment/serializers.py` (157 lines, 8 classes) is local and fully
readable. Every module doc marks fields as **confirmed** when the
serializer is one of those 8, and **external, not confirmable** when it
comes from `util.serializers`.

`enrollment/models.py`, `admin.py`, `views.py`, and `tests.py` are all
untouched Django boilerplate (3–5 lines each, no custom code) and are not
documented separately.

## Sub-areas table

| Source file | Endpoints | Description | Status |
| --- | --- | --- | --- |
| `c_code_gen.py` | 5 | Candidate Code Generation Schedule cascade (Programme Type → Group → Class → Admission Year → College) | ✅ Complete — found a copy-paste bug where the "College List" endpoint returns admission-year data instead |
| `enroll.py` | 26 | Enrollment district/college/programme lookups, bulk upload, per-student enrollment, and Enable/Extend windows | ✅ Complete — found a `transaction.atomic()` gap in two write endpoints, a 4-instance recurring unassigned-variable bug, a caste/religion field mixup, and a list-reference-aliasing bug |

**Scope estimate:** 2 module files behind `urls.py`, 31 live routed
endpoints total, ~996 lines of view code (`enroll.py` 798 + `c_code_gen.py`
198), plus a fully-readable 157-line local `serializers.py`.

**Pacing:** went smallest-file-first — `c_code_gen.py` (198 lines, 5
endpoints), then `enroll.py` (798 lines, 26 endpoints), each read in full
before writing its doc. Both module files are now documented; this is the
full doc set for the `enrollment` app.

## Known cross-cutting issues

### 1. Stray `"` embedded in `template_name` on failure paths
On several failure-path `format_response()` calls, the `template_name`
string literal has a trailing double-quote character embedded *inside* the
string value itself, e.g. (from `c_code_gen.py`):

```python
template_name='c_code_gen_schedule.html"'
```

The string is single-quoted, so the value actually passed is
`c_code_gen_schedule.html"` — with a literal trailing `"` — which won't
match a real template filename. Confirmed present in multiple endpoints in
`c_code_gen.py` (see below) and in the equivalent `enrollment_enable.html`/
`enrollment_extend.html` failure paths in `enroll.py`, so this reads as a
copy-paste typo that propagated across both files rather than an isolated
mistake. Flagged here once; module docs will cross-reference this section
instead of re-explaining it.

### 2. No `permission_classes` anywhere in this app
Neither `enroll.py` nor `c_code_gen.py` sets `permission_classes` on any
routed view, even though both files import `AllowAny` and `IsAuthenticated`
from `rest_framework.permissions`. Every endpoint therefore falls back to
whatever DRF's project-wide default permission is — which isn't visible in
this upload. Worth confirming project settings, since several of these
endpoints (enrollment enable/extend, bulk upload) look like admin-only
actions.

## Notes on this source file
(none yet — see individual module docs)

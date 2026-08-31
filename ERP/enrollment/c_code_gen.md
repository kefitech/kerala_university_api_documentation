# c_code_gen

Source file: `enrollment/c_code_gen.py`. URL mapping is defined in
`enrollment/urls.py`.

Status: 5 of 5 routed endpoints documented. Contains one significant bug —
the "College List" endpoint (`CCodeGenScheduleCollegeList`) returns
admission-year data, not colleges — and a second endpoint
(`CCodeGenScheduleAdmissionYear`) that silently ignores the cascade filters
it reads from the request. See the "Known cross-cutting issues" section in
`00-overview.md` for the `template_name` typo shared with `enroll.py`.

## 1. Candidate Code Generation — Programme Type List

| Field | Value |
| --- | --- |
| **URL** | `/c_code_generation_schedule/` |
| **Method** | GET |
| **Auth** | No `permission_classes` set — falls back to project default (not confirmable, see overview) |
| **View class** | `CCodeGenScheduleProgrammeTypeList` (`ListAPIView`) |

### Request body
No parameters read (the view reads `jsondata = request.data` but never
uses it — dead read on a GET request).

### Response — success (200)
Programme types are derived from the logged-in staff's section
assignments, not queried directly — the view walks
`StaffBase → StaffSection → SubsectionProgramme` and collects each
programme's `programme_type`, de-duplicated in a Python loop (not via
`.distinct()`), then serializes with the local `ProgrammeTypeSerializer`
(confirmed fields: `id`, `title`).

```json
{
  "success": true,
  "message": "success",
  "data": {
    "prgtype": [
      {"id": 1, "title": "..."}
    ]
  }
}
```
*(envelope key names — `success`/`message`/`data` — are inferred from the
argument order passed into `format_response()`, not confirmed; see
overview.)*

### Response — failure (400)
```json
{
  "success": false,
  "message": "<value of BAD_GATEWAY>",
  "error_code": "<value of BAD_GATEWAY_ERROR_CODE>",
  "data": {}
}
```

### Notes
- `staffbase = StaffBase.objects.filter(user=request.user).first()` has no
  explicit `None` check. It doesn't crash outright, though: the next line,
  `StaffSection.objects.filter(staff=staffbase)`, passes `staffbase=None`
  straight into a `filter()`, which Django resolves as `staff__isnull=True`
  rather than raising. So a user with no `StaffBase` row won't 500 here —
  they'll silently get whichever `StaffSection` rows (if any) have a null
  `staff` FK, which is probably not the intended result either. Worth
  confirming whether that's ever a real, non-empty case in this schema.
- `except Exception as e:` is broad but not silent — it logs
  (`logger.error(e, exc_info=True)`) and returns a proper failure envelope,
  so this isn't the "log-and-fall-through" pattern.

## 2. Candidate Code Generation — Programme Group List

| Field | Value |
| --- | --- |
| **URL** | `/c_code_gen_sch_gp/` |
| **Method** | POST |
| **Auth** | No `permission_classes` set |
| **View class** | `CCodeGenScheduleProgrammeGroupList` (`ListAPIView`) |

### Request body
| Field | Type | Notes |
| --- | --- | --- |
| `prgm_type[]` | array | Read via `request.data.get('prgm_type[]')` — **read but never applied to the query** (see Notes) |

### Response — success (200)
Same `StaffBase → StaffSection → SubsectionProgramme` walk as endpoint 1,
this time collecting unique `programme_group` values, serialized with the
**external** `ProgrammeGroupSerializer` (imported from `util.serializers`
— fields not confirmable, not defined in this upload).

```json
{
  "success": true,
  "message": "success",
  "data": {
    "prggrp": ["<ProgrammeGroupSerializer output — external, not confirmable>"]
  }
}
```

### Response — failure (400)
Same shape as endpoint 1, `<value of BAD_GATEWAY>` / `<value of
BAD_GATEWAY_ERROR_CODE>`. `template_name` here has the stray-quote bug
described in the overview:
```python
template_name='c_code_gen_schedule.html"'
```

### Notes
- **Unused filter param:** `prgm_type` is read from the POST body but
  never referenced anywhere in the `SubsectionProgramme.objects.filter(...)`
  call below it — the group list is computed across *all* of the staff
  member's subsection programmes, regardless of which programme type the
  user selected in step 1 of the cascade. Compare with endpoint 3
  (`CCodeGenScheduleProgrammeClassList`), which does apply an equivalent
  filter correctly — this looks like an omission specific to this
  endpoint rather than an intentional design choice.

## 3. Candidate Code Generation — Programme Class List

| Field | Value |
| --- | --- |
| **URL** | `/c_code_gen_sch_cls/` |
| **Method** | POST |
| **Auth** | No `permission_classes` set |
| **View class** | `CCodeGenScheduleProgrammeClassList` (`ListAPIView`) |

### Request body
| Field | Type | Notes |
| --- | --- | --- |
| `prgm_type[]` | array | Applied — filters `programme__programme_type_id__in` |
| `prgm_group[]` | array | Applied — filters `programme__programme_group_id__in` |

### Response — success (200)
```python
subsectionprg = SubsectionProgramme.objects.filter(
    sub_section_id__in=subsection,
    programme__programme_group_id__in=prgm_group,
    programme__programme_type_id__in=prgm_type,
).order_by('-programme__title')
```
Unique `programme_class` values, serialized with the local
`CCGenProgrammeClassSerializer` (confirmed fields: `id`, `title`).

```json
{
  "success": true,
  "message": "success",
  "data": {
    "prgcls": [
      {"id": 1, "title": "..."}
    ]
  }
}
```

### Response — failure (400)
Same envelope shape; same `template_name='c_code_gen_schedule.html"'`
stray-quote bug as endpoint 2.

### Notes
This is the one endpoint in the cascade that correctly applies **both**
upstream filters it receives — useful as the reference point for how
endpoints 2, 4, and 5 should have behaved.

## 4. Candidate Code Generation — Admission Year List

| Field | Value |
| --- | --- |
| **URL** | `/c_code_gen_sch_admyr/` |
| **Method** | POST |
| **Auth** | No `permission_classes` set |
| **View class** | `CCodeGenScheduleAdmissionYear` (`ListAPIView`) |

### Request body
| Field | Type | Notes |
| --- | --- | --- |
| `prgm_type[]` | array | Read, then **discarded entirely** |
| `prgm_group[]` | array | Read, then **discarded entirely** |
| `prgm_class[]` | array | Read, then **discarded entirely** |
| `adm_year[]` | array | Read, then **immediately overwritten** by the queryset result (see below) |

### Response — success (200)
```python
adm_year = request.data.get('adm_year[]')
...
adm_year = AdmissionYearMaster.objects.filter().all()
adm_year = CCGenAdmissionYear(adm_year, many=True)
```
All four request fields are read into local variables and then never used
— `adm_year` in particular is reassigned twice more, so the client-supplied
value is silently thrown away. The query itself is an unfiltered
`.filter().all()` — every `AdmissionYearMaster` row is returned regardless
of the selected programme type/group/class. Serialized with the local
`CCGenAdmissionYear` (confirmed fields: `id`, `academic_year`,
`admission_year`).

```json
{
  "success": true,
  "message": "success",
  "data": {
    "admyear": [
      {"id": 1, "academic_year": "...", "admission_year": "..."}
    ]
  }
}
```

### Response — failure (400)
Same envelope shape; same stray-quote `template_name` bug.

### Notes
- A leftover `print(adm_year)` debug statement remains before the return.
- Given the cascade is meant to narrow admission years by
  type/group/class, this endpoint currently returns the same, unfiltered
  full list no matter what the user picked upstream.

## 5. Candidate Code Generation — College List

| Field | Value |
| --- | --- |
| **URL** | `/c_code_gen_sch_clglist/` |
| **Method** | POST |
| **Auth** | No `permission_classes` set |
| **View class** | `CCodeGenScheduleCollegeList` (`APIView`) |

### Request body
No parameters read at all (not even `.get()` calls for the cascade
fields — this view doesn't touch `request.data` beyond the initial
`jsondata = request.data`, which is also unused).

### Response — success (200) — likely bug
```python
class CCodeGenScheduleCollegeList(APIView):
    def post(self, request):
        try:
            jsondata=request.data

            adm_year=AdmissionYearMaster.objects.filter().all()
            adm_year=CCGenAdmissionYear(adm_year,many=True)
            print(adm_year)

            return format_response(True,"success",{"admyear":adm_year.data},status_code=status.HTTP_200_OK,template_name="c_code_gen_schedule.html")
```
This is byte-for-byte the same query, serializer, response key
(`"admyear"`), and even the same leftover `print()` as endpoint 4
(`CCodeGenScheduleAdmissionYear`). Despite the class name, URL
(`c_code_gen_sch_clglist/`), and URL name (`c_code_gen_sch_clg_list`) all
promising a **college** list, the endpoint returns
**admission years** — there's no college model or serializer referenced
anywhere in this method. This looks like a copy-paste of endpoint 4 where
the developer forgot to swap in the real college query, leaving the
cascade's final step (Programme Type → Group → Class → Admission Year →
**College**) with no working implementation.

```json
{
  "success": true,
  "message": "success",
  "data": {
    "admyear": [
      {"id": 1, "academic_year": "...", "admission_year": "..."}
    ]
  }
}
```

### Response — failure (400)
Same envelope shape as endpoint 4, `template_name="c_code_gen_schedule.html"`
(no stray-quote bug on this one's failure path — it uses double quotes
throughout, matching endpoint 1's clean version rather than endpoints 2–4's
typo).

### Notes
Standout finding for this file — see "Known cross-cutting issues" in the
overview for the shared `template_name` typo; this endpoint's issue is
separate and specific to this file.

## Notes on this source file

- **Response envelope:** all 5 endpoints use `format_response()`
  consistently; no raw `Response()` calls, no shape deviations.
- **Auth:** no endpoint in this file sets `permission_classes`, despite
  `AllowAny`/`IsAuthenticated` being imported at the top — see overview.
- **Cascade filter pattern:** of the four downstream steps in this
  cascade (Group, Class, Admission Year, College), only Class (endpoint 3)
  correctly applies the filters it's handed. Group drops one filter,
  Admission Year drops all of them, and College doesn't even query the
  right model. Worth treating this file as needing a working pass on the
  whole cascade, not just a one-line fix to the College endpoint.
- **Debug leftovers:** two `print(adm_year)` statements (endpoints 4 and
  5) look like they were left in from development.
- **`.filter()` before `.all()`:** `AdmissionYearMaster.objects.filter().all()`
  (endpoints 4 and 5) — the empty `.filter()` is a no-op; functionally
  identical to `AdmissionYearMaster.objects.all()`, just unusual style.

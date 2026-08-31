# enroll

Source file: `enrollment/enroll.py`. URL mapping is defined in
`enrollment/urls.py`.

Status: All 26 routed endpoints documented. Two write endpoints
(`StudentUpload`, `EnrollmentDateEnable`) run multi-record creation loops
with no `transaction.atomic()`. A specific bug pattern — a variable
assigned only inside a loop/conditional that might not execute, then read
unconditionally afterward — recurs in **four separate endpoints**
(`DistrictCollegeMapping`, `StudentView`, `PrgmGrpClsCollegeView`,
`CollegeProgrammeView`). `StudentView` also has a caste/religion field
mixup, and `EnrollmentSchdlView` has a list-reference-aliasing bug that
makes every programme in its response show the same (fully accumulated)
college list. See `00-overview.md` for the `template_name` typo and
missing-`permission_classes` issues, which also apply throughout this
file.

## 1. Enrollment District List

| Field | Value |
| --- | --- |
| **URL** | `/enrollment-district-view/` |
| **Method** | GET |
| **Auth** | No `permission_classes` set |
| **View class** | `EnrollmentDistrict` (`APIView`) |

### Request body
No parameters.

### Response — success (200)
Districts are derived from the staff member's section-scoped programmes →
colleges, de-duplicated via `list(dict.fromkeys(...))`, serialized with
the **external** `DistrictSerializers` (from `util.serializers` — fields
not confirmable).
```json
{
  "success": true,
  "message": "<value of EXAM_CNTR_FETCH_SUCCESS_MSG>",
  "data": {
    "districts": ["<DistrictSerializers output — external, not confirmable>"]
  }
}
```

### Response — failure (500)
```json
{
  "success": false,
  "message": "<value of BAD_GATEWAY>",
  "error_code": "<value of BAD_GATEWAY_ERROR_CODE>",
  "data": {}
}
```

### Notes
`staffbase=StaffBase.objects.filter(user=request.user).first()` has no
`None` check, same as every other endpoint in this file that does this
lookup — see "Notes on this source file" below rather than repeating this
per endpoint.

## 2. District → College Mapping

| Field | Value |
| --- | --- |
| **URL** | `/district-college-view/` |
| **Method** | GET |
| **Auth** | No `permission_classes` set |
| **View class** | `DistrictCollegeMapping` (`APIView`) |

### Request body
| Field | Type | Notes |
| --- | --- | --- |
| `district_id` | query param | `request.GET['district_id']` — required, KeyError if absent (caught by the outer `except`, returned as a generic failure) |

### Response — success (201) — unassigned-variable risk
```python
subsection=[]
for  subsect in staffsection:
    subsection.append(subsect.sub_section)
    district_list = []
    subsectionprg=SubsectionProgramme.objects.filter(sub_section_id__in=subsection).order_by('-programme__title')
    for subsectprg in subsectionprg:
        ...
        district_list.append(colg)
    college_list=AffiliatedCollege.objects.filter(id__in=district_list,district=distict_id).order_by('code')
    college = CollegeSerializer(college_list,many=True)
return format_response(True,FETCH_SUCCESS_MSG,data={"clglist":college.data},...)
```
`district_list`, `college_list`, and `college` are all built **inside**
the `for subsect in staffsection:` loop, and the `return` (outside the
loop) reads `college` from whatever the loop last set it to. If
`staffsection` is ever empty — e.g. the current user has no `StaffBase`/
`StaffSection` rows — the loop body never runs, `college` is never
assigned, and this line raises `NameError: name 'college' is not
defined` instead of returning an empty list gracefully. Separately, this
also means the college query re-runs once per `staffsection` row, using
only the last iteration's (cumulative) result — functionally correct
when the loop runs at least once, just wasteful.
```json
{
  "success": true,
  "message": "<value of FETCH_SUCCESS_MSG>",
  "data": {
    "clglist": ["<CollegeSerializer output — external, not confirmable>"]
  }
}
```

### Response — failure (400)
Standard envelope, `<value of BAD_GATEWAY>` / `<value of
BAD_GATEWAY_ERROR_CODE>`.

### Notes
See "Notes on this source file" — this is instance 1 of 4 of the same
unassigned-variable pattern in this file.

## 3. College → Programme Mapping

| Field | Value |
| --- | --- |
| **URL** | `/college-programme-mapping/` |
| **Method** | GET |
| **Auth** | No `permission_classes` set |
| **View class** | `CollegeProgrammeMapping` (`APIView`) |

### Request body
| Field | Type | Notes |
| --- | --- | --- |
| `college_id` | query param | `request.GET['college_id']` — required |

### Response — success (201)
Walks `CollegeDepartment → CollegeProgramme → CollegeBatchProgramme` for
the given college, serializing each match's programme with the local
`ProgrammeClassSerializers` (confirmed fields: `id`, `code`, `title`,
`pclass`).
```json
{
  "success": true,
  "message": "<value of FETCH_SUCCESS_MSG>",
  "data": {
    "prglist": [
      {"id": 1, "code": "...", "title": "...", "pclass": "..."}
    ]
  }
}
```

### Response — failure (400)
Standard envelope.

### Notes
Unlike most other endpoints in this file, this one does **not** scope by
the caller's staff section — it resolves purely from the given
`college_id`. That may be intentional (college was already scoped
upstream by endpoint 2), but it's worth confirming.

## 4. Programme → Admission Year Mapping

| Field | Value |
| --- | --- |
| **URL** | `/programme-year-mapping/` |
| **Method** | GET |
| **Auth** | No `permission_classes` set |
| **View class** | `ProgrammeYearMapping` (`APIView`) |

### Request body
| Field | Type | Notes |
| --- | --- | --- |
| `collegeid` | query param | required |
| `programme_id` | query param | required |

### Response — success (201)
Returns each matching `CollegeBatchProgramme`'s academic year, serialized
with the **external** `AdmissionYearSerializer` (from `util.serializers`
— not the local `CCGenAdmissionYear`; fields not confirmable).
```json
{
  "success": true,
  "message": "<value of FETCH_SUCCESS_MSG>",
  "data": {
    "adm_list": ["<AdmissionYearSerializer output — external, not confirmable>"]
  }
}
```

### Response — failure (400)
Standard envelope.

### Notes
`jsondata = request.data` is read and never used (dead read on a GET
request) — same harmless pattern seen repeatedly through this file;
noted once here, not flagged again per occurrence.

## 5. Enrollment Bulk Upload — Preview

| Field | Value |
| --- | --- |
| **URL** | `/enrollment-bulk-upload/` |
| **Method** | POST |
| **Auth** | No `permission_classes` set |
| **View class** | `EnrollmentBulkUpload` (`APIView`) |

### Request body
| Field | Type | Notes |
| --- | --- | --- |
| `sheet` | file upload | Excel file, read via `pandas.read_excel`, expects columns: `ApplicationNumber, StudentId, StudentName, Gender, DOB, RelCastCode, HouseName, Place, PostOffice, Street, District, Pincode, Phone, Email, Mobile, FeeConcessionStatus` |

### Response — success (201)
Parses the sheet to a list of dicts and adds `delete_status: 0` to every
row — this looks like a staging/preview step (nothing is persisted to
the DB here); presumably `StudentUpload` (endpoint 7) does the real
write once the user reviews this preview.
```json
{
  "success": true,
  "message": "<value of FETCH_SUCCESS_MSG>",
  "data": [
    {"ApplicationNumber": "...", "StudentId": "...", "...": "...", "delete_status": 0}
  ]
}
```

### Response — failure (400)
Standard envelope.

### Notes
```python
excel_data_df = pandas.read_excel(request.FILES['sheet'])
excel_data_df = pandas.read_excel(request.FILES['sheet'], sheet_name='Sheet1', usecols=[...])
```
The file is read twice — the first `read_excel()` call's result is
immediately discarded by the second, more specific call. Harmless, but
reads like a leftover from development rather than an intentional
double-read.

## 6. Student View — Single Row Lookup

| Field | Value |
| --- | --- |
| **URL** | `/studentwise-view/` |
| **Method** | POST |
| **Auth** | No `permission_classes` set |
| **View class** | `StudentView` (`APIView`) |

### Request body
| Field | Type | Notes |
| --- | --- | --- |
| `stdid` | string | Student ID to look up within the uploaded sheet |
| `college` | int | College ID |
| `sheet` | file upload | Same Excel format as endpoint 5 |

### Response — success (201) — two real bugs
Re-parses the uploaded sheet, finds the row matching `stdid`, and looks
up the college code, district title, caste, and religion for display.

**Bug 1 — caste/religion mixup.** The caste lookup result is thrown away
and replaced by the religion lookup, so both `caste` and `religion` in
the response come from the *same* religion record:
```python
cast=UserCasteMaster.objects.filter(code=cas).first()
rel=cast=UserReligionMaster.objects.filter(code=rel).first()
```
The first line correctly looks up caste into `cast`. The very next line
immediately overwrites `cast` (via the chained `rel=cast=...`
assignment) with a `UserReligionMaster` lookup — so the caste value
computed on the line above is discarded before it's ever used, and
`data["caste"]` ends up populated from a religion record, not a caste
record.

**Bug 2 — unassigned-variable risk, three variables.** `col_code`,
`dist_title`, `cast`, and `rel` are each assigned only inside a
conditional block that may not execute:
```python
for det in collegedata:
  col_code=det['code']          # only set if collegedata is non-empty
...
for al in alldata:
  st=str(al['StudentId'])
  if st==str(stdid):            # only set if a row matches stdid
    ...
    for di in distr:
      dist_title=di['title']    # only set if distr is non-empty, nested inside the match branch
data={
  "datarecord":datarecord,
  "colcode":col_code,
  "district":dist_title,
  "caste":cast.name,
  "religion":rel.name,
}
```
If the given `college` ID doesn't resolve to a row, `col_code` is never
assigned. If no row in the sheet matches `stdid`, `dist_title`, `cast`,
and `rel` are never assigned. Either case raises a `NameError` when
building `data` (or `AttributeError` on `cast.name`/`rel.name` if `cast`/
`rel` are assigned but `None` from a `.first()` miss). This is the same
class of bug as endpoint 2, but manifesting through unmet `if`
conditions and nested loops rather than an empty outer loop.

```json
{
  "success": true,
  "message": "<value of FETCH_SUCCESS_MSG>",
  "data": {
    "datarecord": {"...": "..."},
    "colcode": "...",
    "district": "...",
    "caste": "<actually the religion name — see Bug 1>",
    "religion": "..."
  }
}
```

### Response — failure (400)
Standard envelope.

### Notes
This endpoint re-parses the entire uploaded sheet from scratch on every
single-student lookup (the `sheet` file has to be re-submitted with each
call) rather than looking up a previously-persisted student — consistent
with endpoint 5 being a pure preview step with nothing saved yet.

## 7. Student Upload — Bulk Create

| Field | Value |
| --- | --- |
| **URL** | `/studentwise-upload/` |
| **Method** | POST |
| **Auth** | No `permission_classes` set |
| **View class** | `StudentUpload` (`APIView`) |

### Request body
| Field | Type | Notes |
| --- | --- | --- |
| `college` | int | required |
| `adm_year` | int | required |
| `prgm` | int | required |
| `year_ad` | string | used only to build photo/signature URLs |
| `deletedid` | string | `+`-separated list of `StudentId`s to skip |
| `sheet` | file upload | Same Excel format as endpoint 5 |

### Response — success (201) — headline bug: no transaction, broken credentials email
This is the real write path behind the preview in endpoint 5: for every
non-skipped row in the sheet, it creates a `User`, `UserProfiles`,
`UserAddress`, `StudentBase`, `StudentDetail`, `StudentSemester`, and
`SemesterRegistrationStatus`, adds the user to the `Enrolled(Student)`
group, and emails them their credentials.

**No `transaction.atomic()`.** All of this happens in a plain Python
`for` loop with a single `try/except` around the *entire* view. Since
none of the per-student model creates are wrapped in a transaction, a
failure partway through the sheet (see the missing-`None`-check below)
leaves the students processed *before* the failure fully created in the
database — accounts, profiles, addresses, everything — while the rest of
the sheet is never attempted, and the caller only sees one generic
failure response with no indication of which rows succeeded.

A concrete way this fires: `dist=District.objects.filter(id=data['District']).first()` has no `None` check, and the very next line uses it:
```python
dist=District.objects.filter(id=data['District']).first()
country=Country.objects.filter(id=1).first()
if dist.id==15:
```
If a row's `District` value doesn't match any `District` row, `dist` is
`None` and `dist.id` raises `AttributeError` — aborting the whole
request (and everything already committed for earlier rows in the sheet
stays committed, per the point above). The same no-`None`-check pattern
also applies to `adtype`, `country`, `province`, `admm_year`,
`colgprgmsem`, `cast`, and `gen` in this same loop — none of the seven
`.first()` calls that feed into the `.create()` calls below are checked.

**Credentials email never includes the credentials.**
```python
email_body = "Hi,\n\n Your Username and Password are "
data1 = {'email_body': email_body, 'to_email':mail,
      'email_subject': 'Login Credentials'}
Util.send_email(data1)
```
`email_body` is a fixed string — the actual username
(`user_data.username`) and generated password (`getdatestring`) are never
interpolated into it. Every new student's "Login Credentials" email says
"Your Username and Password are" and then... stops. The subject promises
credentials the body never delivers.

**Security-sensitive pattern:** the auto-generated password is the
student's own date of birth, formatted `%d-%m-%Y`
(`user_data.set_password(getdatestring)`) — a low-entropy, guessable
default credential for anyone who knows (or can find) the student's DOB.

**Hardcoded PKs:** `AddressTypeMaster` id `1`, `Country` id `1`,
`Province` id `1`/`2` (branched on `dist.id==15`) are all hardcoded
rather than looked up by code/label — consistent with the same
hardcoded-PK pattern flagged elsewhere in this university system's other
apps.

```json
{
  "success": true,
  "message": "<value of FETCH_SUCCESS_MSG>",
  "data": {}
}
```

### Response — failure (400)
Standard envelope. Given the lack of a transaction, a failure response
here does **not** mean nothing was written — see above.

### Notes
- `colgprgmsem` is queried twice with the identical filter (once
  immediately after `admm_year`, unused until the second, later
  duplicate query is actually used in the `StudentSemester.objects.create(...)`
  call) — dead/redundant query.
- `if ex_user.exists(): {}` — the "student already exists, skip" branch
  is a bare `{}` expression (a no-op dict literal), functioning like
  `pass` but unusual style.
- `getdate=data['DOB'].date()` assumes pandas parsed the `DOB` column as
  a real timestamp; a blank or unparseable cell would raise
  `AttributeError` here, caught only by the same broad, transaction-less
  `except` discussed above.

## 8. Enrollment Enable — Scheme List

| Field | Value |
| --- | --- |
| **URL** | `/enrollment-enable-view/` |
| **Method** | GET |
| **Auth** | No `permission_classes` set |
| **View class** | `EnrollmentEnable` (`APIView`) |

### Request body
No parameters (`jsondata = request.data` is read, unused).

### Response — success (201)
All `SchemeMaster` rows, serialized with the **external**
`SchemeMasterSerializer` (fields not confirmable).
```json
{"success": true, "message": "<value of FETCH_SUCCESS_MSG>", "data": {"scheme": ["<external — not confirmable>"]}}
```

### Response — failure (400)
Standard envelope.

### Notes
None beyond the file-wide patterns.

## 9. Enrollment Enable — Programme Type List

| Field | Value |
| --- | --- |
| **URL** | `/enrollment-scheme-type/` |
| **Method** | POST |
| **Auth** | No `permission_classes` set |
| **View class** | `EnrollmentProgrammeType` (`APIView`) |

### Request body
| Field | Type | Notes |
| --- | --- | --- |
| `scheme` | comma-separated string | Scheme IDs |

### Response — success (201)
Intersects (in Python, via `set(p) & set(prgm_typ)`) the programme types
reachable from the selected schemes with the programme types the caller's
staff section covers, then serializes with the **external**
`ProgrammeTypeMasterPanelSerializers`.
```json
{"success": true, "message": "<value of FETCH_SUCCESS_MSG>", "data": {"prgmdata": ["<external — not confirmable>"]}}
```

### Response — failure (400)
Standard envelope.

### Notes
The type intersection is done with Python `set()` operations after
pulling both querysets fully into memory, rather than a single filtered
DB query — functionally fine at this app's likely scale, just less
efficient than it could be.

## 10. Enrollment Enable — Programme Group List

| Field | Value |
| --- | --- |
| **URL** | `/enrollment-prgm-grp/` |
| **Method** | POST |
| **Auth** | No `permission_classes` set |
| **View class** | `EnrollmentProgrammeGroupList` (`ListAPIView`) |

### Request body
| Field | Type | Notes |
| --- | --- | --- |
| `prgm_type` | comma-separated string | Applied — filters `programme__programme_type_id__in` |

### Response — success (200)
Correctly filters by `prgm_type` (unlike the equivalent endpoint in
`c_code_gen.py`, which drops this filter — see that module's notes).
Serialized with the **external** `ProgrammeGroupSerializer`.
```json
{"success": true, "message": "success", "data": {"prggrp": ["<external — not confirmable>"]}}
```

### Response — failure (400)
```python
template_name='enrollment_enable.html"'
```
Has the stray-quote `template_name` typo — see overview.

### Notes
None beyond the shared typo above.

## 11. Enrollment Enable — Programme Class List

| Field | Value |
| --- | --- |
| **URL** | `/enrollment-prgm-cls/` |
| **Method** | POST |
| **Auth** | No `permission_classes` set |
| **View class** | `EnrollmentProgrammeClassList` (`ListAPIView`) |

### Request body
| Field | Type | Notes |
| --- | --- | --- |
| `prgm_type` | comma-separated string | Applied |
| `prgm_group` | comma-separated string | Applied |

### Response — success (200)
Both filters correctly applied. Serialized with the **external**
`ProgrammeClassMasterPanelSerializers`.
```json
{"success": true, "message": "success", "data": {"prgcls": ["<external — not confirmable>"]}}
```

### Response — failure (400)
Same stray-quote `template_name` typo as endpoint 10.

### Notes
None.

## 12. Enrollment Enable — Academic Year List

| Field | Value |
| --- | --- |
| **URL** | `/enrollment-acd-year/` |
| **Method** | POST |
| **Auth** | No `permission_classes` set |
| **View class** | `EnrollmentAcademicYear` (`ListAPIView`) |

### Request body
No parameters read.

### Response — success (200)
All `AdmissionYearMaster` rows (unfiltered — but unlike the equivalent
endpoint in `c_code_gen.py`, this one never claimed to accept cascade
filters in the first place, so it's not a regression, just a plain
unfiltered list). Serialized with the local `CCGenAdmissionYear`
(confirmed fields: `id`, `academic_year`, `admission_year`).
```json
{"success": true, "message": "success", "data": {"admyear": [{"id": 1, "academic_year": "...", "admission_year": "..."}]}}
```

### Response — failure (400)
Same stray-quote `template_name` typo.

### Notes
None.

## 13. Enrollment Enable — Enrollable List

| Field | Value |
| --- | --- |
| **URL** | `/enrollment-enable-list/` |
| **Method** | POST |
| **Auth** | No `permission_classes` set |
| **View class** | `EnrollmentListEnable` (`ListAPIView`) |

### Request body
| Field | Type | Notes |
| --- | --- | --- |
| `prgmtype` | comma-separated string | required |
| `prgmgrp` | comma-separated string | required |
| `prgmcls` | comma-separated string | required |
| `admyear` | value | required |

### Response — success (200) — missing None-check
```python
prgscheme=UniversityRegulationSchemeProgramme.objects.filter(programme__id=er['prg_id']).last()
er['scheme']=prgscheme.scheme.title
```
No `None` check on `.last()` — if a programme has no matching
`UniversityRegulationSchemeProgramme` row, this raises `AttributeError`
and fails the whole list for every programme, not just the one missing a
scheme mapping.

Serialized primarily via the local `SemesterRegistrationProgrammeSerializer`
(confirmed fields: `prg_id`, `programme_code`, `programme_title`,
`programme_class`), then enriched per-row with `display_status` (whether
any `EnrollmentScheduleDate` already exists for that programme/year) and
`scheme`.
```json
{
  "success": true,
  "message": "success",
  "data": [
    {"prg_id": "...", "programme_code": "...", "programme_title": "...", "programme_class": "...", "display_status": 1, "scheme": "..."}
  ]
}
```

### Response — failure (400)
Same stray-quote `template_name` typo.

### Notes
`.distinct("programme_semester__programme__id")` — PostgreSQL-specific
`DISTINCT ON` usage (consistent with this codebase's PostgreSQL-only
deployment target noted elsewhere in this system).

## 14. Enrollment — Enable With Date (write)

| Field | Value |
| --- | --- |
| **URL** | `/enrollment-enable-with-date/` |
| **Method** | POST |
| **Auth** | No `permission_classes` set |
| **View class** | `EnrollmentDateEnable` (`APIView`) |

### Request body
| Field | Type | Notes |
| --- | --- | --- |
| `start_date` | string | not validated |
| `end_date` | string | not validated |
| `admyear` | int | |
| `prgsem` | string | comma-separated `programmeId-classId` pairs |

### Response — success (201) — no transaction, missing None-checks
Creates one `EnrollmentSchedule` per distinct class in `prgsem`, then one
`EnrollmentScheduleDate` per matching `CollegeBatchProgramme`, in nested
loops:
```python
for x in newset1:
  cl=ProgrammeClassMaster.objects.filter(id=x).first()
  adm=AdmissionYearMaster.objects.filter(id=admyear).first()
  sch = EnrollmentSchedule.objects.create(schedule_name='Test',prgm_class=cl,academic_year=adm,...)
  for p in prg_cls:
    ...
    if x==prcl:
     colg=CollegeBatchProgramme.objects.filter(...)
     for cl in colg:
      schdate=EnrollmentScheduleDate.objects.create(enrolmnt_sch=sch,clg_batch_prgm=cl,...)
```
- No `transaction.atomic()` around this multi-record, nested-loop write —
  same partial-failure risk as endpoint 7 (`StudentUpload`): if a later
  iteration fails, earlier `EnrollmentSchedule`/`EnrollmentScheduleDate`
  rows already created in this request stay committed.
- `cl` and `adm` (from `.first()`) have no `None` check before being
  passed into `EnrollmentSchedule.objects.create(...)` — an unmatched
  `id` would either pass `None` into the FK (raising `IntegrityError` if
  the field is non-nullable) or silently create a schedule with a null
  class/year if it is nullable (not confirmable without the model
  definition).
- The inner loop reuses `cl` as its loop variable name
  (`for cl in colg:`), shadowing the outer `cl` (the
  `ProgrammeClassMaster` instance) for the rest of that outer iteration.
  It happens to work here because the outer `cl` isn't read again after
  the inner loop starts, but it's a fragile naming choice.
- `schedule_name='Test'` is a hardcoded literal — every schedule created
  through this endpoint gets the same name, "Test", regardless of what
  the caller might have wanted to call it.

```json
{"success": true, "message": "<value of FETCH_SUCCESS_MSG>", "data": {}}
```

### Response — failure (400)
Standard envelope. As with endpoint 7, a failure here doesn't guarantee
nothing was written.

### Notes
None beyond the above.

## 15. Current Date

| Field | Value |
| --- | --- |
| **URL** | `/enrollment-current-date/` |
| **Method** | GET |
| **Auth** | No `permission_classes` set |
| **View class** | `CurrentDate` (`APIView`) |

### Request body
No parameters.

### Response — success (201)
```json
{"success": true, "message": "<value of FETCH_SUCCESS_MSG>", "data": {"today": "<server date, date.today()>"}}
```

### Response — failure (400)
Standard envelope.

### Notes
Simplest endpoint in the file — no DB access, nothing to flag beyond the
201-for-a-non-creating-GET pattern noted in the overview.

## 16. Enrollment Extend — Programme Type List

| Field | Value |
| --- | --- |
| **URL** | `/enrollment-extend-view/` |
| **Method** | GET |
| **Auth** | No `permission_classes` set |
| **View class** | `EnrollmentExtend` (`APIView`) |

### Request body
No parameters.

### Response — success (201)
Same staff-section-derived unique-programme-type pattern as elsewhere,
serialized with the **external** `ProgrammeTypeMasterSerializers` (note:
distinct from the local `ProgrammeTypeSerializer` used in
`c_code_gen.py` — different class, external, fields not confirmable).
```json
{"success": true, "message": "<value of FETCH_SUCCESS_MSG>", "data": {"pgm_types": ["<external — not confirmable>"]}}
```

### Response — failure (400)
Standard envelope.

### Notes
None.

## 17. Enrollment Extend — Programme Group List

| Field | Value |
| --- | --- |
| **URL** | `/prgm-type-grp-view/` |
| **Method** | GET |
| **Auth** | No `permission_classes` set |
| **View class** | `PrgmTypeGrpView` (`APIView`) |

### Request body
| Field | Type | Notes |
| --- | --- | --- |
| `prgmtype` | query param | required, applied correctly to the filter |

### Response — success (201)
```json
{"success": true, "message": "<value of FETCH_SUCCESS_MSG>", "data": {"prggrp": ["<ProgrammeGroupSerializer output — external, not confirmable>"]}}
```

### Response — failure (400)
Standard envelope.

### Notes
None.

## 18. Enrollment Extend — Programme Class List

| Field | Value |
| --- | --- |
| **URL** | `/prgm-grp-cls-view/` |
| **Method** | GET |
| **Auth** | No `permission_classes` set |
| **View class** | `PrgmGrpClsView` (`ListAPIView`) |

### Request body
| Field | Type | Notes |
| --- | --- | --- |
| `prgmgrp` | query param | required |
| `prgmtype` | query param | required |

### Response — success (201)
Both filters applied correctly. Serialized with the **external**
`ProgrammeClassMasterSerializer` (distinct from the local
`CCGenProgrammeClassSerializer` — fields not confirmable).
```json
{"success": true, "message": "<value of FETCH_SUCCESS_MSG>", "data": {"prgclass": ["<external — not confirmable>"]}}
```

### Response — failure (400)
Standard envelope.

### Notes
None.

## 19. Enrollment Extend — College List

| Field | Value |
| --- | --- |
| **URL** | `/prgm-class-college-view/` |
| **Method** | GET |
| **Auth** | No `permission_classes` set |
| **View class** | `PrgmGrpClsCollegeView` (`ListAPIView`) |

### Request body
| Field | Type | Notes |
| --- | --- | --- |
| `prgmgrp` | query param | required |
| `prgmtype` | query param | required |
| `prgmcls` | query param | required |

### Response — success (201) — unassigned-variable risk (instance 2)
```python
subsection=[]
for  subsect in staffsection:
  subsection.append(subsect.sub_section)
  colg_list = []
  subsectionprg=SubsectionProgramme.objects.filter(
      sub_section_id__in=subsection,programme__programme_group_id=prggrp,
      programme__programme_type_id=prg_type,programme__programme_class_id=prgmcls
  ).order_by('-programme__title')
for subsectprg in subsectionprg:
    ...
    colg_list.append(colg)
college_list=AffiliatedCollege.objects.filter(id__in=colg_list).order_by('code')
```
`colg_list` and `subsectionprg` are assigned inside the `for subsect in
staffsection:` loop, but the `for subsectprg in subsectionprg:` line that
consumes `subsectionprg` is indented to sit **outside** that loop (a
sibling statement, not nested inside it) — so it only works because
Python leaves a `for` loop's variables holding their last value after the
loop ends. If `staffsection` is empty, `subsectionprg` (and `colg_list`)
are never assigned at all, and the `for subsectprg in subsectionprg:`
line raises `NameError: name 'subsectionprg' is not defined`. This is the
same underlying bug as endpoint 2 (`DistrictCollegeMapping`), just with
the two loops fully separated rather than nested together — see also
endpoint 20, which has the identical structure.
```json
{"success": true, "message": "<value of FETCH_SUCCESS_MSG>", "data": {"college": ["<CollegeSerializer output — external, not confirmable>"]}}
```

### Response — failure (400)
Standard envelope.

### Notes
See "Notes on this source file" — instance 2 of 4.

## 20. Enrollment Extend — Programme List

| Field | Value |
| --- | --- |
| **URL** | `/prgm-class-college-prgm-view/` |
| **Method** | GET |
| **Auth** | No `permission_classes` set |
| **View class** | `CollegeProgrammeView` (`ListAPIView`) |

### Request body
| Field | Type | Notes |
| --- | --- | --- |
| `prgmgrp` | query param | required |
| `prgmtype` | query param | required |
| `prgmcls` | query param | required |

### Response — success (201) — unassigned-variable risk (instance 3)
Identical structure to endpoint 19: `prgm_list` and `subsectionprg` are
set inside the `for subsect in staffsection:` loop, but the `for
subsectprg in subsectionprg:` loop that reads `subsectionprg` sits
outside it. Same `NameError: name 'subsectionprg' is not defined` risk if
`staffsection` is empty.
```json
{"success": true, "message": "<value of FETCH_SUCCESS_MSG>", "data": {"prgm": ["<ProgrammeMasterSerializers output — external, not confirmable>"]}}
```

### Response — failure (400)
Standard envelope.

### Notes
See "Notes on this source file" — instance 3 of 4. Endpoints 19 and 20
are near-duplicates of each other (same request params, same staff-
section walk, same bug) — differing only in whether the final step
resolves to colleges or programmes.

## 21. Enrollment Extend — Schedule Search

| Field | Value |
| --- | --- |
| **URL** | `/enrollment-prgm-view/` |
| **Method** | POST |
| **Auth** | No `permission_classes` set |
| **View class** | `EnrollmentPrgmView` (`ListAPIView`) |

### Request body
| Field | Type | Notes |
| --- | --- | --- |
| `prgmtype` | value | read but not applied to any filter (see Notes) |
| `prgmgrp` | value | read but not applied to any filter |
| `prgmcls` | value | read but not applied to any filter |
| `college` | comma-separated string | applied |
| `programme` | comma-separated string | applied |

### Response — success (200)
Finds existing `EnrollmentScheduleDate` rows matching the given colleges
and programmes, serialized with the local `EnrollmentSerializer`
(confirmed fields: `id`, `start_date`, `end_date`, `clg_batch_prgm`,
`prg_id`, `programme_title`, `colg`).
```json
{"success": true, "message": "success", "data": {"prg": [{"id": 1, "start_date": "...", "end_date": "...", "clg_batch_prgm": 1, "prg_id": "...", "programme_title": "...", "colg": "..."}]}}
```

### Response — failure (400)
```python
logger.error(e,exc_info=True)
logger.error(e,exc_info=True)
```
Same exception logged twice (harmless, copy-paste leftover). Also has the
stray-quote `template_name` typo.

### Notes
`prgmtype`, `prgmgrp`, and `prgmcls` are pulled out of the request body
but never referenced in the `EnrollmentScheduleDate` query below —
dead/unused parameters, similar in spirit to the dropped filters found in
`c_code_gen.py`. A commented-out earlier version of the query
(`#enrprg=EnrollmentScheduleDate.objects.filter(...)`) is left directly
above the live one.

## 22. Enrollment Extend — Modal Detail View

| Field | Value |
| --- | --- |
| **URL** | `/enrollment-modal-view/` |
| **Method** | POST |
| **Auth** | No `permission_classes` set |
| **View class** | `EnrollmentModalView` (`ListAPIView`) |

### Request body
| Field | Type | Notes |
| --- | --- | --- |
| `programme_sch` | comma-separated string | `EnrollmentScheduleDate` IDs |
| `college` | value | echoed back unchanged in the response, not otherwise used |

### Response — success (201)
```json
{"success": true, "message": "<value of FETCH_SUCCESS_MSG>", "data": {"prgm": [{"id": 1, "start_date": "...", "end_date": "...", "...": "..."}], "college": "<echoed back>"}}
```

### Response — failure (400)
Standard envelope.

### Notes
None.

## 23. Enrollment Extend — Extend Dates (write)

| Field | Value |
| --- | --- |
| **URL** | `/enrollment-modal-extend/` |
| **Method** | POST |
| **Auth** | No `permission_classes` set |
| **View class** | `EnrollmentModalExtend` (`ListAPIView`) |

### Request body
| Field | Type | Notes |
| --- | --- | --- |
| `enrid` | `-`-separated string | `EnrollmentScheduleDate` IDs to extend |
| `extendend_date` | value | new `end_date`, not validated |

### Response — success (201) — missing None-check, no transaction
```python
for i in enrid:
  if i:
    enrobj=EnrollmentScheduleDate.objects.filter(id=i).first()
    enrobj.end_date=newdate
    enrobj.save()
```
No `None` check on `.first()` — an `id` that doesn't match any row raises
`AttributeError` on `enrobj.end_date=newdate`. This is also a
multi-record write loop with no `transaction.atomic()`: rows already
updated earlier in the loop keep their new `end_date` even if a later
`id` in the same request fails and aborts the rest.
```json
{"success": true, "message": "<value of FETCH_SUCCESS_MSG>", "data": {}}
```

### Response — failure (400)
Standard envelope.

### Notes
`newdate` isn't validated as a real date or checked against the
schedule's own `start_date` before being saved — a malformed or
nonsensical `extendend_date` would be persisted as-is.

## 24. Enrollment Extend — Date Conflict Check

| Field | Value |
| --- | --- |
| **URL** | `/enrollment-modal-date-checking/` |
| **Method** | POST |
| **Auth** | No `permission_classes` set |
| **View class** | `EnrollmentModalDateChecking` (`ListAPIView`) |

### Request body
| Field | Type | Notes |
| --- | --- | --- |
| `programme_sch` | comma-separated string | `EnrollmentScheduleDate` IDs |
| `college` | value | read, not used in this view |
| `cdate` | string | date to compare against |

### Response — success (201)
```python
temp=0
for p in pgm:
  if p['end_date']>cdate:
    temp=1
  elif p['end_date']==cdate:
    temp=2
```
`temp` is a single scalar shared across every row in `pgm` — if
`programme_sch` resolves to more than one `EnrollmentScheduleDate`, only
the **last** row's comparison survives; earlier rows' results are
silently overwritten rather than combined. The comparison itself relies
on `p['end_date']` (DRF's default ISO `YYYY-MM-DD` serialization, per the
local `EnrollmentSerializer`) and `cdate` being in the same
lexicographically-sortable format — not confirmable from this file alone
whether the frontend always sends `cdate` as ISO.
```json
{"success": true, "message": "<value of FETCH_SUCCESS_MSG>", "data": {"temp": 1}}
```

### Response — failure (400)
Standard envelope.

### Notes
`college` is read from the request but never referenced anywhere in this
method.

## 25. Enrollment Schedule List

| Field | Value |
| --- | --- |
| **URL** | `/enrollment-list-view/` |
| **Method** | GET |
| **Auth** | No `permission_classes` set |
| **View class** | `EnrollmentList` (`APIView`) |

### Request body
No parameters.

### Response — success (200) — missing None-check
```python
for s in sch:
  ...
  edates=EnrollmentScheduleDate.objects.filter(enrolmnt_sch=s.id).first()
  startdate=edates.start_date.strftime("%d-%m-%Y")
```
No `None` check — any `EnrollmentSchedule` with zero associated
`EnrollmentScheduleDate` rows makes `edates` `None`, and
`edates.start_date` raises `AttributeError`, which (per the outer
`except`) fails the **entire** list for every schedule, not just the one
missing dates.
```json
{"success": true, "message": "<value of EXAM_CNTR_FETCH_SUCCESS_MSG>", "data": {"schedule": [{"sid": 1, "name": "...", "startdate": "22-08-2026", "enddate": "..."}]}}
```

### Response — failure (500)
Standard envelope, `HTTP_500_INTERNAL_SERVER_ERROR` (this endpoint and
#26 use 500 on failure rather than the 400 used almost everywhere else in
this file).

### Notes
None beyond the above.

## 26. Enrollment Schedule Detail

| Field | Value |
| --- | --- |
| **URL** | `/enrollment-schdl-view/` |
| **Method** | POST |
| **Auth** | No `permission_classes` set |
| **View class** | `EnrollmentSchdlView` (`APIView`) |

### Request body
| Field | Type | Notes |
| --- | --- | --- |
| `sid` | int | `EnrollmentSchedule` ID |

### Response — success (200) — missing None-checks, list-aliasing bug
```python
sch=EnrollmentSchedule.objects.filter(id=schid).first()
edates=EnrollmentScheduleDate.objects.filter(enrolmnt_sch=schid).first()
...
name=sch.schedule_name
startdate=edates.start_date.strftime("%d-%m-%Y")
```
Same missing-`None`-check pattern as endpoint 25, on both `sch` and
`edates` — an `sid` with no matching schedule, or a schedule with no
dates, raises `AttributeError`.

**List-reference-aliasing bug** in the per-programme college breakdown:
```python
colprg=[]
col=[]
for e in edatesall:
  prgmname=e.clg_batch_prgm.college_programm.programme.title
  ...
  call=EnrollmentScheduleDate.objects.filter(
      enrolmnt_sch=schid,
      clg_batch_prgm__college_programm__programme__id=pid,
      enrolmnt_sch__academic_year__admission_year=admnyr
  ).all()
  for c in call:
    colgname=e.clg_batch_prgm.college_programm.college_department.college.name
    colgcode=e.clg_batch_prgm.college_programm.college_department.college.code
    col.append({"colgname":colgname,"colgcode":colgcode})
  colprg.append({"prgmname":prgmname,"prgmcode":prgmcode,"col":col})
```
Two separate problems here:
1. **Wrong loop variable used.** The inner loop iterates `call` as `c`,
   presumably meaning to read each matching row's own college — but the
   body reads `e.clg_batch_prgm...` (the *outer* loop's variable) instead
   of `c.clg_batch_prgm...`. `c` is never actually used. So every college
   appended inside this inner loop is the same one belonging to the
   outer row `e`, repeated once per item in `call`, rather than each
   distinct college that `call` was queried for.
2. **`col` is never reset per programme.** `col=[]` is created once,
   before the outer loop starts, and every `colprg.append({...,
   "col":col})` appends a reference to that *same* list object — it's
   never reassigned to a fresh list inside the loop. By the time the
   response is built, every entry in `colprg` points at the identical,
   fully-accumulated `col` list (i.e. every programme's `"col"` shows
   *all* colleges seen across *every* programme processed so far, not
   just its own). Combined with bug 1, this means the college breakdown
   in the response is unreliable for any schedule with more than one
   programme.

```json
{
  "success": true,
  "message": "<value of EXAM_CNTR_FETCH_SUCCESS_MSG>",
  "data": {
    "schedule": [{"sid": 1, "name": "...", "startdate": "...", "enddate": "...", "admnyr": "..."}],
    "colprg": [
      {"prgmname": "...", "prgmcode": "...", "col": ["<same accumulated list for every entry — see bugs above>"]}
    ]
  }
}
```

### Response — failure (500)
Standard envelope, `HTTP_500_INTERNAL_SERVER_ERROR`.

### Notes
Given the aliasing bug, `col` should be reinitialized to `[]` at the top
of each `for e in edatesall:` iteration, and the inner loop should read
from `c`, not `e`.

## Notes on this source file

- **Response envelope:** consistent `format_response()` usage throughout,
  same as `c_code_gen.py` — no raw `Response()` calls found.
- **Auth:** no endpoint in this file sets `permission_classes` — see
  overview. Worth flagging specifically for the write endpoints here
  (`StudentUpload`, `EnrollmentDateEnable`, `EnrollmentModalExtend`),
  since none of them verify the caller's staff-section scope actually
  covers the `college`/`programme`/`admyear` IDs being written — the
  staff-scoping filters in this file are only ever applied to the
  *read* (list-building) endpoints.
- **Recurring unassigned-variable pattern (4 instances):** endpoints 2
  (`DistrictCollegeMapping`), 6 (`StudentView`), 19
  (`PrgmGrpClsCollegeView`), and 20 (`CollegeProgrammeView`) all build a
  variable inside a loop or conditional that isn't guaranteed to run,
  then read it unconditionally afterward. Endpoints 19 and 20 in
  particular are structurally identical copies of each other, which
  suggests this is a copy-paste pattern rather than four independent
  mistakes — worth fixing as a single sweep rather than four separate
  patches.
- **Transaction safety:** no use of `transaction.atomic()` anywhere in
  this file, despite three endpoints (`StudentUpload`,
  `EnrollmentDateEnable`, `EnrollmentModalExtend`) performing multi-
  record writes in loops.
- **Hardcoded PKs:** seen in `StudentUpload` (`AddressTypeMaster` id 1,
  `Country` id 1, `Province` id 1/2, `District` id 15) — consistent with
  the same anti-pattern already flagged elsewhere in this university
  system's codebase.
- **`template_name` typo:** present on the failure paths of endpoints
  10–14 and 21 in this file (`'...html"'` with the stray embedded quote)
  — see `00-overview.md`, "Known cross-cutting issues."

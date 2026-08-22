# cdunit_mobile

Source file: `cd_unit/cdunit_mobile.py` (267 lines). URL mapping is defined in
`cd_unit/urls.py`.

Status: 7 routed endpoints backing a mobile app's login/OTP/profile flow. This is the most
bug-dense file documented so far, including one endpoint (`CDChangePassword`) where the
success path is **unreachable dead code**, a systemic pattern of failure responses reporting
`success: true`, and an OTP-comparison that's likely a type mismatch.

## 1. CD Login

| Field | Value |
| --- | --- |
| **URL** | `api/v1/user/cd-login` |
| **Method** | POST |
| **Auth** | `permission_classes = (AllowAny,)` |
| **View class** | `CDLogin` (extends `ListAPIView`) |

### Request body

| Field | Type | Notes |
| --- | --- | --- |
| `username` | string | |
| `password` | string | |

### Response — success (201)

```json
{
  "success": true,
  "message": "<value of USER_LOGIN_SUCCESS_MSG>",
  "data": { "token": "...", "isFirstLogin": true, "image": "...", "username": "...", "mobile": "..." }
}
```

### Response — failure (still HTTP 201 — see Notes)

```json
{ "success": true, "message": "<value of USR_LOGIN_FAILURE_MSG>", "data": {} }
```

### Response — exception (400)

```json
{ "success": false, "message": "<value of BAD_GATEWAY>", "data": {}, "error_code": "<value of BAD_GATEWAY_ERROR_CODE>" }
```

### Notes

- **Failure responses report `success: true`**: both the "no Messenger-group user"
  fall-through (line 82) and the (unreachable, see below) invalid-credentials branch (line 78)
  return `format_response(True, USR_LOGIN_FAILURE_MSG, ...)` — `True`, not `False` — with HTTP
  201. Any client that checks the `success` boolean rather than parsing the message string would
  read a failed login as a success. This exact pattern repeats in `CDSendOTP` and
  `CDResetPassword` below — see "Notes on this source file".
- **Questionable authorization gate**: the code checks
  `test = User.objects.filter(groups__name=MESSENGER).first()` — whether **any** user anywhere
  in the system belongs to the `MESSENGER` group — not whether the specific user attempting to
  log in does. A commented-out line directly above it (`# if
  Group.objects.get(name="Messenger").user_set.filter(id=request.user.id).exists():`) suggests
  the original intent was a per-user check, but at this point in a login flow (pre-
  authentication, `AllowAny`) `request.user` would typically be `AnonymousUser`, so that
  original check wouldn't have worked either. As written, the gate effectively always passes as
  long as at least one Messenger-group user exists anywhere in the system, regardless of who's
  logging in.
- **Dead code — `authenticate()` result discarded**: `user = authenticate(username=username,
  password=password)` is computed but its return value is never checked (not tested for `None`)
  before being unconditionally overwritten two lines later by
  `user = serializer.validated_data['user']`. The manual `authenticate()` call does nothing.
- **Unreachable `else` branch**: `serializer.is_valid(raise_exception=True)` either returns
  `True` or raises (caught by the outer `except`) — it never returns a falsy value — so the
  `else:` at line 77 (`return format_response(True, USR_LOGIN_FAILURE_MSG, ...)`) can never
  execute.
- **Latent missing-return risk**: inside the `else:` branch for a returning user (line 62-76),
  the code is `if user: ... return ...` with no corresponding `else`. Given
  `serializer.validated_data['user']` should always be a truthy `User` instance when validation
  succeeds, this branch is not currently reachable with a falsy `user` — but if that assumption
  ever breaks, the method would fall through with no `return` statement, and DRF would raise
  `AssertionError: Expected a Response... but received None` (a 500 error) rather than the
  intended graceful response.
- Uses `status.HTTP_201_CREATED` for both success and (non-exception) failure responses on what
  is a login, not a resource-creation call — status-code/semantics mismatch.

## 2. CD Logout

| Field | Value |
| --- | --- |
| **URL** | `api/v1/user/cd-logout` |
| **Method** | GET |
| **Auth** | `permission_classes = (IsAuthenticated,)` |
| **View class** | `CDLogout` (extends `ListAPIView`) |

### Request body

None.

### Response — success (201)

```json
{ "success": true, "message": "<value of USER_LOGOUT_SUCCESS_MSG>", "data": {} }
```

### Response — failure (204)

```json
{ "success": false, "message": "<value of USR_LOGOUT_FAILURE_MSG>", "data": {} }
```

### Notes

- **HTTP 204 used for a failure response.** 204 No Content conventionally signals a *successful*
  request with no body. Returning it alongside `success: false` is a status-code/semantics
  mismatch that recurs across most of this file's exception handlers (see "Notes on this source
  file").
- This is the only endpoint in the file with a correctly-scoped `permission_classes =
  (IsAuthenticated,)`.

## 3. CD Change Password

| Field | Value |
| --- | --- |
| **URL** | `api/v1/user/cd-change-password` |
| **Method** | POST |
| **Auth** | `permission_classes = (AllowAny,)` — see Notes |
| **View class** | `CDChangePassword` (extends `ListAPIView`) |

### Request body

| Field | Type | Notes |
| --- | --- | --- |
| `old_password` | string | validated via `ChangePasswordSerializer` (external, not confirmable) |
| `new_password` | string | validated via `ChangePasswordSerializer` (external, not confirmable) |

### Response — intended success (201, see Notes — currently unreachable)

```json
{ "success": true, "message": "<value of USER_PASSWORD_UPDATE_SUCCESS_MSG>", "data": {} }
```

### Response — old password incorrect (400)

```json
{ "success": false, "message": "<value of USR_OLD_PASSWORD_FAILURE_MSG>", "data": {} }
```

### Response — exception (204)

```json
{ "success": false, "message": "<value of USR_PASSWORD_UPDATE_FAILURE_MSG>", "data": {} }
```

### Notes

- **This endpoint cannot currently succeed — the success path is unreachable dead code.** The
  full body, confirmed directly from source:
  ```python
  if serializer.is_valid(raise_exception= True):
      old_password = serializer.data.get("old_password")

  if not self.object.check_password(old_password):
      messages.error(request,USR_OLD_PASSWORD_FAILURE_MSG,extra_tags='error')
      return format_response(False,USR_OLD_PASSWORD_FAILURE_MSG,{},status_code=status.HTTP_400_BAD_REQUEST,template_name=None)
      self.object.set_password(serializer.data.get("new_password"))
      self.object.save()
      return format_response(True,USER_PASSWORD_UPDATE_SUCCESS_MSG,{},status_code=status.HTTP_201_CREATED)
  ```
  `self.object.set_password(...)`, `.save()`, and the success `return` are indented **inside**
  the `if not self.object.check_password(old_password):` block, positioned **after** an
  unconditional `return` statement on the line above them. The consequence:
  - If the old password is **wrong**, the function returns the failure response correctly (the
    `return` on the line right after the `if` executes).
  - The three lines below that `return` (`set_password`, `.save()`, and the success response)
    are unreachable — they can never execute, because they sit after an unconditional `return`
    inside the same block.
  - If the old password is **correct**, the `if not ...:` block's body is skipped entirely
    (including the unreachable code inside it), and the method falls through to the end of the
    function with **no `return` statement at all**. DRF's view dispatch expects every view
    method to return a `Response` object; returning `None` raises
    `AssertionError: Expected a Response... but received NoneType` — an unhandled 500 error, and
    this happens *outside* the method's own `try`/`except` scope (the `AssertionError` is raised
    by DRF's dispatcher after the view method has already returned), so it is not caught here.
  - Net effect: supplying the *wrong* old password returns a clean 400 error; supplying the
    *correct* old password (the success case) currently crashes with a 500 and never actually
    changes the password.
- **`permission_classes = (AllowAny,)` on a password-change endpoint.** The view relies on
  `self.request.user` (via `get_object()`) to know whose password to check/change, which implies
  it's meant to be used by an authenticated user — but `AllowAny` permits unauthenticated
  requests through. Django's `AnonymousUser.check_password()`/`set_password()` raise
  `NotImplementedError` rather than working normally, so an unauthenticated call would fail (via
  the broad `except`) rather than silently corrupt data — but the permission class still looks
  like it should very likely be `IsAuthenticated`, matching `CDLogout` above.

## 4. CD Send OTP

| Field | Value |
| --- | --- |
| **URL** | `api/v1/user/cd-send-otp` |
| **Method** | POST |
| **Auth** | No `permission_classes` set |
| **View class** | `CDSendOTP` (extends `ListAPIView`, `serializer_class = ResetPasswordEmailRequestSerializer`) |

### Request body

| Field | Type | Notes |
| --- | --- | --- |
| `emailid` | string | matched against `User.email` |

### Response — success (201)

```json
{ "success": true, "message": "<value of OTP_SENT_SUCCESS_MSG>", "data": {} }
```

### Response — failure, still HTTP 201 (see Notes)

```json
{ "success": true, "message": "<value of EMAIL_SENT_FAILURE_MSG>", "data": {} }
```

### Response — exception (204)

```json
{ "success": false, "message": "<value of OTP_SEND_FAILURE_MSG>", "data": {} }
```

### Notes

- **Serializer instantiated but never validated.** `serializer =
  self.serializer_class(data=request.data)` is built and then never used — `.is_valid()` is
  never called on it. `emailid` is read directly off `request.data` instead, so whatever
  validation `ResetPasswordEmailRequestSerializer` would have provided is entirely bypassed —
  dead object.
- **Unreachable `else` branch, same shape as `CDLogin`**: `User.objects.get(email=emailid)`
  either returns a `User` or raises `User.DoesNotExist` (caught by the outer `except`) — it
  never returns `None` — so the `if user != None:` check right after it is redundant and its
  `else:` (returning `EMAIL_SENT_FAILURE_MSG`, again with `success: true`) is unreachable.
- **Failure response reports `success: true`** — same systemic pattern as `CDLogin`.
- **Fire-and-forget email with no delivery confirmation**: `Util.send_email(data)` starts a raw
  `threading.Thread` subclass (`EmailThread`) and returns immediately; the view already responds
  "OTP sent" before the thread's `EmailMessage.send()` has actually run. Any send failure inside
  that thread is not caught by this view (uncaught exceptions in a plain Python thread don't
  propagate to the caller) — a user could be told the OTP was sent when it wasn't.
- **Global cache used for a per-user OTP, with an unused duplicate cache** — this is the
  security-sensitive pattern called out explicitly in the doc prompt. `cache_code` (module
  level, just above this class) is decorated with `@cached(cache=TTLCache(maxsize=1024,
  ttl=600))`, so the generated 4-digit OTP is memoized in a single shared, process-wide
  `cachetools.TTLCache` keyed by `email_id`, for 10 minutes. `cachetools.TTLCache` is not
  documented as thread-safe, so concurrent OTP requests across threads in a multi-threaded WSGI
  server could race. Separately, a module-level `cache = TTLCache(1024, 86400)` is defined at
  the top of the file (line 30) and **never referenced anywhere else in the file** — confirmed
  dead code; the actual caching happens through `cache_code`'s own decorator-created cache, not
  this variable.
- A 4-digit OTP (10,000 possible values) cached for 10 minutes, with no visible rate-limiting
  on the verification endpoint (`CDResetPassword`, below) and no `permission_classes` set here
  either, is a narrow but real brute-force surface — flagged as a security-sensitive pattern,
  not asserted as an exploited bug.

## 5. CD Reset Password

| Field | Value |
| --- | --- |
| **URL** | `api/v1/user/cd-reset-password` |
| **Method** | POST |
| **Auth** | No `permission_classes` set |
| **View class** | `CDResetPassword` (extends `ListAPIView`) |

### Request body

| Field | Type | Notes |
| --- | --- | --- |
| `code` | likely int (see Notes) | the OTP to verify |
| `emailid` | string | used to re-derive the cached OTP via `cache_code(emailid)` |

### Response — success (201)

```json
{ "success": true, "message": "<value of USR_PASSWORD_SEND_AND_RESET_SUCCESS_MSG>", "data": {} }
```

### Response — invalid OTP, still HTTP 201 (see Notes)

```json
{ "success": true, "message": "<value of USR_OTP_INVALID>", "data": {} }
```

### Response — exception (204)

```json
{ "success": false, "message": "<value of USR_PASSWORD_RESET_FAILURE_MSG>", "data": {} }
```

### Notes

- **Likely type mismatch in the OTP comparison**: `number = cache_code(emailid)` returns an
  `int` (from `randint(...)`). `code = request.data['code']` comes straight from the request
  body with no explicit cast. If the client sends `code` as a JSON string (e.g. `"1234"`), then
  `number == code` (`1234 == "1234"`) evaluates to `False` in Python regardless of whether the
  digits match, causing every otherwise-correct OTP to be rejected as invalid. This can't be
  fully confirmed without the calling frontend/mobile client, but it's a real, common bug shape
  worth checking against how `code` is actually sent.
- **Relies on `request.user`, which contradicts a "forgot password" flow**: `userid =
  request.user.id` is used to look up which user's password to reset. With no
  `permission_classes` restricting this endpoint, an unauthenticated caller would have
  `request.user` as `AnonymousUser`, whose `.id` is `None` — `User.objects.get(id=None)` then
  raises `DoesNotExist`, caught by the broad `except`. More importantly, a genuine "I forgot my
  password" flow implies the user is *not* logged in, yet this endpoint needs an authenticated
  session to know whose password to change — the email address is already available in
  `emailid` and could resolve the target user directly instead. Worth confirming this is the
  intended design rather than an oversight.
- **Failure response reports `success: true`** — same systemic pattern as `CDLogin`/`CDSendOTP`.
- New password is a random 8-character string emailed to the user in plaintext
  (`User.objects.make_random_password(length=8)`) — a common but non-ideal pattern (credential
  sent unencrypted over email); flagged per the prompt's security-sensitive-pattern ask, not as
  a functional bug.

## 6. CD Profile View

| Field | Value |
| --- | --- |
| **URL** | `api/v1/user/cd-profile-view` |
| **Method** | GET |
| **Auth** | No `permission_classes` set |
| **View class** | `CDProfileView` (extends `ListAPIView`) |

### Request body

None.

### Response — success (201)

```json
{
  "success": true,
  "message": "<value of CD_PROFILE_SUCCESS_MSG>",
  "data": { "collegeName": "...", "designation": "...", "userName": "...", "mobileNo": "...", "phnno": "...", "image": "..." }
}
```

### Response — failure (204)

```json
{ "success": false, "message": "<value of USR_PASSWORD_UPDATE_FAILURE_MSG>", "data": {} }
```

### Notes

- **Confirmed unassigned-variable bug — the exact pattern the doc prompt calls out.** Source:
  ```python
  user_designation = Group.objects.filter(user=user)   # assigned, never used

  if user.groups.filter( Q(name = MESSENGER )).exists():
      userprof = UserProfiles.objects.filter(user=user).first()
      serialized_data=UserProfilesSerializer(userprof)
      collegestaf = CollegeStaff.objects.filter(staff__user=user).first()
      college = collegestaf.college.name
      user_data = serialized_data.data
      userName = user_data['user']
      mobileNo = user_data['mobile_number']
      designation = user_data['designation'][0].get('name')
      phno = user_data['land_number']
      image = user_data['photo_url']

  return format_response(True,CD_PROFILE_SUCCESS_MSG,{'collegeName':college,'designation':designation,'userName':userName,"mobileNo":mobileNo,"phnno":phno,"image":image},...)
  ```
  `college`, `designation`, `userName`, `mobileNo`, `phno`, and `image` are all assigned **only**
  inside the `if user.groups.filter(...MESSENGER...).exists():` block, then referenced
  **unconditionally** in the `return` right after it — with no `else`. Any user not in the
  `MESSENGER` group hits `NameError: name 'college' is not defined` (or whichever name Python's
  evaluation order reaches first) when this view tries to build its response.
- **Constant/message mismatch on the resulting exception**: because that `NameError` is caught
  by the surrounding broad `except Exception`, the caller sees
  `USR_PASSWORD_UPDATE_FAILURE_MSG` — a password-related constant — as the error message for
  what is actually a profile-fetch failure. The handler also calls
  `messages.error(request, 'Password Reset Failed', extra_tags='error')` — a **hardcoded**
  string, also about password reset, also unrelated to this endpoint. Both read as leftover
  copy-paste from the password-reset view above it in the same file.
- `user_designation` (the `Group.objects.filter(user=user)` line) is computed and never used —
  dead code.

## 7. Route List

| Field | Value |
| --- | --- |
| **URL** | `api/v1/user/cd-route-list` |
| **Method** | POST |
| **Auth** | No `permission_classes` set |
| **View class** | `RouteList` (extends `ListAPIView`) |

### Request body

| Field | Type | Notes |
| --- | --- | --- |
| `route_id` | int | id passed to `RouteCamp.objects.filter(route_id=...)` |

### Response — success (201)

```json
{
  "success": true,
  "message": "<value of CD_PROFILE_SUCCESS_MSG>",
  "data": {
    "route_list": [ { "route_id": 0, "route_name": "..." } ],
    "camp_details": [ { "camp_id": 0, "camp_name": "..." } ]
  }
}
```

### Response — exception (500)

```json
{ "success": false, "message": "<value of BAD_GATEWAY>", "data": {}, "error_code": "<value of BAD_GATEWAY_ERROR_CODE>" }
```

### Notes

- **Missing empty-result check**: `route_data = serialized_data.data` (from
  `RouteCampSerializer(routeObj, many=True)`) is indexed unconditionally at
  `route_data[0].get('routename')`. If `route_id` doesn't match any `RouteCamp` row, `route_data`
  is an empty list and this raises `IndexError` — caught by the broad `except`, so it's handled,
  but genuinely invalid input still surfaces only as the generic `BAD_GATEWAY` message.
- **Success message reused from an unrelated endpoint**: the success response uses
  `CD_PROFILE_SUCCESS_MSG` (the same constant `CDProfileView` uses) rather than a route-
  list-specific message — likely copy-paste, though functionally harmless if the constant's
  value is a generic "success" string.
- **The one endpoint in this file with a conventionally-correct exception response** — `success:
  false` with a real error `status_code` (500), unlike the `success: true`/`204` patterns seen
  everywhere else in this file.

## Notes on this source file

- **Systemic bug: "logical" failures report `success: true`.** Across `CDLogin`, `CDSendOTP`,
  and `CDResetPassword`, every *logical* failure path (wrong credentials, email not found,
  invalid OTP) returns `format_response(True, <failure message>, ...)` — `True`, not `False` —
  distinguishing them from each other only by the message string and an unchanged HTTP 201.
  Only *exception-driven* failures in this file use `success: false`. Any client that branches on
  the `success` flag (the pattern established by every other module in this app) rather than
  string-matching the message would treat these as successful responses. This is the most
  consequential cross-cutting finding in the file — see `00-overview.md`'s cross-cutting issues
  section.
- **Secondary pattern: HTTP 204 used for exception-path failures.** `CDLogout`,
  `CDChangePassword`, `CDSendOTP`, `CDResetPassword`, and `CDProfileView` all return
  `status.HTTP_204_NO_CONTENT` alongside `success: false` in their exception handlers. 204
  conventionally signals a successful, empty-body response — pairing it with a failure body is a
  semantic mismatch that could confuse any HTTP-status-aware client or proxy. `CDLogin` (400) and
  `RouteList` (500) are the only two endpoints in this file that use a real error status code on
  their exception path.
- Only `CDLogout` has a correctly-scoped `permission_classes` (`IsAuthenticated`). Every other
  endpoint in this file is either explicitly `AllowAny` or has no `permission_classes` at all —
  whether that's covered by a project-wide default is not confirmable from this upload, but it's
  worth a deliberate review given this file handles login, password change, and password reset.
- Unlike `qp_list_detailes.py`, this file correctly defines its own `logger =
  logging.getLogger(__name__)` at module scope (line 28) — the `logger.error(...)` calls in this
  file's exception handlers are not at risk of the `NameError` flagged in that other module.

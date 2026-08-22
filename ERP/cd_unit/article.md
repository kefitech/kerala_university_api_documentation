# article

Source file: `cd_unit/article.py` (61 lines). URL mapping is defined in `cd_unit/urls.py`.

Status: 1 routed endpoint, both `GET` and `POST` on the same view. One dead-code finding (a
serialized list computed but never returned) and no visible auth restriction.

## 1. Article Request

| Field | Value |
| --- | --- |
| **URL** | `api/v1/article-request` |
| **Method** | GET, POST (both handled by the same view) |
| **Auth** | No `permission_classes`/`authentication_classes` set on the view — not confirmable whether a project-wide default applies (not in this upload) |
| **View class** | `ArticleRequest` (extends `rest_framework.generics.ListAPIView`, but overrides both `get` and `post` directly rather than using ListAPIView's built-in `list()`) |

### Request body (POST)

Body is passed straight into `ArticleRequestCollegeSerializer(data=request.data, ...)`. The
serializer is defined in `util.serializers` (not included in this upload) — fields not
confirmable.

### Response — success, GET (200)

```json
{
  "success": true,
  "message": "<value of ARTICLE_REQUEST_FETCH_SUCCESS_MSG>",
  "data": {
    "articlenames": [ /* ArticleMasterSerializers output — external, not confirmable */ ],
    "prioritytype": [ /* PriorityMasterSerializer output — external, not confirmable */ ]
  }
}
```
(Envelope shape itself — `success`/`message`/`data` keys — is inferred from `format_response`'s
call-site argument order across the app; see `00-overview.md`. Not confirmed against
`format_response`'s own source.)

### Response — success, POST (200)

```json
{ "success": true, "message": "<value of ARTICLE_REQUEST_MSG>", "data": {} }
```

### Response — failure, POST (400)

```json
{ "success": false, "message": "<value of BAD_GATEWAY>", "data": {}, "error_code": "<value of BAD_GATEWAY_ERROR_CODE>" }
```

### Notes

- **Dead code / computed-but-unused value**: `GET` builds
  `articlerequestcollegeObj = ArticleRequestCollegeSerializer(article, many=True)` from every
  `ArticleRequestCollege` row, but `articlerequestcollegeObj.data` is never included in the
  returned `data` dict — only `articlenames` (article types) and `prioritytype` are returned.
  The actual list of existing article requests is computed and then discarded:
  ```python
  articlerequestcollegeObj = ArticleRequestCollegeSerializer(article,many=True)
  ...
  return format_response(True,ARTICLE_REQUEST_FETCH_SUCCESS_MSG,data={"articlenames":articletypeserializer.data,"prioritytype":prioritytypeserializer.data},...)
  ```
  If the intent was for `GET` to return existing article requests (as the endpoint name
  suggests), this looks like a bug rather than intentional design — worth confirming with
  whoever owns this endpoint.
- **Leftover debug print**: `print(article)` on the `GET` path (line 46) — a raw queryset print
  left in from debugging, harmless but noisy in production logs.
- **Exception handling on POST is solid** for this endpoint specifically: the `try`/`except`
  wraps the whole body, logs via `logger.error(e, exc_info=True)`, and returns a distinct error
  response — no silent swallow here.
- **No auth gap confirmed, but flagged**: no `permission_classes` is set on `ArticleRequest`,
  and no ownership/authority check ties the incoming request to a specific college — anyone
  who can reach this endpoint can submit an article request or (on `GET`) read the full type/
  priority master lists. Whether this matters depends on project-wide default permissions not
  visible in this upload.
- `GET` uses `ListAPIView` only as a base class for convention — it does not call `self.list()`
  or use `self.queryset`/pagination at all, so the "List" semantics from the base class are
  unused here.

# Document API

Nota's product model is `Document > Page > Representation`: a Document is the owner, visibility,
comments, and lifecycle boundary; a Page is one Markdown content unit; HTML, Markdown, and JSON are
representations rather than separate resources. Existing `/share` routes and `documents` JSON fields
remain compatible.

Use the active Nota origin selected by `SKILL.md`. Preserve returned Document and Page URLs
exactly, and never send `ONA_API_KEY` to a different origin.

If an authenticated operation needs `X-API-KEY` and `ONA_API_KEY` is absent, complete the browser
device authorization flow in `SKILL.md` first, then resume this API operation.

When the user has not supplied a target Nota URL, use the active Nota origin; create at
`{nota_origin}/share`. Do not infer the business API origin from where the Skill was installed.
When the user supplies a complete Nota URL, preserve it rather than reconstructing it. If its
origin differs from the active Nota origin, anonymous reads may proceed, but authenticated
operations require an explicit matching `ONA_NOTA_ORIGIN` before sending `ONA_API_KEY`.

## Read

- `GET {publication_url}` returns server-rendered HTML by default.
- `GET {publication_url}?format=markdown` returns the default document as Markdown.
- `GET {publication_url}?format=json` returns Document metadata without Markdown bodies.
- `GET {publication_url}/{relative_path}?format=markdown|json` reads one named document.
- The `format` query takes precedence over `Accept`.
- Save the response `ETag`; every Page in one Document shares the Document ETag. Comments use a
  separate comments ETag and do not change this ETag.

Anonymous reads work for `share=link`. A private machine read needs the owner API key. A `404`
may mean missing, deleted, or hidden from the caller; do not claim which one without owner access.

## Create

Send `POST {nota_base}/share` with `Content-Type: application/json` and `X-API-KEY`:

```json
{
  "title": "Project Docs",
  "share": "private",
  "comments": "open",
  "default_path": "README.md",
  "documents": [
    {
      "path": "README.md",
      "title": "Overview",
      "markdown": "# Project Docs\n"
    }
  ]
}
```

Rules that change requests:

- Omit `share` unless the user explicitly asks to share; the default is `private`.
- Use `share=link` when the user asks for a link others can read.
- Omit `comments` unless the user explicitly asks for `locked` or `off`; new Documents default to
  `open`.
- `documents` is non-empty and has at most 50 items. Paths are unique, case-sensitive,
  normalized UTF-8 relative paths with no empty, `.`, `..`, backslash, or leading `/` segment.
- `default_path`, when present, matches one submitted path; otherwise the first document is used.
- Each Markdown body is at most 10 MiB, all Markdown at most 20 MiB, and the encoded JSON body at
  most 21 MiB.

On `201`, retain the complete `Location`/response URL and `ETag`. Report
`relative_image_not_uploaded` warnings; Nota does not upload relative images or attachments. The
response `documents[]` entries include the opaque Page `id` needed by Comment APIs.

## Comments

Comments are attached to a Page but coordinated under one Document. Comment reads and writes use
their own `ETag: "comments-{revision}"`; never use it as a Document `If-Match`, or use a Document
ETag for a Comment mutation.

### Read

- `GET {nota_origin}/api/documents/{document_id}/pages/{page_id}/comments` returns the current
  Page's visible comments. A link Document is anonymously readable while comments are `open` or
  `locked`; a private Document needs owner access.
- `GET {nota_origin}/api/documents/{document_id}/comments` needs the owner key and returns all
  comments, including each Comment's creation-time Page path/title snapshot and whether that Page
  is still current.
- Save the comments ETag from either list response before changing state or deleting a Comment.

Comment JSON contains the opaque Comment `id`, author display name, Markdown `body`, and UTC
`created_at`; it never contains the Seedling user ID.

### Create

Send `POST {nota_origin}/api/documents/{document_id}/pages/{page_id}/comments` with
`Content-Type: application/json`, `X-API-KEY`, and a caller-selected body only:

```json
{ "body": "This section needs an example." }
```

The trimmed UTF-8 Markdown body must be non-empty and at most 8 KiB. Images, raw HTML, attachments,
replies, and editing are unsupported. On `201`, retain the Comment `Location` and current comments
ETag. A `locked` or `off` thread rejects creation. Nota applies Redis-backed limits across service
instances; do not retry a `429` before `Retry-After`, and do not bypass a
`503 rate_limit_unavailable`.

### Change state

The owner sends `PATCH {nota_origin}/api/documents/{document_id}/comments` with `X-API-KEY`, the
current comments `If-Match`, and exactly one state:

```json
{ "state": "open" }
```

`locked` preserves visible comments but prevents creation. `off` hides comments from non-owners
without deleting them. Reopening preserves the same Comment thread.

### Permanently delete a Comment

After the required deletion confirmation, send
`DELETE {nota_origin}/api/documents/{document_id}/comments/{comment_id}` with `X-API-KEY` and the
current comments `If-Match`. A Comment Author may delete their own Comment; the Document owner may
delete any Comment. On `204`, use the returned comments ETag for later mutations. Deletion is not
recoverable.

## Replace the complete snapshot

There is no single-Page update. To update:

1. If the input may be a Page URL, GET its `?format=json` representation and take the complete
   Document root URL from the response: top-level `url` for a Document response, or
   `publication.url` for a Page response. Otherwise GET `{publication_url}?format=json`. Save the
   current ETag.
2. Read any current Markdown needed to preserve unchanged documents.
3. Build the entire desired snapshot using the create body without `share`.
4. Send `PUT {publication_url}` with `X-API-KEY` and `If-Match: {current_etag}`.

Paths omitted from the PUT disappear. On `409 version_conflict`, re-read the current Document
metadata and affected Markdown, explain the conflict, and ask whether to merge or replace. Never
retry with the new ETag without that decision. On an ambiguous timeout or `503`, read the current
ETag before deciding whether a retry is needed.

## Change sharing

Resolve a document URL through its JSON response if necessary, read the current ETag, then send
`PATCH {publication_url}` with `X-API-KEY`, `If-Match`, and one of:

```json
{ "share": "link" }
```

```json
{ "share": "private" }
```

Changing back to `link` reuses the same URL. If a leaked URL must never become valid again, the
only current remedy is permanent deletion followed by a new Document.

## Permanently delete

Explain that deletion is irreversible and obtain explicit confirmation. Resolve a document URL
through its JSON response if necessary, then read the current ETag and send
`DELETE {publication_url}` with `X-API-KEY` and `If-Match`. A `204` means reads are permanently
`404`; physical content cleanup is asynchronous. Do not retry a deletion whose result is unknown
until a follow-up read establishes the visible state.

## Errors

Nota errors use `{name, message, data}`. Preserve the status and `name` in failure reports.
Important cases are:

- `400 ambiguous_credentials`: both browser session and API key were sent; retry with only the
  credential appropriate for the operation.
- `401 unauthenticated`: credentials are absent or invalid; do not fall back to anonymous owner
  access.
- `404 not_found`: missing, deleted, or hidden.
- `409 version_conflict`: re-read the resource matching the ETag type and surface the conflict.
- `409 comments_not_open` or `comment_thread_full`: changing the request cannot create a Comment
  until the owner reopens the thread or frees capacity.
- `429 rate_limited`: wait at least the response `Retry-After`; rejected retries do not need a new
  body.
- `413 request_too_large`: reduce the Document within documented limits.
- `428 precondition_required`: obtain the current ETag and retry the intended mutation once.
- `502 content_integrity_error`: do not render or overwrite based on damaged content.
- `503 platform_unavailable`, `content_store_unavailable`, `rate_limit_unavailable`, or
  `coordination_store_unavailable`: preserve the error; for an uncertain mutation, read state
  before any retry.

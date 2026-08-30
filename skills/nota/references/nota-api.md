# Publication API

Use the active Nota origin selected by `SKILL.md`. Preserve returned publication and document URLs
exactly, and never send `ONA_API_KEY` to a different origin.

When the user has not supplied a target Nota URL, use the active Nota origin; create at
`{nota_origin}/share`. Do not infer the business API origin from where the Skill was installed.
When the user supplies a complete Nota URL, preserve it rather than reconstructing it. If its
origin differs from the active Nota origin, anonymous reads may proceed, but authenticated
operations require an explicit matching `ONA_NOTA_ORIGIN` before sending `ONA_API_KEY`.

## Read

- `GET {publication_url}` returns server-rendered HTML by default.
- `GET {publication_url}?format=markdown` returns the default document as Markdown.
- `GET {publication_url}?format=json` returns publication metadata without Markdown bodies.
- `GET {publication_url}/{relative_path}?format=markdown|json` reads one named document.
- The `format` query takes precedence over `Accept`.
- Save the response `ETag`; all documents in one publication share the publication ETag.

Anonymous reads work for `share=link`. A private machine read needs the owner API key. A `404`
may mean missing, deleted, or hidden from the caller; do not claim which one without owner access.

## Create

Send `POST {nota_base}/share` with `Content-Type: application/json` and `X-API-KEY`:

```json
{
  "title": "Project Docs",
  "share": "private",
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
- `documents` is non-empty and has at most 50 items. Paths are unique, case-sensitive,
  normalized UTF-8 relative paths with no empty, `.`, `..`, backslash, or leading `/` segment.
- `default_path`, when present, matches one submitted path; otherwise the first document is used.
- Each Markdown body is at most 10 MiB, all Markdown at most 20 MiB, and the encoded JSON body at
  most 21 MiB.

On `201`, retain the complete `Location`/response URL and `ETag`. Report
`relative_image_not_uploaded` warnings; Nota does not upload relative images or attachments.

## Replace the complete snapshot

There is no single-document update. To update:

1. If the input may be a document URL, GET its `?format=json` representation and take the complete
   publication URL from the response (`publication.url` for a document response). Otherwise GET
   `{publication_url}?format=json`. Save the current ETag.
2. Read any current Markdown needed to preserve unchanged documents.
3. Build the entire desired snapshot using the create body without `share`.
4. Send `PUT {publication_url}` with `X-API-KEY` and `If-Match: {current_etag}`.

Paths omitted from the PUT disappear. On `409 version_conflict`, re-read the current publication
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
only current remedy is permanent deletion followed by a new publication.

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
- `409 version_conflict`: re-read and surface the conflict.
- `413 request_too_large`: reduce the publication within documented limits.
- `428 precondition_required`: obtain the current ETag and retry the intended mutation once.
- `502 content_integrity_error`: do not render or overwrite based on damaged content.
- `503 platform_unavailable`, `content_store_unavailable`, or
  `coordination_store_unavailable`: preserve the error; for an uncertain mutation, read state
  before any retry.

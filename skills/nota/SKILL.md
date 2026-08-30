---
name: nota
description: Use Nota to read shared Markdown URLs and to publish, replace, share, or permanently delete Nota publications. Use when the user provides a Nota URL or asks to publish or manage Markdown with Nota.
license: MIT
metadata:
  short-description: Use Nota with an agent
  version: 0.2.0
  released-at: "2026-08-30T04:26:32Z"
---

# Nota

Use Nota as a URL-first bridge between Agent-authored Markdown and human-readable web pages.

## Origin

Use `https://nota.ona.cool` as the Nota origin by default. Only when `ONA_NOTA_ORIGIN` exists,
use its value instead. An override must be a non-empty exact HTTPS origin with no credentials,
path, query, or fragment; reject an invalid override instead of falling back to the default.

## Route the request

- For any Nota URL, or for create, replace, sharing, and deletion requests, read
  [Publication API](references/nota-api.md).
- Preserve every complete URL returned by Nota. A document URL is valid for reads but not for
  mutations. Before replacing, changing sharing, or deleting from an arbitrary Nota URL, read its
  JSON representation and use the server-returned publication URL; never strip path segments or
  reconstruct that URL.
- Nota has no publication list, attachment upload, history, or restore API. State these limits
  when they affect the request; do not invent substitutes.

## Authorization

Anonymous reads are valid only for `link` publications. Owner reads and every mutation use
`X-API-KEY`.

Use only the Agent-private `ONA_API_KEY` credential for authenticated requests, and send it only
when the request URL's origin exactly matches the active Nota origin. If it is absent, ask for an
API key immediately before the first authenticated request and store it privately as
`ONA_API_KEY`. Never place a key in a URL, project file, Skill file, command output, log, or
response.

Creating and conditionally updating content are within a user's explicit publish/update request.
Always obtain explicit confirmation immediately before permanent deletion. A version conflict,
source-origin change, or ambiguous mutation result also requires a new decision; do not silently
overwrite it.

---
name: nota
description: Use Nota to create and read shareable Markdown Documents, update or remove owned content, and work with Page comments.
license: MIT
metadata:
  short-description: Use Nota with an agent
  version: 0.4.0
  released-at: "2026-09-01T13:33:50Z"
---

# Nota

Use Nota as a URL-first bridge between Agent-authored Markdown and human-readable web pages.

## Origin

Use `https://nota.ona.cool` as the Nota origin by default. Only when `ONA_NOTA_ORIGIN` exists,
use its value instead. An override must be a non-empty exact HTTPS origin with no credentials,
path, query, or fragment; reject an invalid override instead of falling back to the default.

## Route the request

- Treat a natural request to make the relevant content shareable as a Nota create request. Choose
  and organize the visible content that serves the request; do not require the user to say Nota,
  Markdown, Document, or a fixed command. When they want other people to open the result, create
  it with `share=link` and return Nota's complete URL.
- For any Nota URL, or for Document create, replace, sharing, deletion, and Comment requests, read
  [Document API](references/nota-api.md).
- Preserve every complete URL returned by Nota. A document URL is valid for reads but not for
  mutations. Before replacing, changing sharing, or deleting from an arbitrary Nota URL, read its
  JSON representation and use the server-returned Document root URL: top-level `url` for a
  Document response, or compatibility `publication.url` for a Page response. Never strip path
  segments or reconstruct that URL.
- Nota has no Document list, attachment upload, history, or restore API. State these limits
  when they affect the request; do not invent substitutes.

## Authorization

Anonymous reads are valid only for `link` Documents. Owner reads and every mutation use
`X-API-KEY`.

Use only the Agent-private `ONA_API_KEY` credential for authenticated requests, and send it only
when the request URL's origin exactly matches the active Nota origin. Never place a key in a URL,
project file, Skill file, command output, log, or response.

If `ONA_API_KEY` is absent immediately before the first authenticated request, use the device
authorization flow below. Do not ask the user to create or paste an API key.

### Sign in with a browser

Before starting, read [Device authorization configuration](references/device-auth.md) and use its
exact public endpoints, client ID, and scope.

1. POST JSON to the authorization endpoint with the configured public `client_id` and the scope as
   a one-item JSON array.
2. Show the returned `user_code` and `verification_uri` clearly. If the host can open a browser,
   it may open `verification_uri_complete`; otherwise the user opens the URL themselves. The user
   must personally sign in or register and approve Nota. Never request or inspect their password,
   verification code, Cookie, or browser session.
3. Poll the configured token endpoint with that `client_id` and the returned `device_code`. Wait at
   least the returned `interval` between polls. Continue on
   `authorization_pending`; after `slow_down`, add 5 seconds to the minimum interval for every
   later poll in this authorization session.
4. On success, consume the response once and save its opaque `credential` privately as
   `ONA_API_KEY`, without printing it. Use the host's private secret store when available; for a
   file-backed store, create its directory with mode `0700` and file with mode `0600` before
   writing. Resume the interrupted Nota request immediately; do not ask the user to repeat their
   intent.

Device authorization is a hard precondition: until step 4 succeeds and the credential is saved, do
not construct or send an authenticated Nota request. On `access_denied`, `expired_token`, or an
unavailable service, end the current task without resuming the mutation or starting another device
session. Explain the result briefly and start a new authorization only after an explicit retry.
Anonymous reads of public Nota links never start device authorization.

Creating and conditionally updating content are within a user's explicit publish/update request.
Always obtain explicit confirmation immediately before permanent deletion. A version conflict,
source-origin change, or ambiguous mutation result also requires a new decision; do not silently
overwrite it.

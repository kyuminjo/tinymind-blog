---
title: "Keep API Credentials Out of Generated Media Downloads"
date: 2026-10-25T08:16:30.508Z
---

*A PopNow developer note, written with OpenAI Codex AI assistance. We operate PopNow. This article explains a documented integration boundary; it does not report a paid generation test or a security audit.*

A generated video can pass through two different authorization systems before it reaches your application. The generation API accepts your account credential. The storage service serving the finished file may accept a short-lived signed URL instead. Treating both requests as “an HTTP call” makes it easy to blur that boundary.

A useful design rule is to keep the credential that manages generation separate from the permission that downloads one result. That applies whether a person clicks Download or an agent collects an image, song or video after an asynchronous job.

## Two requests, two permissions

PopNow's [media documentation](https://popnow.ai/docs/media) describes the split explicitly. Read the task through the authenticated API. A successful task exposes its output assets, including signed download links. Download those files without attaching the API key; possession of the signed link supplies the temporary download permission.

These permissions have different reach. An API key may authorize account operations subject to its configured limits. A signed asset URL grants access to a particular output for a limited period. Neither belongs in an unrelated analytics event, a public issue or an error screenshot.

Keep two request paths in your application. The first talks to the documented API origin and supplies its account authorization. The second fetches an approved media URL and supplies only the headers required for that download. Avoid a general helper that adds the account's bearer header to every URL it receives.

```text
Authenticated API client
  -> task status / download-link request
  -> signed output URL

Separate media downloader
  -> approved signed URL
  -> local file or application storage
```

This is an architecture sketch, not a complete client implementation. Do not paste a real key or a live signed URL into documentation to make the example look runnable.

## A signed URL is temporary access

A signed link often looks like an ordinary HTTPS URL with a long query string. That appearance does not make it public information. Someone holding a valid link may be able to retrieve the file while the permission remains valid.

Record the provider's task identifier and your own asset identifier as durable state. Treat the download URL as temporary data. When access expires, use the provider's documented authenticated flow to obtain a fresh link, provided the output is still retained. Adding your API bearer token to an expired storage request is not that refresh flow.

In PopNow's contract, output retention and link expiry are separate. Read the returned expiry and retention fields rather than assuming that a working link is permanent. Copy an output you need to retain into storage you control, under your application's retention and access policy.

## Keep diagnostics useful without publishing access

A failed download still needs an observable trail. Store the logical operation ID, provider task ID, response status, attempt number and a sanitized error category. Those are more useful for recovery than dumping every request header and complete URL into a log.

Redact account authorization and signed query parameters before sending diagnostics to error monitoring. Do the same for exported support bundles. A carefully scoped API key does not compensate for distributing a signed private output link to a much broader audience.

If your backend follows user-supplied or provider-returned URLs, give that downloader its own destination and redirect checks. HTTPS by itself is not a complete server-side request policy. The [OWASP SSRF Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html) covers the additional validation concerns. A downloader with no API bearer header still needs protection against inappropriate network destinations.

## Verify the boundary with a stub

You can test the credential boundary before purchasing a generation. Have a stub API return an example output URL. Assert that the status request contains the intended API authorization and that the subsequent media request does not contain it. Also exercise an expired link, a rejected destination, a redirect, and a log capture.

Those tests establish what your application sends and records. They do not prove a provider's uptime, rendering quality, access controls or output retention behavior. A real integration check needs current documentation and an authorized test budget.

PopNow's public [agent-skill helper source](https://github.com/popnow-ai/popnow-skills/tree/main/skills/popnow-video) is another concrete reference: API requests and file downloads are separate operations. The [API docs](https://popnow.ai/docs) and [MCP setup guide](https://popnow.ai/mcp) describe the supported entry points.

The practical boundary is small: use the account key for the API that owns it, use the signed permission for the file it names, and keep both out of public diagnostics.

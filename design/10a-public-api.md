# 10.1 Our public API

**Priority:** P0 — build it into the architecture from the start ([why](10d-model-requirements.md#why-section-10-is-p0))
**Siblings:** [10.2 import](10b-import.md) · [10.3 export](10c-export.md) · [10.4 model requirements](10d-model-requirements.md) · [10.5 legal](10e-legal-sources.md)

- [ ] 10.1.1 Who is the API for: third-party clients and bots / a future mobile app / the user's own scripts ("dump my collection with a script") / internal use by our own frontend only.
- [ ] 10.1.2 Style: REST / GraphQL (align with [11.5](11-stack.md)). A public API severely limits your freedom to change internals — are we ready?
- [ ] 10.1.3 Versioning (`/api/v1`, a header) and backwards-compatibility policy: what counts as a breaking change, how we deprecate.
- [ ] 10.1.4 Authentication: personal access token / OAuth2 for third-party apps / session only. Do we need scopes (`read:catalog`, `read:collection`, `write:collection`, `write:catalog`)?
- [ ] 10.1.5 What is available anonymously (the catalogue) and what requires a token (collection, wantlist, messages). Collection privacy ([3.7](03-collection.md)) must be enforced in the API too — it is not a "UI feature".
- [ ] 10.1.6 Pagination (cursor vs offset), maximum `per_page`, request limits per token/IP, throttling of heavy queries (see [9.4](09-nfr.md), [14.5](14-security.md)).
- [ ] 10.1.7 Do we allow **writing to the catalogue** via the API? Then those edits must go through the same moderation and versioning model as edits from the UI ([section 4](04-editing.md)) — and it is one more vandalism vector.
- [ ] 10.1.8 Error format and codes, caching (ETag / If-None-Match), conditional requests.
- [ ] 10.1.9 Documentation: OpenAPI (schema-first or generated from code), a sandbox, client generation.
- [ ] 10.1.10 Webhooks / subscribing to catalogue changes — will we ever need them (this affects what the change log must look like, [4.2](04-editing.md)).
- [ ] 10.1.11 Terms of use for data accessed via the API: attribution, restrictions on bulk downloading, key on request or freely available (see [13.1](13-legal.md)).
- [ ] 10.1.12 Does the frontend call the same public API (dogfooding), or do we have two separate layers? This decision heavily affects [11.1/11.5](11-stack.md).

## Working notes

_(none yet)_

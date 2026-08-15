# 10.1 Our public API

**Priority:** P0 — build it into the architecture from the start ([why](10d-model-requirements.md#why-section-10-is-p0))
**Siblings:** [10.2 import](10b-import.md) · [10.3 export](10c-export.md) · [10.4 model requirements](10d-model-requirements.md) · [10.5 legal](10e-legal-sources.md)

**What this section is not free to decide.** [1.9](01-product.md) put the API out of the MVP and
[11.5](11-stack.md) fixed its style as REST *if* it exists, while explicitly leaving "whether" to
[10.1.1](10a-public-api.md) here. [11.5](11-stack.md) also settled the shape of the answer in
advance: the reuse boundary is the service layer rather than HTTP, so an API is a second adapter
over code the UI already exercises — which is what makes it cheap to add later and worthless to add
now. The section is P0 because of that structural claim, not because an API is planned.

- [x] 10.1.1 Who is the API for: third-party clients and bots / a future mobile app / the user's own scripts ("dump my collection with a script") / internal use by our own frontend only.
      **Decision: for nobody, in any planned stage. The API is not built — and the one audience with
      a real need is already served by [10.3](10c-export.md).**
      The item lists four candidates and it is worth taking each seriously, because "no API" is only
      an honest answer if the audiences were actually examined.
      **Third-party clients and bots — none, and this is the argument that decides it.** Not merely
      "zero today at tens of users" ([1.4](01-product.md)), which would be an argument about timing.
      The structural version: an API's value is proportional to what its data can do that nothing
      else's can, and our catalogue is **thin by construction** ([1.5](01-product.md)) — a subset of
      what a developer can already get from Discogs or MusicBrainz, minus their tracklists, credits
      and coverage. There is nothing here a third party would build against, and shipping an API
      would not change that.
      **A future mobile app — excluded upstream.** [7.6](07-search-ux.md) declined a native app and
      a PWA; the answer to mobile is that the pages work on a phone.
      **The user's own scripts — the only real one, and [10.3](10c-export.md) is its answer.** "Dump
      my collection with a script" is precisely what export already does: one request, everything the
      owner holds, in lossless JSON ([10.3.2](10c-export.md)), synchronous
      ([10.3.3](10c-export.md)). It needs no pagination, no versioned resource shapes and no client
      library.
      **What is genuinely missing for that user is authentication, not an API**, and naming it is the
      useful part of this item: [11.6](11-stack.md) authenticates with a session cookie, so a script
      must drive a login form and keep a cookie jar. **If anything ever ships for this audience it is
      a personal access token accepted on [10.3](10c-export.md)'s export URL — roughly a day's work
      — and not an API.** That is the minimum viable version of everything below.
      **Internal use by our own frontend — settled at [11.5](11-stack.md)**: pages render from
      services directly and there is no JSON hop to consume. See
      [10.1.12](10a-public-api.md).
      **On reopening.** Not on a feeling that a portfolio piece ought to have an API, but on evidence
      of the same shape [11.2](11-stack.md) demands for the stack: a named person wanting a named
      thing that export cannot give them. Everything the API would need already exists for other
      reasons (see this section's working notes), so the option costs nothing to keep and nothing is
      reserved to keep it.
- [x] 10.1.2 Style: REST / GraphQL (align with [11.5](11-stack.md)). A public API severely limits your freedom to change internals — are we ready?
      **Decision: REST, as [11.5](11-stack.md) fixed — and the second question is the important one,
      because the answer is no, and that is a third reason not to ship one.**
      **REST** is not re-argued here: [11.5](11-stack.md) excluded tRPC (it presumes a TypeScript
      client that [11.3](11-stack.md) does not have) and GraphQL (a resolver layer and an N+1 problem
      bought so that clients can shape responses, at ~12,000 releases and zero clients).
      **"Are we ready to limit our freedom to change internals?" — no, and the timing argument is
      specific rather than general.** A public API freezes the *shape of the entities*, and our
      entities are the part of the design that is still moving: the catalogue is thin by construction
      ([1.5](01-product.md)) and fills by hand ([section 4](04-editing.md)), so the fields that
      matter, the vocabularies ([2.3.2](02-catalogue-model.md) and the rest of the curated-set
      invariant) and the format structure ([2.4.2](02-catalogue-model.md)) will all learn things from
      the first real year of use. Publishing a resource representation before that is committing to a
      guess.
      **The one public contract we do accept is the export file** ([10.3.2](10c-export.md)), and it
      was accepted knowingly because it buys [1.3](01-product.md)'s "your data is never locked in".
      An API buys nothing comparable while [10.1.1](10a-public-api.md) has no audience.
- [x] 10.1.3 Versioning (`/api/v1`, a header) and backwards-compatibility policy: what counts as a breaking change, how we deprecate.
      **Decision: the version in the path (`/api/v1`), and one compatibility rule shared with the
      export file rather than a second one invented here.**
      **Path, not header.** A version in a header is invisible in a browser, awkward in `curl`, and
      easy to omit accidentally — three costs against an aesthetic argument about resource identity.
      `/api/v1/...` is readable, pasteable and greppable in a log line ([9.5](09-nfr.md) logs the
      path).
      **Compatible: adding a field, adding an endpoint, adding an optional parameter, relaxing a
      validation. Breaking: removing or renaming a field, changing a type or its meaning, tightening
      a validation, changing a default sort or page size, or changing an error code's meaning.**
      This is deliberately the same sentence as [10.3.2](10c-export.md)'s file contract — appending
      is compatible, renaming and reordering are not — because **one compatibility rule for files and
      API is a rule that gets remembered**, and two would mean two habits and one of them wrong.
      JSON export additionally carries `format_version`; the API carries the path.
      **Deprecation, sized for the population it serves:** `v2` is served alongside `v1`, never as a
      mutation of it; `v1` responses gain `Deprecation` and `Sunset` headers; and since every token
      is attached to an account ([10.1.4](10a-public-api.md)) we **email the holders**, which is
      possible here and is not at most services. Six months, or longer if that is nobody's
      inconvenience.
- [x] 10.1.4 Authentication: personal access token / OAuth2 for third-party apps / session only. Do we need scopes (`read:catalog`, `read:collection`, `write:collection`, `write:catalog`)?
      **Decision: personal access tokens, three scopes, and no OAuth2 at any stage.**
      **Tokens** are 32 bytes from the OS CSPRNG, **stored hashed exactly as sessions are**
      ([14.2](14-security.md)), shown once at creation, named by the user, listed and revocable on
      the profile page, with a `last_used_at` so a forgotten one is visible. They carry the creating
      user's identity and role ([4.7](04-editing.md)) — a token is never more privileged than the
      person who made it, and a moderator's token can be scoped down but never up.
      **OAuth2 is refused, and for the same reason [6.1](06-accounts.md) refused OAuth login turned
      around.** As an OAuth *server* it is an authorisation-code flow, a consent screen, a client
      registry, redirect-URI validation and refresh-token rotation — built so that *third-party
      applications* can act for a user. [10.1.1](10a-public-api.md) established there are no
      third-party applications. A personal token is what the actual audience — a person writing their
      own script — would want anyway.
      **Scopes: `read`, `write:collection`, `read:messages`. Chosen per token, default `read` only.**
      Deliberately not the item's list. `read:catalog` does not exist because the catalogue is public
      and needs no token at all ([10.1.5](10a-public-api.md)); `write:catalog` does not exist because
      [10.1.7](10a-public-api.md) refuses catalogue writes outright, and an unimplementable scope in
      a picker is worse than an absent one. `read:messages` is separate from `read` because
      [14.1](14-security.md) ranks message bodies as the most sensitive thing in the system, and a
      script that dumps a collection should not be able to read a mailbox.
      **Session cookies are not accepted on `/api/` routes, and tokens are not accepted anywhere
      else.** Two clean surfaces: it keeps [14.3](14-security.md)'s CSRF story confined to the form
      helper (a token-authenticated request is not ambient-authority and needs no CSRF token), and it
      means no browser can be tricked into making an authenticated API call on a user's behalf.
- [x] 10.1.5 What is available anonymously (the catalogue) and what requires a token (collection, wantlist, messages). Collection privacy ([3.7](03-collection.md)) must be enforced in the API too — it is not a "UI feature".
      **Decision: anonymous sees exactly what a logged-out browser sees, and privacy is not enforced
      "in the API too" — it is enforced once, below both.**
      **Anonymous:** the public catalogue — release, master, artist, label, their credits and
      vocabularies, and search ([7.1](07-search-ux.md)) — plus any collection or wantlist whose owner
      has made it public ([3.7](03-collection.md)). No token, and no key required to read the
      catalogue at all; [10.1.11](10a-public-api.md) explains why gating CC0 data behind a signup
      would be theatre.
      **Token required:** one's own collection and wantlist including the owner-only fields
      ([3.7](03-collection.md): note, purchase date and place), the wantlist of a private account,
      messages ([5.x](05-messaging.md)), settings ([6.6](06-accounts.md)), imports
      ([10.2](10b-import.md)) and export ([10.3.1](10c-export.md), owner-only).
      **The item's warning is real and is already answered structurally.** Privacy is not a rule the
      API must remember to apply: [14.3](14-security.md) puts authorisation in the `WHERE` clause of
      every user-scoped query, inside the service layer ([10.4.4](10d-model-requirements.md)), and
      the API is an adapter that supplies an `Actor` to those same functions
      ([11.5](11-stack.md)). **The API cannot see a private collection because the service cannot**,
      given that actor. There is no second privacy implementation to keep in step, which is exactly
      what [3.7](03-collection.md) and [11.5](11-stack.md) each demanded in advance.
      **One thing that does not carry over: [3.7](03-collection.md)'s share link.** It is a
      capability minted for a browser, and it stays there — the API authenticates with tokens.
      Letting a share token authenticate an API call would give the capability a second surface, a
      second leak path and a second revocation story for no gain.
- [x] 10.1.6 Pagination (cursor vs offset), maximum `per_page`, request limits per token/IP, throttling of heavy queries (see [9.4](09-nfr.md), [14.5](14-security.md)).
      **Decision: cursor pagination keyed on the entity id, `per_page` 50 by default and 200 at most,
      and the limits are [14.5](14-security.md)'s table with no new numbers invented here.**
      **Cursor, and [10.4.6](10d-model-requirements.md) is what makes it free.** Ids are monotonic
      `bigint`s, so a cursor is `(sort key, id)` encoded opaquely and the query is a plain
      keyset seek on an index we have anyway. Offset was the alternative and it is wrong for the case
      that matters: paging a 10,000-item collection while its owner edits it repeats and skips rows,
      and offset over [7.3](07-search-ux.md)'s filtered listings makes the database count rows it
      then discards. Every list response carries `next` or nothing; there is **no total count** on
      cursor-paged collections, which is honest rather than a limitation — [7.3](07-search-ux.md)'s
      facet counts are a separate, deliberately live query.
      **Limits: [14.5](14-security.md)'s table, keyed on the token's *user*, not on the token and not
      on the IP.** Minting a second token must not double an allowance. This is the promise
      [9.4](09-nfr.md) made — that a future API inherits the numbers rather than reimplementing them
      — and it holds automatically because the counters are checked in the service layer
      ([10.4.4](10d-model-requirements.md)), below HTTP.
      **Heavy queries** — [7.3](07-search-ux.md)'s search with facet counts and
      [3.8](03-collection.md)'s statistics, both named in [9.1](09-nfr.md)'s budget table — take
      their own buckets from the same table, and anonymous callers fall back to
      [9.4](09-nfr.md)'s per-IP proxy cap. **Exceeding a limit is a 429 with `Retry-After` and a
      `problem+json` body** ([10.1.8](10a-public-api.md)): never a silent drop, never a shadowban,
      per [14.5](14-security.md).
- [x] 10.1.7 Do we allow **writing to the catalogue** via the API? Then those edits must go through the same moderation and versioning model as edits from the UI ([section 4](04-editing.md)) — and it is one more vandalism vector.
      **Decision: no. The API is read-only over the catalogue, permanently, and there is no scope
      that would grant it.**
      **Note first that the item's own condition is already satisfied and is therefore not the
      objection.** Catalogue writes through the API would automatically get [4.2](04-editing.md)'s
      revisions, [4.7](04-editing.md)'s roles, [6.1](06-accounts.md)'s verified-email barrier and
      [14.5](14-security.md)'s limits, because [10.4.4](10d-model-requirements.md) puts all four
      inside the service function rather than in the handler. There is no "same model" to
      re-implement.
      **The objection is the second half of the item: it is a vandalism vector with nobody on the
      other end of it.** [4.9](04-editing.md) declined captcha and honeypots on the strength of the
      verified-email wall *and* the fact that an edit is a human at a form; a write token turns an
      edit into a loop, and it hands a compromised account
      ([14.1](14-security.md) ranks catalogue integrity last precisely because a human vandal is a
      chore) a machine-speed version of the same. Against that: zero clients
      ([10.1.1](10a-public-api.md)) asking to write.
      **Collection writes are a different question and are allowed** — `write:collection`
      ([10.1.4](10a-public-api.md)) touches only the caller's own rows, which are the ones
      [3.10](03-collection.md) already lets them delete outright, and bulk-tagging one's own
      collection from a script is the plausible use.
      **The route back in, if it is ever wanted:** catalogue writes restricted to `moderator`
      ([4.7](04-editing.md)), which makes the vector a person we already trust with merge.
      Not built, and not reserved.
- [x] 10.1.8 Error format and codes, caching (ETag / If-None-Match), conditional requests.
      **Decision: `application/problem+json` (RFC 9457) over the service layer's own error type, and
      `ETag` from the newest revision id on public catalogue resources only.**
      **Errors are not a new taxonomy.** The service layer already returns typed failures for the UI
      to render ([11.9](11-stack.md): decode failures are values, not exceptions); the API renders
      the same values as `problem+json` with `type`, `title`, `status`, `detail` and a stable
      machine-readable `code`. A second error vocabulary invented at the HTTP edge would drift from
      the one the pages show, and the two would disagree about the same failure.
      Validation failures carry a per-field list, because a client that cannot say *which* field was
      wrong forces its user to guess.
      **`ETag` is [10.4.7](10d-model-requirements.md)'s freebie**: the newest `revision.id` for the
      entity is a strong validator that already exists, changes on every mutation including merge and
      delete, and needs no new state. `If-None-Match` → `304`. Applied to single public catalogue
      resources only.
      **Not cached, deliberately:** every user-scoped response is `Cache-Control: private, no-store`
      — a collection listing is [14.1](14-security.md)'s protected data and must not sit in an
      intermediary. Lists and search results carry no validator (their facets and membership change
      for reasons no single revision id captures). **No `Last-Modified`** — a second validator for
      the same question, with worse resolution. **No CDN and no shared cache anywhere**
      ([1.4](01-product.md) vetoes it).
- [x] 10.1.9 Documentation: OpenAPI (schema-first or generated from code), a sandbox, client generation.
      **Decision: OpenAPI generated from the Servant API type, served as a file and as one static
      page. No sandbox, no generated clients.**
      **Generated, not schema-first**, and [11.2](11-stack.md) already bought this: the API-as-a-type
      is most of why Servant was chosen over Scotty, and `servant-openapi3` turns it into a document
      that cannot describe an endpoint the code does not have. Schema-first would put a second source
      of truth beside the type — the same objection [11.4](11-stack.md) made to ORM-shaped layers
      keeping a schema beside the migrations, and [11.9](11-stack.md) made to a hand-maintained
      shared schema.
      Served at `/api/v1/openapi.json`, plus one page rendering it. **The rendering must not pull a
      script from a CDN** — [13.4](13-legal.md)'s invariant and [14.3](14-security.md)'s CSP both
      forbid it — so it is either a vendored viewer ([11.3](11-stack.md)'s one-file rule) or, more
      likely, plain generated HTML.
      **No sandbox.** It is a second environment with its own data, and [12.4](12-infrastructure.md)
      declined staging for the same reason; the read-only endpoints are their own sandbox against
      real public data.
      **No generated client libraries.** Publishing one means versioning and maintaining it in a
      language we do not otherwise use, for the population [10.1.1](10a-public-api.md) established.
      The OpenAPI document is there for anyone who wants to generate their own.
- [x] 10.1.10 Webhooks / subscribing to catalogue changes — will we ever need them (this affects what the change log must look like, [4.2](04-editing.md)).
      **Decision: never. Webhooks are refused outright, and the reason is not effort but
      [14.3](14-security.md).**
      **A webhook is an outbound HTTP request to a URL a user supplied.** That is the exact code path
      [14.3](14-security.md) forbids by name — *no code path may take a URL from a user and fetch
      it* — because [1.5](01-product.md) had already deleted SSRF as a category and
      [14.4](14-security.md) refused link previews on the same ground. Building a delivery system
      would reintroduce, deliberately, the one attack surface this design gets for free.
      Everything else is merely the second reason: a delivery queue, retries with backoff, a
      dead-letter path, request signing, and endpoints that go stale silently — an availability
      commitment ([9.2](09-nfr.md) declined to make one) owed to subscribers who do not exist.
      **The answer for a consumer that ever exists is polling
      [10.4.7](10d-model-requirements.md)'s revision cursor**, which costs us one indexed query and
      puts the retry problem on the side that can see it.
      **This closes the item's own worry: [4.2](04-editing.md)'s change log needs nothing added for
      delivery.** [10.4.7](10d-model-requirements.md) already fixed the two properties that make a
      pull feed possible, and neither was bought for this.
- [x] 10.1.11 Terms of use for data accessed via the API: attribution, restrictions on bulk downloading, key on request or freely available (see [13.1](13-legal.md)).
      **Decision: we cannot restrict the data and do not pretend to — the terms govern the service.
      Keys are free and self-serve; attribution is requested, not required.**
      **[13.1](13-legal.md) settled the data half and it leaves little room:** the catalogue is
      **CC0**. There is no attribution requirement we could impose without contradicting our own
      licence, no non-commercial clause and no share-alike — [13.1](13-legal.md) rejected CC BY-SA
      explicitly. So the API terms **request** attribution as a courtesy and say so in those words.
      **What the terms do govern is access to the service**, which is ours to condition:
      [14.5](14-security.md)'s rate limits, a `User-Agent` identifying the client and a contact
      address, no reselling of access, and — the one that matters — **no republication of anything
      user-scoped**. A public collection is readable ([3.7](03-collection.md)) but it is a person's
      data, not CC0 catalogue data, and [13.1](13-legal.md) excludes collections, wantlists, notes,
      tags, messages and profiles absolutely.
      **Images are excluded too**, and this is the trap worth naming: an API response may carry image
      *keys and URLs* ([8.2](08-media.md)), and those bytes are **not** CC0 — we never held rights in
      them, which is why [8.4](08-media.md) removes rather than argues. Same rule as
      [10.3.4](10c-export.md)'s dump: keys, never bytes, never a licence claim.
      **Bulk downloading is not forbidden by the terms** — it cannot be, for CC0 data — and is simply
      rate-limited like everything else. **The honest answer to a bulk consumer is
      [10.3.4](10c-export.md)'s dump**, which is a scheduling decision rather than a licensing one
      now, and a far better experience than crawling [1.4](01-product.md)'s box.
      **Keys are free, self-serve from the profile page, with no application form and no review.** A
      key on request is a queue for one person to drain ([4.3](04-editing.md)'s objection to every
      queue) that gates data we have given away.
- [x] 10.1.12 Does the frontend call the same public API (dogfooding), or do we have two separate layers? This decision heavily affects [11.1/11.5](11-stack.md).
      **Decision: it does not, and [11.5](11-stack.md) decided this — the frontend has nothing to
      dogfood.**
      Pages render from the service layer directly ([10.4.4](10d-model-requirements.md)); there is no
      JSON hop in the middle and no API in the MVP to consume if there were. Building one so that our
      own server could call it over HTTP would add a serialisation round trip and a versioned
      contract between two halves of one process ([11.1](11-stack.md)).
      **The protection the question is reaching for is real, and it is obtained lower down.**
      Dogfooding exists to stop an API rotting into something nobody has run, and to stop rules
      drifting between the two surfaces. Here the shared, exercised thing is the **service layer** —
      used by every page, by the job runner and by [11.10](11-stack.md)'s level-2 tests — so any API
      is a thin adapter over code that is executed constantly, and [4.7](04-editing.md)'s roles,
      [6.1](06-accounts.md)'s barrier and [3.7](03-collection.md)'s privacy hold in both by
      construction rather than by discipline.
      **What is genuinely given up, stated rather than glossed:** the adapter itself — routing,
      serialisation, pagination, error mapping — is covered only by its own tests, since no user
      traffic crosses it. That is a real cost of not dogfooding and it is accepted, because it is
      bounded to the thinnest layer and because [10.1.1](10a-public-api.md) means there is currently
      no such layer at all.

## Working notes

- **2026-08-15 — Section closed. All 12 items decided, and the headline is that the API is not
  built.** [10.1.1](10a-public-api.md) is the decision; the other eleven fix the shape it would take
  so that the question never has to be reopened from scratch, and so that
  [15.5](15-roadmap.md) has something to schedule against. Three of them are permanent refusals
  rather than deferrals: **no OAuth2** ([10.1.4](10a-public-api.md)), **no catalogue writes**
  ([10.1.7](10a-public-api.md)) and **no webhooks, ever** ([10.1.10](10a-public-api.md)).
- **2026-08-15 — Nothing is reserved, because everything is already there.** This is the point of
  [section 10.4](10d-model-requirements.md) and it is worth having in one list — an API needs:
  the service layer with an explicit actor ([10.4.4](10d-model-requirements.md)); authorisation in
  the query and rate limits below HTTP ([14.3](14-security.md), [14.5](14-security.md)); stable
  public identifiers ([10.4.6](10d-model-requirements.md)); a monotonic revision id for `ETag` and
  for polling ([10.4.7](10d-model-requirements.md)); and a generated OpenAPI document
  ([11.2](11-stack.md), [11.5](11-stack.md)). Every one of those exists for its own reason. **No
  schema change, no migration and no new architecture** is needed the day this is reopened — only
  the adapter, the token table and its profile page.
- **2026-08-15 — The realistic first step is not this section.** If scripting one's own collection
  ever comes up, ship [10.1.4](10a-public-api.md)'s personal access token accepted on
  [10.3](10c-export.md)'s export URL — a day's work, no versioning commitment, no resource shapes
  frozen ([10.1.2](10a-public-api.md)'s worry), and it satisfies the only audience
  [10.1.1](10a-public-api.md) could find. Treat that as the actual roadmap item and the API as the
  thing it postpones.
- **2026-08-15 — What was declined, in one place.** GraphQL and tRPC (at [11.5](11-stack.md));
  header-based versioning; OAuth2 as an authorisation server; `write:catalog` as a scope; session
  authentication on API routes and token authentication on browser routes; share-token
  authentication ([3.7](03-collection.md) stays a browser capability); offset pagination; total
  counts on cursor-paged lists; `Last-Modified`; a sandbox environment; generated client libraries;
  webhooks; and any attribution or non-commercial requirement on the data, which
  [13.1](13-legal.md)'s CC0 forbids us from imposing.

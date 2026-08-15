# Cross-cutting design notes

Only what belongs to **no single section** lives here. A note about one section goes in that
section file's `Working notes`.

Keep entries short and dated (`YYYY-MM-DD`). When a note hardens into a decision, move it into the
relevant item as a `**Decision:**` block and delete it from here.

## Agreed invariants

Rules that hold across the whole design and must not be quietly broken by a later decision.
Each one names the item it was decided in.

- **2026-08-14 — Two channels in, and no others.** Data enters the system only via (1) a file the
  user uploads or (2) a person typing into our UI. The server never calls an external music
  database, downloads a dump, or scrapes. Any design that needs "we can just fetch it from
  Discogs/MusicBrainz" is invalid. Decided in [1.5](01-product.md).
  **This constrains provenance, not the schema** — no field is forbidden because an external source
  would have been the convenient way to fill it.
- **2026-08-14 — The catalogue is thin by construction.** Entries start with only what a user's
  export carried and are enriched by hand afterwards. Nothing may assume a fully populated release
  (no required tracklists, no required images, no required credits) — but equally, nothing forbids
  those fields existing and being filled in by hand. Follows from [1.5](01-product.md).
- **2026-08-14 — One small box, and a veto list to keep it that way.** No search engine, no Redis,
  no replicas, no CDN, no orchestration — see [1.4](01-product.md) for the full list and the row
  estimates behind it. Cite that item to reject scale-driven proposals in
  [section 9](09-nfr.md) and [section 12](12-infrastructure.md). Exceptions that still get real
  engineering: image storage ([section 8](08-media.md)), import as a background job
  ([10.4.5](10d-model-requirements.md)), and tested backups.
- **2026-08-14 — Nothing order-shaped in the model.** No price, availability or seller state on any
  entity, at any stage. Decided in [1.7](01-product.md).
- **2026-08-14 — UI language and catalogue data are separate concerns.** The interface is English
  only ([1.8](01-product.md)) with no i18n machinery. Catalogue data is independent of that: names
  and titles are recorded as they appear on the object, in any script, and are never translated or
  transliterated by the application. The multi-script problem (`Кино` / `Kino`) is therefore a
  catalogue modelling question for [section 2](02-catalogue-model.md), unaffected by the UI language
  decision.
- **2026-08-14 — Physical media of any format.** Vinyl, CD, cassette and others are equal citizens;
  no format-specific field may be promoted onto the release itself. Decided in
  [1.2](01-product.md).
- **2026-08-15 — Every release belongs to a master.** `release.master_id` is `NOT NULL`, with no
  orphan case anywhere in the model, the UI, the API or the export. Decided in
  [2.1.2](02-catalogue-model.md), against the opening recommendation and with the cost accepted.
  **Consequence that reaches other sections:** the importer mints a master per row, so duplicate
  masters are the normal state and **master merge moves into early scope** alongside release merge —
  [section 4](04-editing.md) cannot treat merging as a late moderation feature.
- **2026-08-15 — Object facts on the release, work facts on the master.** Where a fact could sit on
  either, it goes on the release: the object is ground truth, the master is an interpretation. Decided
  in [2.1.1](02-catalogue-model.md); it is what puts the bootleg flag on the release
  ([2.4.7](02-catalogue-model.md)) and genres on the master ([2.3.2](02-catalogue-model.md)).
- **2026-08-15 — No uniqueness constraint on releases, ever.** Two pressings may differ only in a
  runout etching, and a constraint would make the importer silently discard a user's row. Duplicates
  are handled by advisory detection at create time and by merge afterwards — never by rejection.
  Decided in [2.4.9](02-catalogue-model.md).
- **2026-08-15 — Nothing a user's collection points at is ever hard-deleted.** Merging leaves a
  `merged_into` pointer, the old ID keeps resolving, and collection items follow. Binds
  [section 3](03-collection.md) and [section 4](04-editing.md). Decided in
  [2.4.9](02-catalogue-model.md).
- **2026-08-15 — The application never translates or transliterates a name, and one artist holds
  many names.** `Кино` and `Kino` are one entity with one primary name and variants beside it, all in
  one search index — not two artists, not a translation string. Decided in
  [2.2.1](02-catalogue-model.md), closing the multi-script question [1.8](01-product.md) parked on
  section 2.
- **2026-08-15 — Everything a catalogue entity credits is an artist entity.** No raw-name credits
  anywhere — not in the billing ([2.3.3](02-catalogue-model.md)), not in role credits
  ([2.4.5](02-catalogue-model.md)). A raw name would be a second class of artist that no page links
  to and no search finds. Creating a stub is one click and [2.2.6](02-catalogue-model.md) requires
  nothing of an artist but a name.
- **2026-08-15 — Merge never copies a field, and delete is never hard.** Merging writes `merged_into`
  on the loser and repoints every reference; no value is moved from loser to winner, so a wrong merge
  destroys nothing and unmerge is the reverse operation. Deletion is `deleted_at` plus a tombstone
  that still resolves. Both are moderator-only. Decided in [4.4](04-editing.md) and
  [4.5](04-editing.md), closing what [2.4.9](02-catalogue-model.md) left to section 4.
  **Consequence:** salvaging a better value from the loser is an ordinary edit performed *before* the
  merge, and every UI that offers merge must say so.
- **2026-08-15 — Edit history is written from day one and shown to nobody.** Every mutation writes an
  aggregate snapshot; the diff viewer, revert flow and change feed are deferred until something needs
  them. Decided in [4.2](04-editing.md). The rule this generalises: **store what cannot be
  reconstructed, defer what can be built later from it.** Applies to any "do we need X history"
  question in a later section.
- **2026-08-15 — Rights come from roles, never from history.** Three roles (`user`, `moderator`,
  `admin`), enforced in the service layer so [10.1](10a-public-api.md) obeys the same rules, and
  nothing anywhere is gated on a contribution count or a reputation score. Decided in
  [4.7](04-editing.md) and [4.8](04-editing.md).
- **2026-08-15 — A collection item holds no copy of catalogue data.** No denormalised title, artist,
  year or format on `collection_item` — it is a foreign key plus the facts about *this copy*.
  Decided in [3.1](03-collection.md); it is what makes a catalogue edit a non-event for collections
  ([3.11](03-collection.md)) and what lets statistics read through to the master
  ([3.8](03-collection.md)). Any proposal to cache catalogue fields on the item is a proposal to
  build a synchronisation problem.
- **2026-08-15 — Nothing money-shaped anywhere, and the deferral is itemised.** [1.7](01-product.md)
  already barred price from catalogue entities; [3.2](03-collection.md) now keeps it off the
  collection item too — no purchase price, no currency, no valuation, no maximum price on a wantlist
  item, no total-value statistic. **Deferred, not refused:** reopening is two nullable columns on
  `collection_item` plus one on `wantlist_item`, listed in that section's working notes.
- **2026-08-15 — An import never discards data silently.** Whatever a user's file carries that our
  model has no home for — price columns ([3.2](03-collection.md)), custom fields
  ([3.4](03-collection.md)) — is either folded into a free-text field with its source label or
  dropped, and **either way the importer reports it**. Binds [10.2](10b-import.md). The reason is
  that a silent drop is indistinguishable from a bug, and the user is the only person who still has
  the original file.
- **2026-08-15 — Curated set, with free text beside it rather than inside it.** Every field drawn
  from a vocabulary (genres and styles, format descriptors, identifier types, credit roles, company
  roles) is seeded and closed, with a free-text companion field for what the vocabulary cannot say.
  Users request new values through moderation; they never mint them inline. The reason holds
  wherever it recurs: the field exists to group or filter, free text destroys grouping, and a closed
  set with no escape hatch makes users lie. Established across
  [2.3.2](02-catalogue-model.md), [2.4.2](02-catalogue-model.md), [2.4.3](02-catalogue-model.md),
  [2.4.5](02-catalogue-model.md), [2.4.6](02-catalogue-model.md).
  **The scope is shared vocabularies.** A user's own collection tags ([3.3](03-collection.md)) are
  private labels nobody else groups by, and are freely named — not an exception to the rule but a
  case outside it.
- **2026-08-15 — Messaging is the only social feature, at any stage.** No comments on releases, no
  reviews or ratings, no following, no activity feed, no forum. Decided in
  [5.11](05-messaging.md). The reason generalises past messaging and should be cited whenever a
  discussion venue is proposed in a later section: **a comment is a fact that failed to become an
  edit**, and [section 4](04-editing.md) exists so that observations land in the catalogue rather
  than in a thread beside it. Same principle as [2.4.7](02-catalogue-model.md)'s refusal of a
  data-quality field — the signal must be data, not an assertion parked next to the data.
- **2026-08-15 — No real-time transport anywhere in the system.** No WebSocket, no SSE, no client
  polling loop, no push service, in any feature. Decided for messaging in [5.3](05-messaging.md) and
  [5.9](05-messaging.md), but it is a property of [1.4](01-product.md)'s target shape — one process
  on one small box, deploys that may drop the site for ten seconds — so it binds
  [section 9](09-nfr.md), [section 11](11-stack.md) and [section 12](12-infrastructure.md) equally.
  Timeliness, where it is needed at all, is email.
- **2026-08-15 — Private user content is plaintext in Postgres, and we say so rather than imply
  otherwise.** Message bodies are readable by whoever holds the database; end-to-end encryption is
  meaningless when the same party renders the pages. Decided in [5.8](05-messaging.md). The
  obligation this creates is on [section 13](13-legal.md) (the privacy policy states it) and
  [section 14](14-security.md) (at-rest and backup handling), not on the messaging module.
- **2026-08-15 — A verified email address is the barrier, and unverified means read-only.** Every
  anti-abuse decision in the design leans on it — [4.9](04-editing.md) declined captcha and
  honeypots, [5.7](05-messaging.md) declined everything heavier than a rate limit — so it may not be
  softened later without reopening both. An unverified account reads public pages and does nothing
  else: no edit, no collection, no message. Decided in [6.1](06-accounts.md), enforced in the service
  layer so [10.1](10a-public-api.md) inherits it.
- **2026-08-15 — A user row is never deleted, only emptied.** Deletion anonymises in place and leaves
  a tombstone: [5.1](05-messaging.md) requires exactly two participant rows per conversation and
  [4.2](04-editing.md) hangs every revision off its author, so a missing row would dangle both.
  Message bodies and catalogue edits survive attributed to the tombstone; the user's own collection,
  wantlist and tags are hard-deleted. Decided in [6.5](06-accounts.md), closing what
  [5.8](05-messaging.md) handed on. **This generalises the earlier rule** that nothing a collection
  points at is hard-deleted: in this design, deletion of anything anyone else references is
  anonymisation or a tombstone, never a `DELETE`.
- **2026-08-15 — One text-rendering policy for all user prose.** Message bodies
  ([5.5](05-messaging.md)) and profile bios ([6.3](06-accounts.md)) are stored verbatim, escaped at
  render, newlines preserved, URLs linkified with `rel="noopener nofollow ugc"`. Any later
  free-text-for-humans field takes the same rule rather than inventing a second one — two policies
  means two sanitising stories and two places to get XSS wrong ([14.3](14-security.md)).
- **2026-08-15 — Postgres is the only datastore, and the binary is the only runtime.** Search, the
  job queue, sessions and rate limits are all tables ([11.4](11-stack.md)); images in object storage
  ([section 8](08-media.md)) are the sole exception, and there is no second daemon, broker, cache or
  identity service anywhere. [1.4](01-product.md) vetoed these by name; [11.4](11-stack.md),
  [11.6](11-stack.md) and [11.7](11-stack.md) record what Postgres does instead, so a later section
  needing one of those functions has its answer already.
- **2026-08-15 — No JavaScript build step, ever, and no framework doing anything silently.** The
  repository has no `package.json`, no bundler and no CSS pipeline: hand-written CSS and one vendored
  HTMX file, served as committed ([11.3](11-stack.md), [11.11](11-stack.md)). Two consequences that
  reach other sections — [section 8](08-media.md) has no JS available for image handling, and CSRF
  tokens are a template concern that [14.3](14-security.md) must specify explicitly, because plain
  HTML forms mean nothing adds them for you. Every core flow works with JavaScript disabled; HTMX
  enhances a working page and is never the mechanism.
- **2026-08-15 — The reuse boundary is the service layer, not HTTP.** Pages render from services
  directly with no JSON hop ([11.5](11-stack.md)), so any future API
  ([10.1](10a-public-api.md)) is a second adapter over the same services rather than a layer the UI
  depends on. This is what makes the rights model ([4.7](04-editing.md)), the verified-email barrier
  ([6.1](06-accounts.md)) and collection privacy ([3.7](03-collection.md)) hold identically in both,
  as each of those items requires — and the build enforces the direction, since `musilka-domain`
  cannot import `musilka-app` and `musilka-app` cannot import `musilka-web`
  ([11.11](11-stack.md)).
- **2026-08-15 — A collection item always points at a release; nothing is ever left unresolved.**
  `collection_item.release_id` is `NOT NULL`, exactly like `release.master_id`. An imported row that
  matches nothing mints a stub release and its master rather than parking as raw text. Decided in
  [10.2.3](10b-import.md), answering [10.4.3](10d-model-requirements.md). **The general rule it
  belongs to:** this design never introduces a half-linked row awaiting human triage — the queue
  would not be drained ([4.3](04-editing.md)) and every query would carry a clause about it forever.
  Corrections happen through merge ([4.4](04-editing.md)), which exists.
- **2026-08-15 — Matching is exact or it does not happen.** An import links a row to an existing
  release only on an exact identifier — **our own `release_id` following `merged_into`, then
  [10.4.1](10d-model-requirements.md)'s `external_ids`, then mint a stub**; barcode, catno+label and
  fuzzy matching are all refused. Decided in [10.2.4](10b-import.md), with the first rung added when
  [10.2.1](10b-import.md) made our own export the importer's input. The reason generalises to any
  later matching question: **a duplicate is repairable by merge, a wrong link is silent and looks
  correct** — so where the two are the alternatives, prefer the duplicate.
- **2026-08-15 — One import format, and everything else is a converter in front of it.** The
  importer reads only what [10.3](10c-export.md) writes; Discogs — and any later source — is a
  **pure function in `musilka-domain`** producing those bytes, with no database access
  ([10.2.1](10b-import.md)). It fits with nothing invented because
  [10.3.5](10c-export.md) already denormalises the catalogue crumbs into every exported row and
  [10.3.2](10c-export.md) already carries `discogs_release_id`. **Three things follow and should be
  cited rather than re-argued:** every third-party quirk — comma-packed fields, Goldmine prose,
  sentinel `none`, retailer format descriptors — is confined to one module and can never reach the
  database; a new source is a new converter, never a schema, job or report change; and the
  never-discard-silently rule crosses the seam, so a converter returns **rows *and* findings** that
  land in [10.2.9](10b-import.md)'s single report. **The converter is not "later":** our own format
  exists only after someone has used the site, so on day one it is the only way in
  ([15.1](15-roadmap.md) gives it E1.7).
- **2026-08-15 — Export is synchronous and owner-only; the job queue exists for import alone.**
  Every export streams from a single query in one request ([10.3.3](10c-export.md)), and only the
  owner may export anything at all ([10.3.1](10c-export.md)). [11.7](11-stack.md) expected export
  generation to be a queue consumer and it is not: the queue's users are the importer
  ([10.2.8](10b-import.md)), outbound email and any image derivatives — which
  [section 12](12-infrastructure.md) should read as one long-running job type, not a platform. The
  importer inherits [11.7](11-stack.md)'s restartability rule in full, since a deploy will kill it
  mid-file.
- **2026-08-15 — Never discard silently, in both directions, and the export file is a contract.**
  [10.2](10b-import.md)'s obligation to report what an import dropped or folded applies equally to
  what an export flattens ([10.3.2](10c-export.md)). And once a file leaves the box its columns and
  identifiers are public: appending a column is compatible, renaming or reordering is not. This is
  what makes [10.4.6](10d-model-requirements.md) (public identifier shape) urgent rather than
  cosmetic — the first export makes that choice permanent.
- **2026-08-15 — There is no catalogue import channel, including for administrators.** An admin
  loading a dump through `psql` is exactly what [1.5](01-product.md) forbids; the shell it happens in
  does not change the act. Catalogue rows arrive only as import-minted stubs
  ([10.2.3](10b-import.md)) or hand edits ([section 4](04-editing.md)). Decided in
  [10.2.10](10b-import.md). Vocabulary seed data loaded by a migration ([11.8](11-stack.md)) is not
  an exception — it is a list we wrote ourselves.
- **2026-08-15 — One cache is permitted, and the test that admits it.** `master.search_tsv` is
  denormalised on purpose ([7.2](07-search-ux.md)) while [3.1](03-collection.md) forbids exactly that
  on a collection item. The rule that reconciles them, and which any later caching proposal must
  pass: **a cache is allowed when one statement rebuilds it from the truth, so a stale value is
  detectable.** A search vector passes; a title copied onto a collection item fails, because a copy
  that differs from its source is indistinguishable from a deliberate edit. Everything else stays
  computed live — [3.8](03-collection.md)'s statistics, [7.3](07-search-ux.md)'s facet counts,
  [6.3](06-accounts.md)'s edit count.
- **2026-08-15 — Search is three indexed columns, and nothing is searchable that we cannot fill.**
  Full text covers artists, masters and labels only; releases are found by exact identifier,
  tracks and users are not searchable at all ([7.1](07-search-ux.md)). Postgres FTS in the `simple`
  configuration plus `unaccent` and `pg_trgm`, with trigram used only as a no-results fallback
  ([7.2](07-search-ux.md)). The recurring reason: a result group over a table the import channel
  cannot fill ([1.5](01-product.md)) is an empty heading on every page.
  **The known gap, recorded so it is not rediscovered as a bug:** artists are findable in any script
  ([2.2.1](02-catalogue-model.md)) but titles have no variant rows, so a Latin-typed query for a
  Cyrillic title fails. The fix is an additive `master_title` table; the workaround is the artist
  page.
- **2026-08-15 — Every view is a URL, and the enhancement is never the mechanism.** Filters, sorting
  and pagination are `GET` parameters ([7.3](07-search-ux.md)) and autocomplete degrades to a text
  field and a button ([7.4](07-search-ux.md)). This is [11.3](11-stack.md)'s no-JS rule made
  concrete, and it is also what makes [7.8](07-search-ux.md)'s indexability free — but note the
  inverse obligation it creates: because filtered URLs exist, they must be `Disallow`ed, or the
  facet combinatorics become a crawl trap pointed at one small box.
- **2026-08-15 — Images: uploads only, object storage only, and no original is kept.** The bucket is
  the sole non-Postgres datastore ([8.2](08-media.md), granted in advance by [1.4](01-product.md)),
  keys are content-addressed by `sha256` so deduplication and immutable caching come for free, and
  every upload becomes exactly two WebP derivatives — 1600 px and 300 px — after which the original
  is discarded ([8.1](08-media.md), [8.3](08-media.md)). This is the one **irreversible** decision in
  the section: nothing uploaded before a change of mind can be re-derived at higher resolution.
  Processing is a `vips` subprocess in the request, not a queue job ([8.3](08-media.md)), which
  leaves the importer and outbound email as the queue's complete set of users.
- **2026-08-15 — Image takedown is the only hard delete of something others reference, and it is
  bounded.** Everything else anyone points at soft-deletes or anonymises (a user's own collection
  rows always were an ordinary delete, [3.10](03-collection.md)); a rights-holder complaint cannot be
  answered by a tombstone that still serves the file, so the blob really goes ([8.4](08-media.md)). What survives
  is the `sha256` plus a `blocked_at`, which content-addressed storage turns into free refusal of a
  re-upload. The release itself is untouched — removing an image is not [4.5](04-editing.md)'s
  deletion.

- **2026-08-15 — The catalogue is CC0; the images and the user's own data are not.** Masters,
  releases, artists, labels, credits and vocabularies are dedicated to the public domain, and the
  ToS makes each contribution a CC0 grant ([13.1](13-legal.md)). Uploaded artwork is excluded — we
  never held rights in it, which is why [8.4](08-media.md) removes rather than argues — and
  collections, wantlists, notes, tags, messages and profiles are excluded absolutely. Any future
  dump ([10.3.4](10c-export.md)) carries catalogue rows and image *keys*, never image bytes and
  never a user's rows.
- **2026-08-15 — The browser talks to our origin and the image bucket, and to nothing else, ever.**
  No analytics, no fonts, no CDN, no CAPTCHA, no error tracker, no third-party script of any kind
  ([13.4](13-legal.md), and [9.5](09-nfr.md) declined Sentry on the same grounds). Three things
  follow and should be cited rather than re-argued: there is exactly one cookie and therefore **no
  cookie banner**, [13.3](13-legal.md)'s sub-processor list is three infrastructure providers, and
  [14.3](14-security.md)'s CSP can forbid `unsafe-inline` — which in turn bans inline styles and
  `hx-on:` attributes in templates.
- **2026-08-15 — Authorisation is a clause in the query, never a check after the load.** Every
  user-scoped read and write takes the acting user and constrains on it in SQL; there is no
  `getById` followed by an `if` ([14.3](14-security.md)). This is what makes sequential integer ids
  harmless — relevant to [10.4.6](10d-model-requirements.md), still open — and it is why
  [3.7](03-collection.md)'s share token is a capability rather than an identifier. It lives in the
  service layer, so [10.1](10a-public-api.md) inherits it like every other rule
  ([11.5](11-stack.md)).
- **2026-08-15 — EU/EEA, everything.** The privacy documents are written to the GDPR and 152-FZ is
  not claimed ([13.3](13-legal.md)), so the VPS, the image bucket, [9.3](09-nfr.md)'s backup bucket
  and the mail sender all sit in the EU/EEA. This is a constraint on
  [12.1](12-infrastructure.md), not a preference, and it is also what removes the international
  transfer question from the policy entirely.
- **2026-08-15 — A public identifier is a `bigint`, and it is permanent from the first exported
  file.** `bigint GENERATED ALWAYS AS IDENTITY`, one sequence per entity type, exposed exactly as it
  is — no UUID, no slug in the identifier position, no opaque code. Decided in
  [10.4.6](10d-model-requirements.md), closing what had become the oldest unblocked P0 item.
  Unguessability was never a requirement because [14.3](14-security.md) puts authorisation in the
  `WHERE` clause; the slug stays cosmetic ([7.5](07-search-ux.md)). **Two rules travel with it:** an
  id never appears without its entity type (`/release/42`, `release_id`), and **old ids keep
  resolving** — merged entities follow `merged_into`, deleted ones resolve to a tombstone — which is
  what makes a two-year-old export re-importable.
- **2026-08-15 — Provenance is the revision, and nothing automatic ever overwrites a human value.**
  No field-level provenance columns anywhere: `revision.source` and `revision.import_id`
  ([10.4.2](10d-model-requirements.md)) plus `external_id.added_by` are the whole of it, and "is this
  an untouched import stub?" is derived (exactly one revision, `source = import`) rather than stored.
  The general form, which any later "should we mark where this came from" question inherits: the
  edit history is data, a provenance flag is an assertion parked beside the data —
  [2.4.7](02-catalogue-model.md) refused the same thing under a different name.
- **2026-08-15 — There is no public API, and nothing anywhere is reserved for one.**
  [10.1.1](10a-public-api.md) looked for an audience and found none: third parties have no reason to
  build on a catalogue that is thin by construction ([1.5](01-product.md)), mobile was declined at
  [7.6](07-search-ux.md), and "dump my collection with a script" is what [10.3](10c-export.md)
  already does. **Everything an API would need already exists for other reasons** — service layer,
  authorisation in the query, limits below HTTP, stable ids, a monotonic revision id, a generated
  OpenAPI document — so the reopening cost is an adapter, not an architecture. Three refusals inside
  it are permanent rather than deferred: **no OAuth2 server**, **no catalogue writes through the
  API**, and **no webhooks, ever** — the last because a webhook is an outbound request to a
  user-supplied URL, which [14.3](14-security.md) forbids by name.
- **2026-08-15 — One compatibility rule, for files and for any API.** Appending a field or an
  endpoint is compatible; renaming, removing, reordering, retyping or tightening is not
  ([10.3.2](10c-export.md), [10.1.3](10a-public-api.md)). Deliberately one sentence rather than two,
  because two would mean two habits and one of them wrong.
- **2026-08-15 — The queue is for work a user started; everything recurring is a timer.** The job
  queue's consumers are exactly two — the importer and outbound mail
  ([10.4.5](10d-model-requirements.md)) — after [10.3.3](10c-export.md) made export synchronous and
  [8.3](08-media.md) made image processing a subprocess. Backups, the error digest, the rate-limit
  sweep and [10.4.8](10d-model-requirements.md)'s file sweeper are systemd timers running a command
  ([12.2](12-infrastructure.md)). Conflating the two is how a queue grows a scheduler.
- **2026-08-15 — Two environments, and production data never leaves the box except encrypted.**
  Local and production; no staging ([12.4](12-infrastructure.md)). Development runs on generated
  fixtures ([11.8](11-stack.md)) because the database holds message bodies and email addresses in
  plaintext ([5.8](05-messaging.md), [14.1](14-security.md)). The single bounded exception is
  [9.3](09-nfr.md)'s restore drill, which decrypts real data on the machine holding the `age` private
  key, drops the scratch database afterwards, and is a drill rather than an environment — it is also
  what replaces staging when a risky migration needs rehearsing ([9.2](09-nfr.md)).

## Rejected approaches

What we considered and turned down, and why — so we do not re-litigate it in three months.

- **2026-08-14 — Seeding the catalogue from a Discogs or MusicBrainz dump/API.** Turned down at
  [1.5](01-product.md). It would solve the thin-catalogue problem, but it buys a licensing question,
  an availability and rate-limit dependency, and a sync problem, in exchange for making the editing
  features — the interesting part of a portfolio project — largely redundant.
- **2026-08-14 — Building for future commercialisation.** Considered at [1.1](01-product.md) as
  "portfolio with commercial potential" and set aside. We do not pay the up-front cost of keeping
  that door open; if it is ever re-opened, re-seeding the catalogue under a different data licence is
  accepted as the price.
- **2026-08-14 — A marketplace, in any form or at any stage.** Turned down at [1.7](01-product.md).
- **2026-08-15 — A global `Track` / composition entity.** Turned down at
  [2.1.4](02-catalogue-model.md). It would give "which releases contain this song", but no import
  channel carries a tracklist ([1.5](01-product.md)) so the table would be near-empty
  ([1.4](01-product.md)); doing it properly needs the work/recording distinction, and doing it by
  title produces garbage. Reversible additively — tracklist items keep their titles — so a future
  work entity is a new table plus a linking pass.
- **2026-08-15 — Free-form genre and style tags.** Turned down at [2.3.2](02-catalogue-model.md) in
  favour of a curated vocabulary. `hip hop` / `hip-hop` / `HipHop` fragments within a week and
  filtering is the entire point of the field.
- **2026-08-15 — A table per medium for formats**, and equally a free-text format string. Turned down
  at [2.4.2](02-catalogue-model.md). Typed-to-SQL makes every new descriptor a migration and leaves a
  half-known imported format nowhere to go; free text kills "show me all my cassettes", which
  [1.2](01-product.md) exists to support. The sum types live in the domain layer instead.
- **2026-08-15 — A data-quality field** in the Discogs "Needs Vote / Complete and Correct" mould.
  Turned down at [2.4.7](02-catalogue-model.md): a status column with no voting workflow behind it is
  decoration that goes stale. The quality signal is the edit history
  ([section 4](04-editing.md)), which is data rather than an assertion.
- **2026-08-15 — Collection folders, and custom user-defined fields.** Turned down at
  [3.3](03-collection.md) and [3.4](03-collection.md) in favour of user-owned tags plus a free-text
  note. Folders impose single membership we cannot justify; custom fields are a per-user schema
  (definitions, types, validation, rendering, export columns, API shape) bought for an undemonstrated
  need. Both are what tags already do, and a Discogs folder imports as a tag losslessly.
- **2026-08-15 — A current-valuation field and a total-collection-value statistic.** Turned down at
  [3.2](03-collection.md) and [3.8](03-collection.md) on top of the money deferral: with
  [1.5](01-product.md) forbidding market data, valuation could only be a number typed once and never
  revisited, and a total computed from it is fiction wearing a statistic's clothes.
- **2026-08-15 — Per-item collection privacy.** Turned down at [3.7](03-collection.md). Real for some
  collectors, but it puts a clause in every listing query and a control on every row to hide what a
  private collection already hides. Sensitive fields (note, purchase date and place) are owner-only
  unconditionally instead, which covers the actual worry.
- **2026-08-15 — Voting on edits, MusicBrainz style.** Turned down at [4.3](04-editing.md). Voting
  presumes an electorate; ours has one member ([1.4](01-product.md)), and a vote queue nobody drains
  is worse than no queue at all. Reopen only if there are ever enough simultaneous editors to
  disagree.
- **2026-08-15 — Reputation, contributor ranks and a `trusted` role.** Turned down at
  [4.8](04-editing.md) and [4.7](04-editing.md). Ranks exist to allocate moderation privileges by
  earned standing, which is the mechanism [4.1](04-editing.md) already declined; at our volume the
  leaderboard has one entry. A derived "edits made" count on the profile is kept — it gates nothing.
- **2026-08-15 — Captcha, honeypot, and a prebuilt mass-revert tool.** Turned down for the MVP at
  [4.9](04-editing.md) as cost against an imaginary threat, behind a verified-email wall at
  portfolio scale. Note the asymmetry that decided it: mass revert is *not built* but its data *is
  stored*, because the tool is a day's work later and the history is unrecoverable.
- **2026-08-15 — `Series` as an entity.** Turned down at [2.1.5](02-catalogue-model.md) — real on
  real objects, but a mostly-empty table with a moderation cost at ~12,000 releases, and expressible
  as a format descriptor or a note. Additive if ever reopened.
- **2026-08-15 — Comments on releases, reviews and ratings, following, an activity feed, a forum.**
  All turned down at [5.11](05-messaging.md), and each for its own reason: comments compete with the
  edit path, ratings are noise at ten users and are RateYourMusic's product anyway, a feed needs a
  population and a rate of activity we will not have, and a forum is a second product with the
  largest moderation load available to us.
- **2026-08-15 — Group chat.** Turned down at [5.1](05-messaging.md). Participant management,
  divergent read state and "who may add whom" bought for a population with no groups in it. Additive
  later only because per-user state lives on a `conversation_participant` row for its own reasons.
- **2026-08-15 — Attachments in messages, and markdown or restricted HTML in message bodies.**
  Turned down at [5.4](05-messaging.md) and [5.5](05-messaging.md). Private attachments are the
  unmoderatable worst case of the storage problem [1.4](01-product.md) already flags; rich text
  means owning a sanitiser and a standing XSS liability for a table with dozens of rows. Plain text
  upgrades to markdown cleanly later; the reverse migration does not exist.
- **2026-08-15 — Web push, and read receipts.** Turned down at [5.9](05-messaging.md) and
  [5.6](05-messaging.md). Push is a service worker, VAPID keys, a permission prompt and a
  subscription table aimed at users who are not in the app. Receipts are social pressure and a
  privacy choice made on the user's behalf — but `last_read_at` is stored regardless, so this is
  another case of the [4.9](04-editing.md) asymmetry: store the data, do not build the tool.
- **2026-08-15 — OAuth login (Google/GitHub/VK), at any stage.** Turned down at
  [6.1](06-accounts.md). Every provider is a console registration, a secret in configuration, a
  callback route and an account-linking matrix, and it removes no work: the mail sender, the address
  column and the verification flow are all needed regardless ([6.1](06-accounts.md)'s barrier). It
  only adds a second way in. Additive later as a provider table beside the password column.
- **2026-08-15 — 2FA in the MVP.** Deferred at [6.2](06-accounts.md), not refused. With nothing
  money-shaped anywhere ([1.7](01-product.md), [3.2](03-collection.md)), the worst case of a stolen
  account is a reversible edit ([4.4](04-editing.md)) and a read mailbox — against secret storage,
  recovery codes and a lockout path with no support channel. If reopened it is TOTP for **everyone**,
  never an admin-only privilege: a second factor with one user is a code path with no coverage.
- **2026-08-15 — A key/value settings table, and every setting not already decided elsewhere.**
  Turned down at [6.6](06-accounts.md). The complete list is three columns on `user`
  (`collection_visibility`, `wantlist_visibility`, `notify_new_message`); a key/value table is a
  schema you cannot query, constrain or index. Explicitly not settings: transactional mail, profile
  visibility, per-conversation muting or digests ([5.9](05-messaging.md)), per-item collection
  privacy ([3.7](03-collection.md)), and "who may message me" — [5.7](05-messaging.md)'s blocking
  covers that reactively.
- **2026-08-15 — A profile links field, and profile fields we have no use for.** Turned down at
  [6.3](06-accounts.md). A link list is a table, a rendering, a validation and an SEO-spam target for
  a page nobody reads; URLs go in the bio and are linkified there. No real name, birthday, gender or
  phone either — data we do not hold is data [13.3](13-legal.md) need not account for and
  [14.6](14-security.md) need not protect.
- **2026-08-15 — An undo window on account deletion.** Turned down at [6.5](06-accounts.md).
  It needs a scheduled purge job and a fourth account state to explain, on top of the three
  ([6.5](06-accounts.md)) already has. Deletion is immediate and irreversible, with an export offered
  on the same page ([10.3](10c-export.md)) so that leaving does not mean losing.
- **2026-08-15 — Elixir/Phoenix, and building both languages in parallel branches.** The fallback in
  [1.12](01-product.md) was not exercised at [11.2](11-stack.md): section 5 refused real time
  ([5.3](05-messaging.md)), so OTP, LiveView and presence have no consumer, and what remained of the
  Elixir case was that it ships sooner — which [1.11](01-product.md) already paid for. Parallel
  branches were raised and declined: every design decision implemented twice at
  [1.11](01-product.md)'s pace, branches that diverge with no cross-language merge, and
  [1.10](01-product.md) needing one deployed thing a stranger can use. A timeboxed bakeoff spike was
  the defensible version and was also declined in favour of committing. Reopening means restarting
  the codebase, so it happens on evidence from a built E0/E1 or not at all.
- **2026-08-15 — An SPA, an SSR framework, Tailwind and every UI kit.** Turned down at
  [11.1](11-stack.md) and [11.3](11-stack.md). An API + SPA split would build and version a JSON
  layer whose only consumer is our own frontend, against [1.9](01-product.md) deferring the public
  API; Tailwind and the kits drag a Node toolchain, a build and a second CI step into a Haskell
  repository for styling alone, when [11.11](11-stack.md) is working to keep the deploy at one
  artifact.
- **2026-08-15 — JWT, and off-the-shelf auth (Auth.js, Keycloak, Supabase Auth).** Turned down at
  [11.6](11-stack.md). JWT is excluded outright by [6.2](06-accounts.md) — reset and email change
  must invalidate *every* session, and honouring that means a denylist, which is the session table
  with worse failure modes. The off-the-shelf products exist chiefly to solve federation, and
  [6.1](06-accounts.md) rejected OAuth, so there is nothing to federate.
- **2026-08-15 — GraphQL and tRPC, and ORM-shaped database layers.** Turned down at
  [11.5](11-stack.md) and [11.4](11-stack.md). tRPC presumes a TypeScript client that
  [11.3](11-stack.md) does not have; GraphQL is a resolver layer and an N+1 problem bought for zero
  third-party clients at ~12,000 releases ([1.4](01-product.md)). `rel8`/`opaleye` and
  `persistent`/`esqueleto` were declined for a shared reason worth remembering: both introduce a
  second schema definition beside the SQL migrations, and [1.12](01-product.md) records the database
  layer as the area with no prior experience at all.
- **2026-08-15 — Down-migrations.** Turned down at [11.8](11-stack.md). A down-migration for anything
  that touched data is a fiction; mistakes are corrected forward, and the actual safety net is
  [section 12](12-infrastructure.md)'s tested restore, which [1.4](01-product.md) already demands.
- **2026-08-15 — A coverage percentage target.** Turned down at [11.10](11-stack.md). It is the one
  measure of [1.1](01-product.md)'s quality bar that is gameable by testing getters, and at
  [1.11](01-product.md)'s pace a number becomes a chore that displaces real tests. Replaced by a
  named list of what must be covered — including [1.10](01-product.md)'s export→import round trip as
  an automated property, rather than a thing checked once by hand.
- **2026-08-15 — Discogs live sync via the user's OAuth, and two-way sync back to Discogs.** Turned
  down at [10.2.1](10b-import.md) and [10.2.11](10b-import.md), and note that neither was really
  ours to decide: [1.5](01-product.md) already bars the server from calling an external music
  database. What the items add is the second cost — write-scoped credentials to a user's account at
  another service, the largest thing we could hold, for a feature with no stated demand. Nothing is
  reserved to keep the option open; it would be a new service against a new API, not a schema change.
- **2026-08-15 — Unresolved collection items, a match-review screen, and fuzzy matching.** Turned
  down at [10.2.3](10b-import.md) and [10.2.4](10b-import.md). All three are the same mistake in
  different clothes: they answer an ambiguous row by asking a human to look at up to ten thousand of
  them. The observed export carries enough (artist, title, label, catno, format) to mint a
  recognisable stub, and [2.1.2](02-catalogue-model.md) already expects duplicate masters, so the
  ambiguity is absorbed by merge instead.
- **2026-08-15 — The preview-then-apply import, and keeping the uploaded file.** Turned down at
  [10.2.6](10b-import.md). A preview needs a parsed ten-thousand-row intermediate with its own
  garbage collection, and buys the right to refuse a result that is already reversible on the side
  that matters (the user's own rows). The file is deleted when the job ends: it is personal data
  ([10.5.3](10e-legal-sources.md)), the user still has the original, and keeping it means writing a
  retention policy for something nobody reads. Only its `sha256` survives, for
  [10.2.7](10b-import.md).
- **2026-08-15 — XLSX export, and a Discogs-compatible CSV.** Turned down at
  [10.3.2](10c-export.md). XLSX is a zip of XML needing a library for a file every spreadsheet opens
  from CSV. Compatibility means tracking an undocumented third-party format we cannot test against —
  while we have not even verified their collection export's columns ([10.2.2](10b-import.md)). The
  part that carries real value, `discogs_release_id` as a column, is kept without the claim.
- **2026-08-15 — Scheduled and filtered exports.** Turned down at [10.3](10c-export.md)'s working
  notes: no monthly emailed export, no per-list format preference, no export of a filtered subset.
  Each is a setting or a job for a page that is one click and under a second, and
  [6.6](06-accounts.md) closed the settings question against exactly this kind of addition.
- **2026-08-15 — Track search, user search, and "similar releases".** Turned down at
  [7.1](07-search-ux.md) and [7.4](07-search-ux.md). The first two are result groups over tables that
  are empty or trivial at our scale — a tracklist nothing fills ([2.1.4](02-catalogue-model.md)) and
  a directory of tens of nicknames whose only consumer would be a deferred feature. Similarity would
  be a recommendation engine computed from genre overlap at ~12,000 releases; the honest version
  already exists as links (other pressings, other releases by this artist, this label).
- **2026-08-15 — Transliteration in the search layer, and title variant rows.** Turned down at
  [7.4](07-search-ux.md). The first is barred outright by [2.2.1](02-catalogue-model.md)'s invariant
  and needed no re-deciding; the second is the additive fix for the gap that leaves, declined for now
  because nothing would fill it — no import channel carries a second title.
- **2026-08-15 — A PWA, a native app, camera barcode scanning, and a dark-theme toggle.** Turned down
  at [7.6](07-search-ux.md) and [7.7](07-search-ux.md). A service worker exists for offline and push,
  and [5.9](05-messaging.md) already refused push; a native app is a second codebase against
  [1.11](01-product.md)'s pace; a scanner is a vendored JS decoder against [11.3](11-stack.md)'s
  no-build-step line, when typing a barcode into the existing field is the whole feature. The theme
  toggle needs somewhere to persist a preference, and [6.6](06-accounts.md) closed that list at three
  columns — `prefers-color-scheme` is what the user already chose.
- **2026-08-15 — Keeping the uploaded original, AVIF, and a third image size.** Turned down at
  [8.1](08-media.md) and [8.3](08-media.md). Each multiplies the one number [1.4](01-product.md)
  predicted would break first, and each buys a re-encode or a breakpoint nobody has asked for. Note
  this is the section's irreversible call: originals already discarded cannot be recovered, so
  reopening means "keep them from this date onward".
- **2026-08-15 — Automated NSFW or copyright screening of uploads.** Turned down at
  [8.4](08-media.md). Every option is an external API ([1.5](01-product.md)) or a model hosted on a
  2 GB box ([1.4](01-product.md)) — and sleeve art is routinely nude, so our content is precisely
  where a classifier's false-positive rate is worst. Moderation is reactive through
  [4.6](04-editing.md)'s existing report row.
- **2026-08-15 — Audio previews, and links to streaming services.** Turned down permanently at
  [8.5](08-media.md). Hosting audio is a licence we cannot get for a catalogue of physical objects
  ([1.2](01-product.md)); the "safer" link alternative needs an identifier that comes from either an
  external call ([2.5.6](02-catalogue-model.md) refused it by name) or hand entry, which means a
  field empty on every row that asks our editors to do a third party's data entry.

- **2026-08-15 — Russian hosting and 152-FZ compliance.** Turned down at [13.3](13-legal.md). It is
  not a legal preference but an architectural one: Art. 18.5's localisation requirement would decide
  [12.1](12-infrastructure.md)'s provider for us and add Roskomnadzor notification, where writing
  one GDPR-shaped policy costs a page and puts the database, both buckets and the mail sender in one
  jurisdiction with no transfer question.
- **2026-08-15 — CC BY-SA for the data, AGPL and a private repository for the code, and a CLA.**
  Turned down at [13.1](13-legal.md) and [13.2](13-legal.md). Share-alike would enclose material
  that reached us as CC0 and buy a compliance burden for reusers of largely uncopyrightable facts;
  AGPL defends against a commercial fork [1.1](01-product.md) says we do not expect; a private
  repository deletes half of what a portfolio project is for. A CLA manages a contributor population
  of one.
- **2026-08-15 — Analytics of every kind, and the cookie banner with them.** Turned down at
  [13.4](13-legal.md). Hosted analytics brings a banner, a transfer question and a script tag to a
  product with none; self-hosted analytics is a second daemon and usually a second database on
  [1.4](01-product.md)'s box. [9.5](09-nfr.md)'s request log answers the same questions offline over
  data we already keep. The banner then has nothing to consent to, and showing one anyway would be a
  lie.
- **2026-08-15 — Age verification, a registered DMCA agent, and DSA compliance machinery.** Turned
  down at [13.5](13-legal.md). Verifying age would collect more personal data than the whole rest of
  the service holds; the DMCA agent is a US safe-harbour formality and [13.3](13-legal.md) put us in
  the EU; statements of reasons, transparency reports and appeals are scoped to services far larger
  than this. A published abuse address and a takedown that actually happens is the substance.
- **2026-08-15 — CAPTCHA, confirmed closed rather than re-decided.** [4.9](04-editing.md) and
  [5.7](05-messaging.md) had already declined it; [14.5](14-security.md) closes it permanently on a
  new ground — every CAPTCHA is a third-party script, which [13.4](13-legal.md)'s invariant now
  forbids outright and [11.3](11-stack.md)'s no-JS rule would break anyway.
- **2026-08-15 — Password composition rules, rotation, a pepper, and hard account lockout.** Turned
  down at [14.2](14-security.md) in favour of a 10-character minimum, a vendored common-password
  list, and exponential backoff. Lockout is the decisive one: it hands an attacker a tool for
  denying a victim their own account, and we have no support channel to undo it.
- **2026-08-15 — SVG uploads and link previews.** Turned down at [14.4](14-security.md) and
  [14.3](14-security.md). An SVG is a script container, and a link preview would be the only code
  path in the system that fetches a URL a user supplied — inventing an SSRF surface
  [1.5](01-product.md) had already deleted.
- **2026-08-15 — UUIDs and slugs as public identifiers, and `synced_at`/`url` on the external-id
  table.** Turned down at [10.4.6](10d-model-requirements.md) and
  [10.4.1](10d-model-requirements.md). UUIDv7 costs 36 characters in every URL, export cell and log
  line plus a wider index on every foreign key, to hide a row count the design publishes; nothing in
  a one-process, one-writer system needs distributed id generation. A slug as the identifier would
  make a retitle a broken link and destabilise an export column. `synced_at` describes a sync
  [1.5](01-product.md) forbids, and a stored `url` is both derivable and an invitation for some
  later code path to fetch it.
- **2026-08-15 — Field-level provenance.** Turned down at
  [10.4.2](10d-model-requirements.md): a parallel schema written on every edit, read only to answer
  a question [2.4.7](02-catalogue-model.md) already refused to answer with a stored status, and
  derivable from the revision anyway.
- **2026-08-15 — OAuth2 as an authorisation server, catalogue writes through the API, and
  webhooks.** Turned down at [10.1.4](10a-public-api.md), [10.1.7](10a-public-api.md) and
  [10.1.10](10a-public-api.md). OAuth2 is a consent screen, a client registry and refresh rotation
  built for third-party applications that do not exist; a write token turns
  [4.9](04-editing.md)'s "an edit is a human at a form" into a loop; and a webhook is the outbound
  fetch of a user-supplied URL that [14.3](14-security.md) forbids outright. Also declined there:
  header-based versioning, offset pagination, a sandbox environment, generated client libraries, and
  any attribution or non-commercial requirement on the data — [13.1](13-legal.md)'s CC0 forbids us
  from imposing one.
- **2026-08-15 — PaaS, Docker on the server, Nix, and a staging environment.** Turned down at
  [12.1](12-infrastructure.md), [12.2](12-infrastructure.md) and [12.4](12-infrastructure.md). Every
  PaaS is built around an ephemeral filesystem, a managed database as an add-on and a scaling model
  [1.4](01-product.md) vetoes, when we need Postgres on a unix socket, a persistent disk and cron.
  A container on the box would be a second artifact where [11.11](11-stack.md) works to keep one.
  Nix would enlarge [1.12](01-product.md)'s largest known gap to solve a reproducibility problem a
  pinned GHC, `index-state` and a pinned builder image already solve. Staging is a second everything
  to rehearse against data that is not the data — [9.3](09-nfr.md)'s restore gives a real-data
  scratch copy on demand instead. Also declined there: secret managers of every kind (four secrets
  on one box), Caddy (for `limit_req` alone), Conventional Commits (no version, no changelog), and
  Linear/Jira and every hosted tracker.

## Constraints discovered

Facts found while designing that constrain later choices (external ToS, format quirks, volumes).

- **2026-08-14 — A Discogs CSV row is richer than the agenda assumed.** Verified on a real export:
  rows carry artist, title, label, catno, format and release_id — enough to build a recognisable
  stub, not an empty shell. `recommendations.md` point 8 overstates the problem and should not be
  used as an argument for needing an external metadata source. Details in
  [10.2 working notes](10b-import.md).
- **2026-08-14 — Discogs packs multiple values into single CSV fields, comma-joined and
  positionally paired** (`label` against `catno`), while label names may themselves contain commas.
  Any CSV parsing design that splits on `,` is wrong. Details in [10.2 working notes](10b-import.md).
  **Rescoped 2026-08-15:** this and the finding above are constraints on the **Discogs converter**,
  not on the importer — [10.2.1](10b-import.md) put a format boundary between them, so a mistake
  here breaks one pure module rather than the import path.
- **2026-08-14 — EU database rights cover repeated extraction of insubstantial parts.** A single
  user's collection export is not a substantial part of anyone's database, so one upload is safe. But
  the Database Directive also bars *repeated and systematic* extraction of insubstantial parts, and
  accreting many users' exports into one public catalogue is arguably that. Accepted as a residual
  risk at portfolio scale ([1.5](01-product.md)); it would need real legal review before any public
  launch at volume or any commercial turn. Not verified against current case law.

## Open questions blocking other items

Questions that must be answered before some other item can be closed. Format: what is blocked ← what we are waiting on.

- ~~[1.9](01-product.md) MVP scope and [1.10](01-product.md) success criterion~~ — **closed
  2026-08-15**, immediately after section 2 removed what they were waiting on. **Section 1 is now
  fully closed.** The MVP is the collector's round trip (upload, browse, correct, image, search,
  export) with messaging, the public API and any moderation queue explicitly out; success is the
  round trip working on a real export plus a second person using it unaided.
  **What this now constrains:** [section 15](15-roadmap.md) inherits a scope line and two sequencing
  rules (export before import; master merge early, not late), and every remaining P0 section should
  be read against the deferred list rather than designed in full.
- ~~Multi-script artist and title names (`Кино` / `Kino`)~~ — **closed 2026-08-15** at
  [2.2.1](02-catalogue-model.md): one artist, many name rows, one search index, no transliteration
  anywhere. Kept here only as a pointer, since [1.8](01-product.md) and several notes refer to it as
  an open question. It is now an invariant above.
- ~~[11.2](11-stack.md) final stack confirmation ← section 5~~ — **closed 2026-08-15.** It is
  **Haskell**, with Servant + warp + Lucid, `hasql` and hand-written SQL. **Section 11 is now fully
  closed** and the Elixir fallback was not exercised; the reasoning below is kept because several
  items still refer to this as pending. What section 11 hands on is listed in
  [its working notes](11-stack.md).
  *Original entry, unblocked earlier the same day:*
  Sections 1–5 are now all closed and **nothing is waiting on a model section any more**;
  [11.2](11-stack.md) can be decided whenever section 11 is taken up. What the model sections handed
  it: section 2 — four entities, no global track table, vocabularies as seeded lookups, an ordered
  artist credit in three parallel tables; sections 3 and 4 — no denormalisation, aggregate-snapshot
  history, role-based rights in a service layer; section 5 — **nothing new at all** (no real-time
  transport, no file storage, no search backend, no push service; five ordinary tables and one email
  template). [1.12](01-product.md) records the preference (Haskell) and the fallback
  (Elixir/Phoenix), and the argument for Elixir that the messaging module might have supplied — a
  real-time vertical — **did not materialise**, since [5.3](05-messaging.md) refused real time
  outright. The stack may be chosen on the catalogue's merits alone.
- ~~[6.5](06-accounts.md) account deletion ← [5.8](05-messaging.md)'s tombstone constraint~~ —
  **closed 2026-08-15.** The account is anonymised in place and the row is never deleted; messages,
  conversations and catalogue edits survive attributed to the tombstone, collection and wantlist are
  hard-deleted. **Section 6 is now fully closed.** What it hands on: [13.3](13-legal.md) inherits
  three statements it may not overstate (what deletion erases and keeps, that message bodies are
  plaintext and survive the sender, that the only personal data held is an address, a nickname and a
  bio); [section 14](14-security.md) inherits password storage ([14.2](14-security.md)), the rate
  limits on registration, login, reset and resend ([14.5](14-security.md)) and log retention
  ([14.6](14-security.md)); [8.2](08-media.md) gains a second consumer in the avatar;
  [section 12](12-infrastructure.md) inherits transactional-mail deliverability for four templates.
- ~~Soft delete and merge semantics ← [section 4](04-editing.md)~~ — **closed 2026-08-15.** It is
  `merged_into` *and* `deleted_at`, as two distinct operations with different meanings, and neither
  is a consequence of edit versioning ([4.4](04-editing.md), [4.5](04-editing.md)). Now an invariant
  above. **What it hands on:** [3.11](03-collection.md) inherits three cases to answer — edited
  (nothing happens), merged (the item follows the pointer), deleted (the item survives and is shown
  as deleted).
- ~~[10.3.4](10c-export.md) a public dump of the whole catalogue ← [13.1](13-legal.md) the data
  licence~~ — **unblocked 2026-08-15.** The licence is CC0 and the ToS grant makes contributions
  match it, so publishing a dump is now a scheduling decision: JSON Lines from
  [10.3](10c-export.md)'s serialisers, plus a `LICENSE` file in the archive and a cron entry.
  **What survives the unblocking is the risk, not the blocker:** the EU database-rights constraint
  below was accepted for *ingesting* one user's export and never for redistributing the accretion,
  and [13.1](13-legal.md) can only dedicate what we hold. A dump at any real volume wants an actual
  lawyer first.
- **The domain is unregistered, and two things wait on it** ← [13.6](13-legal.md) deferred it to
  [12.1](12-infrastructure.md) deliberately. The TLS certificate is trivial; the one that bites is
  transactional mail — [section 6](06-accounts.md)'s four templates from a fresh domain with no
  SPF, DKIM or DMARC land in spam, and that breaks [6.1](06-accounts.md)'s verified-email barrier,
  which every anti-abuse decision in the design leans on. Register and publish the records in the
  same sitting as the first deploy setup, and give reputation time before [1.10](01-product.md)'s
  stranger is invited.
  **Status as of 2026-08-15:** still open, and now open *by choice* — [12.1](12-infrastructure.md)
  and [12.5](12-infrastructure.md) carry `[~]` because naming a hosting provider and a mail sender
  was postponed. Everything vendor-independent about both is decided (a single EU/EEA VPS with
  Postgres on a unix socket, nginx with [9.4](09-nfr.md)'s caps, certbot, two buckets at two
  providers, forwarding-only inbound mail). **What is blocked is the first deploy, and nothing
  analytical** — no design decision anywhere waits on the names. [15.4](15-roadmap.md) carries
  deliverability as risk 2 and [15.3](15-roadmap.md) notes that week one ends at green CI if the
  provider is still unchosen.
- ~~[10.4.6](10d-model-requirements.md) public entity identifiers ← nothing, and that is the
  problem~~ — **closed 2026-08-15.** It is `bigint GENERATED ALWAYS AS IDENTITY`, one sequence per
  entity type, exposed as it is, and it is now an invariant above. Both callers are satisfied:
  [10.3](10c-export.md)'s column sets have a defined type in their `release_id` position, and
  [7.5](07-search-ux.md)'s `/<entity>/<id>/<slug>` has no hole left in it. **The irreversibility
  stands** — from the first exported file onward, changing it means a migration plus every file
  already in a user's hands being wrong.
- ~~[8.3](08-media.md) serving WebP with no fallback ← [9.6](09-nfr.md) browser support~~ —
  **closed 2026-08-15.** The baseline is current evergreen browsers and WebP has been safe in all of
  them for years: no JPEG fallback, no second derivative, and [8.2](08-media.md)'s bucket estimate
  stands. Had it gone the other way the price was roughly a doubled bucket, which is the number
  [1.4](01-product.md) was watching.
- Seeding the vocabularies ← [section 15](15-roadmap.md). Genres, styles, credit roles, format
  descriptors and country codes (ISO plus historical `SU`, `YU`, `DD`, `CS`) all need initial
  contents, and [1.5](01-product.md) forbids fetching them, so they are hand-written seed data — our
  own list, not an extraction from anyone's database. Nobody has estimated the work; it is a real
  roadmap task, not a footnote.
  **Placed 2026-08-15, still unestimated:** [15.1](15-roadmap.md) puts it in E1.2 with the catalogue
  entities and [15.4](15-roadmap.md) carries it as risk 4, with the mitigation of starting
  deliberately small — [section 4](04-editing.md)'s moderation path exists precisely so users can
  request what a seeded list is missing.

## Needs verification against reality

Claims in the agenda that are assumptions until checked against a real file, API or document.

- [10.2.2](10b-import.md) — **partly done 2026-08-14, and left open as `[~]` on 2026-08-15.** A real
  Discogs *inventory* export was inspected (see that file's working notes). Discogs' CSV conventions
  are now known; the *collection* export's own column set is still unverified and needs a real
  collection export. **It no longer blocks anything**: [10.2.2](10b-import.md) decided the parser
  resolves columns by header name through a lookup table and treats every column as optional, so a
  wrong guess costs a table row and is surfaced by the importer's own report on the first real file.
- [10.2.7](10b-import.md) — still open, and now **closed as a decision while open as a fact**.
  Whether the collection export carries `Collection Item Instance ID` is unconfirmed (`listing_id`
  in the inventory export is weak evidence that it does). [10.2.7](10b-import.md) specifies both
  branches — instance id where present, file hash where not — so the answer changes which defence
  runs, not the schema.
- [10.2.5](10b-import.md) — the literal name of Discogs' default collection folder, which the
  importer skips rather than turning into a tag every item carries. Same file, same verification.
- [10.5.1](10e-legal-sources.md) — **checked 2026-08-15, as far as is possible without a lawyer, and
  closed on that basis.** The collection export is an official, documented Discogs feature; the
  licensing clauses in their ToS run *user → Discogs* (plus Discogs' onward licensing via the Data
  Dump and API) rather than restricting what a user does with a copy of their own collection. The
  framing that actually decides it: **their ToS is not our contract** — it binds the user, not us —
  and for an EU user, GDPR portability is a right no terms can remove. Their ToS page refuses
  automated fetching (HTTP 403), which is consistent with a service that does not want to be
  crawled, and we do not crawl it. Not lawyer-reviewed; worth ten minutes before the ToS is
  published, alongside [13.5](13-legal.md) and [13.1](13-legal.md) below.
  **The live risk was never here:** it is the EU database-rights note above, which concerns *us*
  accreting exports, and it becomes real only at [10.3.4](10c-export.md)'s dump.
- [13.5](13-legal.md) — the claim that a single-operator service of this size owes no DSA
  compliance machinery (statements of reasons, transparency reports, appeals) and needs no
  registered DMCA agent. Believed right and decided on that basis; not checked against current law,
  and worth ten minutes before the ToS is published.
- [13.1](13-legal.md) — that a ToS clause is an effective waiver of the EU sui generis database
  right in a contribution. Standard practice (Wikidata and MusicBrainz both do it), never
  lawyer-reviewed by us. The exposure is small while [10.3.4](10c-export.md) stays unpublished.

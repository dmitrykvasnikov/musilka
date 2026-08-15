# 1. Product: scope and goals

**Priority:** P0

- [x] 1.1 What is this: pet project / portfolio / commercial product / internal tool? This drives how deep the infrastructure goes and the required code quality.
      **Decision:** A **portfolio piece**. The quality bar is what a reader of the repository would
      judge: clean architecture, tests, CI, a real deploy — the code is part of the deliverable, the
      scale is not. Monetisation, payments, ads, SLA and on-call are out of every planned stage
      ([1.6](01-product.md) stays open but the default answer is "none").
      Commercial use is **not planned and not designed for**. It was considered and set aside as
      "portfolio for now": if it ever comes up it will be re-opened here, accepting that some work
      (notably re-seeding the catalogue under a different data licence, [10.5](10e-legal-sources.md))
      may have to be redone. We do not pay that insurance premium up front.
- [x] 1.2 Target audience: vinyl collectors? music lovers in general? a narrow niche (e.g. only certain genres/countries)?
      **Decision:** Collectors of **physical media of any format** — vinyl, CD, cassette, and whatever
      else people keep on a shelf. Not vinyl-only, and not "music lovers in general" (which would
      flatten the model to release-level metadata and lose the interesting part).
      Consequence for [section 2](02-catalogue-model.md): format is a **first-class, multi-format**
      structure from day one — no vinyl assumptions baked into the schema, and no format-specific
      fields promoted to the release itself. Pressing/edition-level detail must be expressible for
      every format, not just for records.
- [x] 1.3 Key value versus Discogs/MusicBrainz/RateYourMusic — why would a user come here? (language? community? UX? absence of a marketplace?)
      **Decision:** Four things, all of which fall out of decisions already taken rather than being
      aspirations:
      1. **Every physical format is a first-class citizen** ([1.2](01-product.md)) — vinyl, CD and
         cassette treated equally, rather than a record-shaped model with the rest bolted on.
      2. **No marketplace** ([1.7](01-product.md)) — a catalogue and a collection, not a shop. No
         seller ranking, no price pressure, no sale-driven listings.
      3. **Editing and history at the centre** ([1.5](01-product.md), [section 4](04-editing.md)) —
         the catalogue is what its users make of it, and every change is attributable and reversible.
      4. **Your data is never locked in** ([section 10.3](10c-export.md)) — export is a first-class
         feature, not a compliance checkbox; recommendation 9 argues it ships before import.
      Honest caveat: for a portfolio piece ([1.1](01-product.md)) this is a positioning statement,
      not a growth strategy. We are not trying to take users from Discogs.
- [x] 1.4 Expected scale: 10 / 1,000 / 100,000 users; how many releases in the catalogue at launch and after a year.
      **Decision:** Portfolio scale. **Tens of users** (order 10, not 1,000). Catalogue **empty at
      launch** ([1.5](01-product.md)) and growing only from uploaded collections — realistically a
      few thousand releases after a year, since a serious collector's export runs to hundreds or low
      thousands of rows.
      **Size the system by rows, not by users** — the two decouple here, since one collector with a
      5,000-item collection creates more catalogue rows than fifty casual users. Working estimates
      after a year, to be designed against:

      | Table | Rows |
      |---|---|
      | Collection items | ~15,000 |
      | Distinct releases | ~12,000 |
      | Artist names / label names | ~6,000 / ~3,000 |
      | Edit history | ~50,000 |
      | Tracks (only if hand-entered) | ~120,000 in theory, near zero in practice |

      The entire database fits in RAM on a 2 GB box several times over. We are two to three orders of
      magnitude away from any performance concern; if this estimate is wrong by 100× the answer is a
      bigger VPS, not a different architecture.
      **Explicit veto list.** Cite this item to reject, in [section 9](09-nfr.md) and
      [section 12](12-infrastructure.md): a dedicated search engine (Elasticsearch, Meilisearch,
      Typesense — Postgres FTS with a trigram index covers fuzzy artist matching at this size,
      recommendation 4); Redis (the job queue is Postgres-backed, sessions live in Postgres or in
      signed cookies); read replicas; a connection pooler; sharding or partitioning; a CDN; a caching
      layer; Kubernetes; autoscaling; a load balancer. Target shape: **one small VPS with Postgres on
      the same box, one process, deploys that may take the site down for ten seconds.**
      **Three things this does *not* excuse:**
      1. **Images ([section 8](08-media.md)) are the exception that scales with the catalogue rather
         than with users** — ~12,000 releases × 2 images × 400 KB ≈ 10 GB, which is real money on a
         small VPS and real time in backups. Expect this to be where the estimate breaks first;
         it argues for object storage and hard size limits from the start, not local disk.
      2. **Import must not assume small input.** One user uploading a 5,000-row CSV is the realistic
         worst case: a background job with progress from day one ([10.4.5](10d-model-requirements.md)).
      3. **Backups are an operational risk, not a capacity one.** At ten users there is no
         redundancy and losing the database loses everything. [Section 12](12-infrastructure.md) must
         still take restore seriously — and test it — despite the small numbers.
      **Why this item exists at all:** infrastructure mistakes are reversible (scale vertically, add
      a cache, move images to object storage) while schema mistakes are not. 1.4 is here to stop the
      caution budget being spent on infrastructure instead of on
      [section 2](02-catalogue-model.md).
- [x] 1.5 Is the catalogue populated only by users from scratch, or imported from external databases? (critical for the UX of the first months)
      **Decision:** **By users only.** The rule is about **how data enters the system, not about
      which fields the model may contain.** Data arrives through exactly two channels:
      1. **A file the user uploads themselves** — Discogs collection CSV first ([10.2](10b-import.md)).
      2. **A person typing into our own UI** — [section 4](04-editing.md), unbounded.
      Nothing else. Our server never calls an external music database, downloads a dump, or scrapes:
      no Discogs or MusicBrainz API, not even for enrichment of an existing entry.
      **This does not restrict the model.** Tracklists, images, credits, country and year are not
      forbidden — they are merely absent *at import*, because no uploaded file carries them. A user
      may add a tracklist by hand, and [section 8](08-media.md) exists so users can upload cover
      images. What we will never do is fetch those fields automatically.
      Note that one import writes to **both** halves of the model: artist/title/label/catno/format/
      release_id are catalogue facts ([section 2](02-catalogue-model.md)), while condition, notes,
      folder, rating and date-added are facts about *this user's copy* ([section 3](03-collection.md)).
      That split is why [10.2.3](10b-import.md) is a schema question rather than an importer detail.
      Rationale: sidesteps the licensing problem of bulk extraction (one person's collection is not a
      substantial part of anyone's database), removes the cold-start problem without a dependency on
      anybody's API or rate limits, and makes [section 4](04-editing.md) — editing with versioning —
      the centre of the product rather than a side feature, which suits a portfolio piece
      ([1.1](01-product.md)).
      Consequences: the catalogue is **thin by construction** at first, so the editing and merge UX
      carries the product ([section 4](04-editing.md)); no external-service availability, quota or
      ToS sits on the critical path ([10.5](10e-legal-sources.md) shrinks to almost nothing); we
      still store external IDs for deduplication ([10.4.1](10d-model-requirements.md)), we simply
      never dereference them.
      Residual risk, logged in NOTES.md: accreting many users' exports into one public catalogue is a
      grey area under EU database rights (repeated extraction of insubstantial parts). Acceptable at
      portfolio scale.
- [x] 1.6 Is monetisation planned at all? (ads, subscription, donations, none)
      **Decision:** **None.** No ads, no subscription, no donations, at any stage. Follows directly
      from [1.1](01-product.md). Consequence: no billing, no payment provider, no plan/entitlement
      concept anywhere in the model, and no analytics collected for revenue purposes.
- [x] 1.7 Marketplace (selling/trading records) — in scope, out of scope, or "maybe later"? Affects the collection data model.
      **Decision:** **Out of scope, permanently.** No selling, trading, orders, payments or disputes
      at any stage. For a portfolio project ([1.1](01-product.md)) it is pure surface area with no
      payoff, and its absence is a cleaner story than a half-built store.
      Consequence for [section 3](03-collection.md): a collection item may carry **condition and
      notes** (useful for the owner's own records, and present in the Discogs CSV we import from —
      [10.2](10b-import.md)), but never price, availability, seller state or anything order-shaped.
- [x] 1.8 Interface languages: Russian only / English only / i18n from day one.
      **Decision:** **English only.** Revised on review 2026-08-14 — an earlier draft of this item
      said "Russian and English from day one", and that was reversed: with no framework to lean on
      ([1.12](01-product.md)) the i18n machinery is hand-built, and shipping every feature's copy
      twice is a permanent tax on a solo project. English also means anyone assessing the portfolio
      piece ([1.1](01-product.md)) can use it.
      A second language is not designed for and not planned. If it is ever wanted, retrofitting
      server-rendered templates is real work but bounded, and that cost is accepted in exchange for
      moving faster now. Do **not** pre-build an abstraction "just in case" — no message-key
      indirection, no locale plumbing, no translation files. Plain English strings in templates.
      **This applies to the interface only, and the distinction still matters.** "English only" is a
      statement about chrome — labels, buttons, navigation, errors, emails. It says nothing about
      catalogue data: artist names, release titles and label names are recorded **as they appear on
      the object**, in whatever script that is, and are never translated or transliterated by the
      application. An English UI over a release credited to `Кино` is correct and expected.
      **Still raised for [section 2](02-catalogue-model.md):** the multi-script question survives
      this decision unchanged, because it comes from the *data*, not from the UI language — the same
      artist may appear as `Кино` on one release and `Kino` on another, and users will search for
      both. That is a catalogue modelling problem (name variants and aliases), not an i18n one, and
      it must not be solved with translation strings. See NOTES.md.
- [x] 1.9 What goes into the MVP (first working release) and what is explicitly deferred. State it in one paragraph.
      **Decision, in the one paragraph the item asks for:** the first working release is **a
      collector's round trip**. You sign up, upload your Discogs collection export, and get a
      browsable catalogue of what you actually own; you correct it by hand, with every change
      attributable and reversible; you add cover images; you search it; and you export the lot back
      out whenever you like. Everything that is not on that path is deferred.
      **In:** accounts and sessions ([section 6](06-accounts.md)); the catalogue entities of
      [section 2](02-catalogue-model.md) with hand editing and history
      ([section 4](04-editing.md)); collection and wantlist ([section 3](03-collection.md)); Discogs
      CSV import as a background job ([10.2](10b-import.md), [10.4.5](10d-model-requirements.md));
      export ([10.3](10c-export.md)); search over Postgres FTS with a trigram index
      ([section 7](07-search-ux.md), [1.4](01-product.md)); release images, capped at eight, in
      object storage ([2.4.8](02-catalogue-model.md), [section 8](08-media.md)).
      **Explicitly out, and this list is the useful half of the item:** messaging
      ([section 5](05-messaging.md)) — at tens of users ([1.4](01-product.md)) there is nobody to
      message, and a mailbox is a whole vertical; the public API ([10.1](10a-public-api.md)); any
      moderation or voting queue ([section 4](04-editing.md) ships attribution and reversal, not a
      review workflow); artist photos ([2.2.6](02-catalogue-model.md)); the companies UI
      ([2.4.6](02-catalogue-model.md)); track-level role credits
      ([2.4.5](02-catalogue-model.md)); membership editing ([2.2.2](02-catalogue-model.md)). The last
      four exist in the schema already and cost nothing to leave switched off.
      **Three things must nonetheless be built in E0–E1 even though the features they serve are
      later**, per recommendation 7 — retrofitting them is the most expensive rework available: the
      service layer through which all mutations pass ([10.4.4](10d-model-requirements.md)), the
      external-ID table with provenance ([10.4.1](10d-model-requirements.md)), and the
      Postgres-backed job queue ([10.4.5](10d-model-requirements.md)).
      **Two sequencing constraints, both already decided elsewhere:** export ships **before** import
      (recommendation 9 — it needs no external format knowledge and delivers
      [1.3](01-product.md)'s "not locked in" promise immediately), and **master merge is early, not
      late** ([2.1.2](02-catalogue-model.md) — the importer mints a master per row, so merge is how
      the catalogue becomes correct at all, not a moderation nicety).
      **Where this will hurt first:** images. [1.4](01-product.md) already predicts they are the item
      that breaks the size estimate, and including them in the MVP accepts that cost knowingly rather
      than discovering it.
- [x] 1.10 MVP success criterion: what has to happen for us to call it a success.
      **Decision: two criteria, both of which must hold.**
      **1. The round trip works on real data.** A real Discogs export of a real collection imports;
      every release is reachable and nothing is orphaned; twenty of them get corrected by hand with
      the history intact; the whole collection exports; and **re-importing that export lands in the
      same state, with no duplicates**. This is deliberately pass/fail on real input rather than a
      judgement call, and it exercises all four value propositions of [1.3](01-product.md) in one
      pass. The re-import step is the sharp end — it is what proves external IDs
      ([2.5.4](02-catalogue-model.md)) and duplicate handling
      ([2.4.9](02-catalogue-model.md)) actually work.
      **2. ~~A second person uses it.~~ — WITHDRAWN 2026-08-15, see the amendment below.** Someone
      who is not the author signs up unaided, uploads their own export, finds and fixes a wrong
      release, and comes back. Flows survive contact with a stranger, or they do not work.
      **Honest caveat on the second:** it depends on finding a willing collector, which is outside
      the author's control. It is the one criterion that circumstance rather than unfinished work can
      block, and if it stalls, that is worth distinguishing from failure.

      **Amendment 2026-08-15 — criterion 2 is withdrawn, and success is criterion 1 alone.**
      Requested by the author, who wants to build as much as possible on localhost. The caveat above
      already identified this as the one criterion that circumstance rather than unfinished work
      could block; withdrawing it removes a dependency on a person nobody had found, rather than
      lowering the bar on anything we control.
      **What this does *not* change, which is most of it.** Criterion 1 stands untouched and is
      still pass/fail on a real Discogs export — it needs [10.2](10b-import.md)'s converter and it is
      still the thing [11.10](11-stack.md) turns into a property test, so the whole of E1 keeps its
      shape. [1.1](01-product.md)'s standing constraint is likewise untouched: clean architecture,
      tests, CI and a repository a stranger can read still apply to every commit. **The portfolio
      artifact was always the repository, and it survives intact.**
      **What it does change, stated so nobody rediscovers it as an inconsistency.** A **public
      deployment is now deferred rather than gating** — it is still wanted, but nothing in E0 or E1
      waits on it. Four consequences follow: [12.5](12-infrastructure.md)'s and
      [13.6](13-legal.md)'s "give the domain reputation time before the stranger is invited" loses
      its deadline (the work does not go away — it moves behind the deployment);
      [15.4](15-roadmap.md)'s risk 8 disappears entirely; [7.8](07-search-ux.md)'s indexability
      keeps its other justification but loses this one; and [13.3](13-legal.md)'s privacy policy and
      ToS become **due before the first user who is not the author**, not before the first release,
      because with one user there is no other party's personal data being processed.
      **The one thing that would be dishonest to leave implied:** localhost cannot demonstrate
      [9.x](09-nfr.md)'s operational half — TLS, deliverability, an off-box backup, an external
      pinger. Those stay real work, marked as such, and are not quietly reclassified as done.
      **Why "judged as code" is not on this list, having been considered:** it is not a criterion but
      a **standing constraint**. [1.1](01-product.md) already sets the bar — clean architecture,
      tests, CI, a real deploy — and it applies to every commit from the first one. Restating it here
      would recast a continuous obligation as an event to reach at the end, which is exactly how that
      bar gets missed.
      **What is explicitly *not* a success criterion:** user counts, catalogue size, uptime,
      engagement or revenue. [1.4](01-product.md) and [1.6](01-product.md) already rule out the
      product ambitions those would measure, and importing them here would quietly reopen decisions
      taken deliberately.
- [x] 1.11 Timeline and resources: are you working alone? how many hours per week? is there a deadline.
      **Decision:** Solo, **2–3 hours per day** (roughly 15–20 hours a week), **no deadline**.
      This is what makes [1.12](01-product.md)'s choice of Haskell affordable: the accepted 2–3×
      multiplier over a Django/Rails build is a real cost, but with no date to hit and a steady daily
      rhythm it buys depth rather than risking the project.
      Consequences for the roadmap ([section 15](15-roadmap.md)): stages are sized in weeks, not
      days; a daily cadence means tasks should be completable in one or two sittings, so the plan
      favours many small tasks over few large ones; and since nothing is date-driven, scope is cut by
      *interest and value* rather than by deadline pressure.
- [x] 1.12 Your experience/preferences regarding the stack — what you already know well, what you want to try/learn on this project. (feeds [11.2](11-stack.md))
      **Decision (input to [11.2](11-stack.md), which stays open until sections 2–5 are closed):**
      **Preference: Haskell.** Explicitly *not* a batteries-included framework in the
      Django/Rails mould — that style was considered and rejected as unappealing to work in,
      which for a project done in spare time is a real constraint rather than a matter of taste.
      Elixir/Phoenix was the other candidate and remains the fallback if Haskell proves too slow
      going ([11.2](11-stack.md)).
      **Existing level:** comfortable with the language itself — full Advent of Code, parsing tools,
      a small language interpreter. So: ADTs, recursion schemes, monads and parser combinators are
      known ground.
      **Known gaps, to be budgeted for in the roadmap rather than discovered:** application
      architecture in the large (settle on `ReaderT AppEnv IO` immediately; effect systems are out
      of scope), the database layer (no prior equivalent), Servant's type-level error messages, and
      build/deploy tooling.
      **Why it fits this project rather than being an indulgence:** the centre of gravity here is
      domain modelling and transformation — multi-format releases as sum types
      ([1.2](01-product.md)), edit history as immutable values ([4.2](04-editing.md)), and CSV
      import/merge as parsing plus reconciliation ([10.2](10b-import.md)). That is Haskell's
      strongest ground, and it is also the part of the codebase a reader of a portfolio piece would
      actually judge ([1.1](01-product.md)).
      **Accepted cost:** roughly 2–3× the calendar time of an equivalent Django/Rails build, and
      hand-building sessions, migrations tooling, the moderation UI, uploads and image handling
      ([section 8](08-media.md)). Loss of a free admin UI is not the blow it first appears —
      catalogue editing is a user-facing feature here ([section 4](04-editing.md)), so a real
      editing UI was always going to be built.

## Working notes

_(none yet)_

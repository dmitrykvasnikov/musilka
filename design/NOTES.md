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
- **2026-08-15 — Curated set, with free text beside it rather than inside it.** Every field drawn
  from a vocabulary (genres and styles, format descriptors, identifier types, credit roles, company
  roles) is seeded and closed, with a free-text companion field for what the vocabulary cannot say.
  Users request new values through moderation; they never mint them inline. The reason holds
  wherever it recurs: the field exists to group or filter, free text destroys grouping, and a closed
  set with no escape hatch makes users lie. Established across
  [2.3.2](02-catalogue-model.md), [2.4.2](02-catalogue-model.md), [2.4.3](02-catalogue-model.md),
  [2.4.5](02-catalogue-model.md), [2.4.6](02-catalogue-model.md).

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
- **2026-08-15 — `Series` as an entity.** Turned down at [2.1.5](02-catalogue-model.md) — real on
  real objects, but a mostly-empty table with a moderation cost at ~12,000 releases, and expressible
  as a format descriptor or a note. Additive if ever reopened.

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
- **2026-08-14 — EU database rights cover repeated extraction of insubstantial parts.** A single
  user's collection export is not a substantial part of anyone's database, so one upload is safe. But
  the Database Directive also bars *repeated and systematic* extraction of insubstantial parts, and
  accreting many users' exports into one public catalogue is arguably that. Accepted as a residual
  risk at portfolio scale ([1.5](01-product.md)); it would need real legal review before any public
  launch at volume or any commercial turn. Not verified against current case law.

## Open questions blocking other items

Questions that must be answered before some other item can be closed. Format: what is blocked ← what we are waiting on.

- [1.9](01-product.md) MVP scope and [1.10](01-product.md) success criterion ←
  **nothing in section 2 any more.** Section 2 closed 2026-08-15 (all 30 items), so the reason these
  two were held is gone: the model's size is known, and [2.5.5](02-catalogue-model.md) has settled
  that CSV import is in the MVP. Section 2 also handed 1.9 two exclusions to fold in — artist images
  are out ([2.2.6](02-catalogue-model.md)) and companies are schema-only, not UI
  ([2.4.6](02-catalogue-model.md)). **These are the two oldest open items in the agenda and are now
  answerable.**
- ~~Multi-script artist and title names (`Кино` / `Kino`)~~ — **closed 2026-08-15** at
  [2.2.1](02-catalogue-model.md): one artist, many name rows, one search index, no transliteration
  anywhere. Kept here only as a pointer, since [1.8](01-product.md) and several notes refer to it as
  an open question. It is now an invariant above.
- [11.2](11-stack.md) final stack confirmation ← sections 3–5, per that file's own priority note.
  Section 2 is no longer part of this. [1.12](01-product.md) records the preference (Haskell) and
  the fallback (Elixir/Phoenix); the binding choice waits until the remaining model sections are
  known. Section 2's answer on size is in: four entities, no global track table, vocabularies as
  seeded lookups, and an ordered artist credit in three parallel tables.
- Soft delete and merge semantics ← [section 4](04-editing.md). [2.4.9](02-catalogue-model.md)
  requires a merged-away release to keep resolving and a collection item never to dangle, but whether
  that is `merged_into` alone, a `deleted_at`, or a consequence of edit versioning is section 4's to
  decide. Flagged so it is not discovered during implementation.
- Seeding the vocabularies ← [section 15](15-roadmap.md). Genres, styles, credit roles, format
  descriptors and country codes (ISO plus historical `SU`, `YU`, `DD`, `CS`) all need initial
  contents, and [1.5](01-product.md) forbids fetching them, so they are hand-written seed data — our
  own list, not an extraction from anyone's database. Nobody has estimated the work; it is a real
  roadmap task, not a footnote.

## Needs verification against reality

Claims in the agenda that are assumptions until checked against a real file, API or document.

- [10.2.2](10b-import.md) — **partly done 2026-08-14.** A real Discogs *inventory* export was
  inspected (see that file's working notes). Discogs' CSV conventions are now known; the
  *collection* export's own column set is still unverified and needs a real collection export.
- [10.2.7](10b-import.md) — still open. The inventory export carries `listing_id`; whether the
  collection export carries `Collection Item Instance ID` is unconfirmed.
- [10.5.1](10e-legal-sources.md) — narrowed by [1.5](01-product.md): we never call the Discogs API,
  so its rate limits are moot. What remains to check is whether Discogs' ToS says anything about
  what a user may do with their **own** export.

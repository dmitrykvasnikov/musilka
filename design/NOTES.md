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

- [1.9](01-product.md) MVP scope and [1.10](01-product.md) success criterion ← [section 2](02-catalogue-model.md).
  Deliberately left open: we cannot say what the first release contains before we know how large the
  catalogue model turns out to be. Everything else in section 1 is closed.
- Multi-script artist and title names (`Кино` / `Kino`) ← [section 2](02-catalogue-model.md).
  Forced by the data rather than by the UI language, so it survived the reversal of
  [1.8](01-product.md) to English-only. Needs a name-variant/alias model on the catalogue entities;
  must not be solved with translation strings.
- [11.2](11-stack.md) final stack confirmation ← sections 2–5, per that file's own priority note.
  [1.12](01-product.md) records the preference (Haskell) and the fallback (Elixir/Phoenix); the
  binding choice waits until the model's size is known.

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

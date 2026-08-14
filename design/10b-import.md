# 10.2 Importing a user's collection (Discogs first and foremost)

**Priority:** P0 — build it into the architecture from the start ([why](10d-model-requirements.md#why-section-10-is-p0))
**Siblings:** [10.1 public API](10a-public-api.md) · [10.3 export](10c-export.md) · [10.4 model requirements](10d-model-requirements.md) · [10.5 legal](10e-legal-sources.md)

- [ ] 10.2.1 Sources in priority order: Discogs collection CSV export, Discogs wantlist CSV, the Discogs API via the user's OAuth (live sync), our own JSON, an arbitrary CSV with manual column mapping.
- [ ] 10.2.2 Verify the actual Discogs export format (before implementation — on a real file). Approximate column set: `Catalog#, Artist, Title, Label, Format, Rated, Released, release_id, CollectionFolder, Date Added, Collection Media Condition, Collection Sleeve Condition, Collection Notes` + user fields. The key fact: **there is almost no release metadata there, but there is a `release_id`**.
- [ ] 10.2.3 The main import question: what do we do if no release with that `discogs_id` exists in our catalogue?
      (a) create a "stub" from the crumbs available in the CSV; (b) pull metadata from the Discogs API/dump; (c) leave the collection item **unresolved** and ask the user to link it manually; (d) a combination. The answer determines whether we allow a collection item with no link to a catalogue release (see [10.4.3](10d-model-requirements.md)) — and that is already a DB-schema question.
- [ ] 10.2.4 Matching, in decreasing order of reliability: `discogs_id` → barcode → catalogue number + label → fuzzy match (artist + title + year + format). Do we need a match-review screen before applying?
- [ ] 10.2.5 Vocabulary mapping: Goldmine grading (`Mint (M)`, `Near Mint (NM or M-)`, …) → our condition codes; Discogs folders → our folders/tags ([3.3](03-collection.md)); rating 1–5; Discogs formats → our format structure ([2.4.2](02-catalogue-model.md)).
- [ ] 10.2.6 The import process as an entity: uploaded file → job → report/preview → apply. Do we keep the source file and the matching log? Can an import be **rolled back as a whole**?
- [ ] 10.2.7 Idempotency: how do we avoid duplicating on a repeated import of the same file or on regular syncing. What is the deduplication key? (On Discogs a collection instance has an `instance_id`, but it is usually absent from the CSV — verify.)
- [ ] 10.2.8 Volumes: a 10,000-item collection means a background job with progress (see [11.7](11-stack.md)), a file size limit, timeouts.
- [ ] 10.2.9 Error handling: is partial success acceptable? A "N rows unrecognised" report + exporting the problematic rows back to CSV for manual fixing.
- [ ] 10.2.10 Importing the **catalogue** (not a collection) — is that an admin/offline operation ([section 2.5](02-catalogue-model.md)) rather than a user feature? Let's separate the two paths explicitly.
- [ ] 10.2.11 Two-way sync with Discogs (our changes → theirs) — do we declare it out of scope right away, or keep the option open?

## Working notes

**2026-08-14 — verified against a real Discogs export.** A genuine export was inspected (18 rows,
CD-heavy) and then deleted; only structural facts are recorded here, no collection contents. The file
was an **inventory** (marketplace listings) export, *not* a collection export — so this settles the
shape of Discogs' CSV conventions but leaves the collection-specific columns still unverified.

Header seen:

```
listing_id, artist, title, label, catno, format, release_id, status, price, listed,
comments, media_condition, sleeve_condition, accept_offer, external_id, weight,
format_quantity, location, quantity
```

Findings that change the design:

1. **`recommendations.md` point 8 is wrong, and in our favour.** The export is *not* "essentially a
   list of `release_id`s without release metadata". Every row carried `artist`, `title`, `label`,
   `catno`, `format` and `release_id`. A stub built from a row is a recognisable release, not an
   empty shell. This materially improves option (a) in [10.2.3](10b-import.md) and is what makes
   [1.5](01-product.md)'s "no external metadata" stance survivable at all.
2. **What is still missing** from a row: no tracklist, no artist or label identifier (names only, as
   free text), no images, no genre/style, no country, and no release year in this export.
3. **`label` and `catno` are comma-joined lists inside one quoted field, positionally paired.** One
   row read `"United Guttural Records, United Guttural Records"` / `"UGR 018, UG018"` — the same
   label twice with two different catalogue numbers. Naive splitting on `,` is wrong because label
   names legitimately contain commas; and the pairing must be preserved, not deduplicated.
4. **`format` is a compound Discogs descriptor string**, not a value: an optional quantity prefix, a
   media type, then free descriptors — `CD` · `CD, Album` · `2xCD, Comp` · `CD, Album, Ltd, RE, Dig`
   · `CD, Album, Dlx, FYE`. `format_quantity` mirrors the `2x` prefix. Feeds
   [2.4.2](02-catalogue-model.md) and [10.2.5](10b-import.md); note `FYE` is a retailer-specific
   descriptor, so the descriptor vocabulary is open-ended and cannot be a closed enum.
5. **`catno` has sentinel and type-confused values**: the literal string `none` where there is no
   catalogue number, and in one row a **barcode** used as the catalogue number. Both break
   [10.2.4](10b-import.md)'s matching ladder if taken at face value.
6. **Condition vocabulary confirmed as Goldmine grading with the parenthetical spelled out** —
   `Near Mint (NM or M-)`, `Very Good Plus (VG+)`, `Very Good (VG)`, `Good Plus (G+)`. Parse the
   abbreviation in the brackets, not the prose. Confirms [10.2.5](10b-import.md).
7. **Empty columns are genuinely empty, not absent** — `external_id` and `location` were empty in
   every row, `comments` in all but one. The importer must treat every column as optional.
8. **Marketplace columns exist in this export and must be discarded on sight**: `price`, `status`,
   `accept_offer`, `listed`, `quantity`, `weight`. Per [1.7](01-product.md) nothing order-shaped
   enters the model, so the importer drops them rather than storing them "just in case".

Still unverified, and needing a real **collection** export (not an inventory one):

- [10.2.2](10b-import.md) — the collection export's own column set. Expected to differ: `Released`,
  `Rating`, `CollectionFolder`, `Date Added`, `Collection Media Condition`,
  `Collection Sleeve Condition`, `Collection Notes`.
- [10.2.7](10b-import.md) — whether `Collection Item Instance ID` is present. This is the
  idempotency key question and it is still open; note that `listing_id` in the inventory export
  plays an analogous role, which is weak evidence that the collection export carries an instance id
  too.

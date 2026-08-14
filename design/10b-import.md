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

_(none yet)_

# 10.2 Importing a user's collection (Discogs first and foremost)

**Priority:** P0 — build it into the architecture from the start ([why](10d-model-requirements.md#why-section-10-is-p0))
**Siblings:** [10.1 public API](10a-public-api.md) · [10.3 export](10c-export.md) · [10.4 model requirements](10d-model-requirements.md) · [10.5 legal](10e-legal-sources.md)

- [x] 10.2.1 Sources in priority order: Discogs collection CSV export, Discogs wantlist CSV, the Discogs API via the user's OAuth (live sync), our own JSON, an arbitrary CSV with manual column mapping.
      **Decision:** **three importers, one of which is not really a second one.**
      1. **Our own JSON** ([10.3.2](10c-export.md)) — first, because it is the other half of the
         round trip [1.10](01-product.md) measures success by and [11.10](11-stack.md) tests, and it
         needs no third-party file to develop against.
      2. **Discogs collection CSV** — the one a real user actually arrives with, and the one
         [1.10](01-product.md)'s success criterion is stated on.
      3. **Discogs wantlist CSV** — the same parser writing `wantlist_item` instead of
         `collection_item` ([3.5](03-collection.md)). Nearly free once 2 exists.
      **Struck out entirely: the Discogs API via OAuth.** It is not a scope call — the two-channels
      invariant ([1.5](01-product.md)) forbids the server calling an external music database at all,
      and OAuth against a third-party account is the largest credential liability in the design,
      bought for a feature nobody has asked for.
      **Deferred: arbitrary CSV with manual column mapping.** A mapping UI is a real feature (column
      preview, per-column target, saved mappings, type coercion) and we cannot design it well from
      one example; it becomes worth building when there is a second real source, and by then we will
      know which one. Until then, "some other service's CSV" is answered by a spreadsheet and our own
      column names.
- [~] 10.2.2 Verify the actual Discogs export format (before implementation — on a real file). Approximate column set: `Catalog#, Artist, Title, Label, Format, Rated, Released, release_id, CollectionFolder, Date Added, Collection Media Condition, Collection Sleeve Condition, Collection Notes` + user fields. The key fact: **there is almost no release metadata there, but there is a `release_id`**.
      **Deferred — it is a verification task and its precondition is unmet.** A real Discogs
      *inventory* export has been inspected (working notes below) and settles their CSV conventions;
      a real *collection* export has not been seen, so the collection-specific column names remain
      guesses. Unblocked by one file from any Discogs user; until then it stays open rather than
      being closed on an assumption.
      **What the other items decided so that this one is not load-bearing:** the parser resolves
      columns **by header name through a lookup table**, never by position, and treats every column
      as optional ([finding 7](#working-notes)). A wrong guess about a column name therefore costs
      one row in a mapping table, not a rewrite — and an unrecognised column is reported to the user
      ([10.2.9](10b-import.md)) rather than ignored, which is what turns the first real file into
      the verification.
- [x] 10.2.3 The main import question: what do we do if no release with that `discogs_id` exists in our catalogue?
      (a) create a "stub" from the crumbs available in the CSV; (b) pull metadata from the Discogs API/dump; (c) leave the collection item **unresolved** and ask the user to link it manually; (d) a combination. The answer determines whether we allow a collection item with no link to a catalogue release (see [10.4.3](10d-model-requirements.md)) — and that is already a DB-schema question.
      **Decision:** **(a), always and only. Every imported row mints a stub release (and its master),
      and there is no such thing as an unresolved collection item.**
      (b) is excluded outright by [1.5](01-product.md).
      (c) is the expensive one and it is what we are actually rejecting. A nullable `release_id`
      puts an "and the resolved ones" clause into every query [section 3](03-collection.md) has —
      the statistics and breakdowns that read through to the master ([3.8](03-collection.md)), the
      privacy filters ([3.7](03-collection.md)), the merge repointing ([3.11](03-collection.md)) —
      plus a second rendering for an item with no release behind it, forever. And it is bought to
      create a triage queue of up to ten thousand rows that nobody drains; the same reasoning
      [4.3](04-editing.md) used against a vote queue applies with more force here, because these
      rows arrive all at once.
      (a) is viable **because of what the file actually carries**: artist, title, label, catalogue
      number and format on every row (finding 1 below), which is a recognisable release rather than
      an empty shell. And it costs nothing structurally: [2.1.2](02-catalogue-model.md) already
      mints a master per release and already accepts duplicate masters as the normal post-import
      state, with master merge in early scope for that reason.
      **The correction path is merge ([4.4](04-editing.md)), not triage.** That is the same shape as
      [2.4.9](02-catalogue-model.md)'s rule — duplicates are resolved afterwards, never by rejecting
      or parking a user's row.
      **What this hands to [10.4.3](10d-model-requirements.md):** `collection_item.release_id` is
      `NOT NULL`, exactly like `release.master_id`. The answer to "do we allow unresolved items" is
      no, and it is now an invariant in [NOTES.md](NOTES.md).
- [x] 10.2.4 Matching, in decreasing order of reliability: `discogs_id` → barcode → catalogue number + label → fuzzy match (artist + title + year + format). Do we need a match-review screen before applying?
      **Decision:** **one rung, not four: exact `discogs_release_id` through
      [10.4.1](10d-model-requirements.md)'s `external_ids`, and otherwise a new stub. No
      match-review screen.**
      The lower rungs are not merely weaker, they are actively harmful here, and each fails for its
      own reason.
      *Barcode* has nothing to match against: no import channel carries one (there is no barcode
      column in the export), so in a catalogue built by import the field is empty on both sides.
      *Catalogue number + label* runs straight into findings 3 and 5 — `catno` carries the literal
      sentinel `none` and, in one observed row, a barcode instead of a catalogue number, while
      `label` is a comma-joined list whose members may themselves contain commas. Matching on that
      attaches a user's copy to somebody else's release, which is **worse than a duplicate**: a
      duplicate is fixed by merge, a wrong link is silent and looks correct.
      *Fuzzy artist + title + year + format* has no year to work with in the observed export, and is
      the same failure mode with a similarity score in front of it.
      **No review screen**, and the reason is that with one exact rung there is nothing to
      adjudicate: a row either has a known `release_id` or it does not. A screen would be a
      ten-thousand-row triage bought to confirm decisions that were never ambiguous. The review that
      does exist is the **report afterwards** ([10.2.6](10b-import.md), [10.2.9](10b-import.md)).
      The importer also does **not** run [2.4.9](02-catalogue-model.md)'s advisory duplicate
      detection per row — it would flag most of them by design ([2.1.2](02-catalogue-model.md)) and
      teach the user to ignore it. Duplicates surface in the merge tooling, where somebody is
      actually looking.
- [x] 10.2.5 Vocabulary mapping: Goldmine grading (`Mint (M)`, `Near Mint (NM or M-)`, …) → our condition codes; Discogs folders → our folders/tags ([3.3](03-collection.md)); rating 1–5; Discogs formats → our format structure ([2.4.2](02-catalogue-model.md)).
      **Decision:** **four table-driven mappings, all one-way, all reporting what they could not
      map.**
      **Conditions** — parse the abbreviation inside the brackets, never the prose (finding 6):
      `Near Mint (NM or M-)` → `NM`. Straight into [3.2](03-collection.md)'s seeded ordered
      vocabulary, sleeve included with its three extra values. Unrecognised grade → `NULL` and a
      report line; both columns are nullable precisely because most rows have no grade at all.
      **Folders → tags** ([3.3](03-collection.md)), losslessly, one tag per folder. Discogs'
      default folder is skipped rather than imported: a tag every item carries groups nothing.
      (Its exact literal is part of [10.2.2](10b-import.md)'s verification.)
      **Rating** — same 1–5 integers as [3.2](03-collection.md), copied as-is; `0` means unrated on
      Discogs and becomes `NULL`.
      **Format** — decompose the compound descriptor string (finding 4): optional `2x` quantity
      prefix → quantity, first token → media type, remaining tokens → descriptors matched against
      [2.4.2](02-catalogue-model.md)'s seeded vocabulary. **Unknown descriptors go into the
      free-text companion field** that the curated-set invariant provides for exactly this, and are
      reported — `FYE` in the observed file is a retailer's abbreviation and will never be in a
      sensible vocabulary.
      **Label/catno pairing** (finding 3), which is not in the item's list but is the one that
      actually breaks: read the CSV with a real parser, then split the field's contents and pair
      positionally **only when the two splits yield equal counts**. When they do not, the pairing is
      genuinely ambiguous — take the whole field as a single label name and put the raw strings in
      the report. Guessing here silently invents a catalogue number.
      Everything unmapped or folded is reported, per the never-discard-silently invariant, which
      also covers [3.2](03-collection.md)'s discarded price columns and [3.4](03-collection.md)'s
      custom-field columns folded into the note.
- [x] 10.2.6 The import process as an entity: uploaded file → job → report/preview → apply. Do we keep the source file and the matching log? Can an import be **rolled back as a whole**?
      **Decision:** **upload → job → apply → report. The preview step is cut, the source file is not
      kept, and rollback covers the collection side only.**

      ```
      import(id, user_id, source, filename, file_bytes, sha256, status,
             rows_total, rows_applied, created_at, finished_at, report)
      collection_item.import_id   nullable FK
      wantlist_item.import_id     nullable FK
      ```

      **No preview/apply split.** It means holding a parsed ten-thousand-row intermediate somewhere
      — a staging table needing its own garbage collection, or the file again — plus a second
      confirmation most people click through without reading. It buys the ability to say no to a
      result that is already fully reversible on the side that matters.
      **The uploaded file lives in `file_bytes` on the import row and is cleared when the job
      finishes.** In Postgres because Postgres is the only datastore; on the row because the job
      needs it and nothing else does; cleared because it is the user's personal data
      ([10.5.3](10e-legal-sources.md), [13.3](13-legal.md)), they still hold the original, and a
      retained copy is a retention policy we would have to write. `sha256` survives the clearing —
      [10.2.7](10b-import.md) needs it. **This answers
      [10.4.8](10d-model-requirements.md): nothing goes to [8.2](08-media.md)'s storage, and nothing
      outlives the job.**
      **The matching log is the report**, a `jsonb` document holding only rows that need one —
      skipped, folded, dropped, unmapped, unparsed — with row numbers. Ten thousand `ok` entries are
      a log nobody reads and a column nobody should pay to store.
      **Rollback is one button and it removes the collection and wantlist rows this import created**
      (`DELETE ... WHERE import_id = ?`, which is [3.10](03-collection.md)'s ordinary hard delete of
      the user's own rows). **The stubs it minted into the catalogue stay.** They are catalogue
      entities other people may already reference, and [4.5](04-editing.md) makes deletion
      moderator-only and soft in any case. The button says so — an honest "your items are gone, the
      releases remain and may be merged" beats a promise we cannot keep.
- [x] 10.2.7 Idempotency: how do we avoid duplicating on a repeated import of the same file or on regular syncing. What is the deduplication key? (On Discogs a collection instance has an `instance_id`, but it is usually absent from the CSV — verify.)
      **Decision:** **two defences, because the good key may not be in the file.**
      Note first what makes this hard and is easy to forget: [3.1](03-collection.md) deliberately
      allows several copies of the same release, so "this user already owns this release" is **not**
      a duplicate signal. Row-level idempotency is impossible without a stable per-instance
      identifier.
      1. **When the file carries an instance id** — `collection_item.source` +
         `source_instance_id`, unique per user. A re-import then **skips** rows it has already
         created and reports the count. Skip rather than update: updating would let a re-upload
         silently revert edits the user made in our UI afterwards, which is a worse failure than a
         no-op and an invisible one.
      2. **When it does not** — a `sha256` of the uploaded bytes, per user. Uploading a file already
         imported is refused with "you imported this file on <date>; it created N items", a link to
         that import, and an explicit override for the user who means it. It costs one column and
         catches the mistake that actually happens: the double submit and the re-upload after a
         browser back.
      **What we do not pretend to support, stated plainly on the upload page:** without an instance
      id there is no incremental re-import. A user who adds twenty records on Discogs and re-exports
      cannot merge that file into what they already have — they import into an empty collection, or
      they roll back ([10.2.6](10b-import.md)) and import afresh. Saying so is better than a
      heuristic that silently duplicates ten thousand rows.
      Whether defence 1 ever activates is [10.2.2](10b-import.md)'s verification question; the design
      accommodates both answers, which is why this item can close while that one cannot.
- [x] 10.2.8 Volumes: a 10,000-item collection means a background job with progress (see [11.7](11-stack.md)), a file size limit, timeouts.
      **Decision:** **a background job on [11.7](11-stack.md)'s queue table, 10 MB and 20,000 rows as
      hard limits, a streaming parse, and a progress counter on a page the user reloads themselves.**
      The limits are checked before anything is applied and are refusals, not truncations —
      truncating a user's collection at row 20,000 is the silent data loss the whole section exists
      to avoid. They are generous against [1.4](01-product.md)'s volumes and exist to bound the job,
      not to ration.
      **Streaming parse**, never the whole file in memory: a 10 MB CSV read incrementally, rows
      applied in batches with a commit per batch. Not one transaction for the file — a transaction
      that fails on row 9,999 discards the other 9,998 and holds locks for minutes, and
      [10.2.9](10b-import.md) accepts partial success anyway.
      `rows_applied` is written every few hundred rows so the import page can show progress.
      **The page does not refresh itself** — no polling loop, no SSE, no push, per the no-real-time
      invariant, which is not worth reopening for a progress bar on a once-per-account event. The
      user reloads. No completion email either: that is a fifth transactional template
      ([section 12](12-infrastructure.md) inherited four) for a job that takes a minute.
      **Wall-clock cap on the job** (15 minutes): past it the import is marked failed, what was
      applied stays and is reported, and rollback is available. A job that hangs must end in a state
      the user can see and act on.
      **Restartability, which [11.7](11-stack.md) requires of every job and which this one gets
      almost for free.** The worker runs in-process, so a deploy kills a running import mid-file. On
      restart the job re-reads `file_bytes` — still on the row, since it is cleared only on
      completion ([10.2.6](10b-import.md)) — skips `rows_applied` rows and continues. That is why
      the counter is persisted per batch rather than held in memory: it is the resume point first
      and a progress display second.
- [x] 10.2.9 Error handling: is partial success acceptable? A "N rows unrecognised" report + exporting the problematic rows back to CSV for manual fixing.
      **Decision:** **yes — partial success is the normal outcome, and the unit of failure is the
      row.** Three bad rows must not cost the other 9,997; a collector's file is thirty years of
      typing and an all-or-nothing importer teaches them to distrust it.
      A row that cannot be parsed or applied is skipped and recorded with its number and its raw
      text. **The failed rows are downloadable as a CSV in the original column layout** — fix in a
      spreadsheet, re-upload just those — which costs nothing because [10.3](10c-export.md) already
      built the CSV writer. That the export machinery serves the importer is a small argument for
      [1.9](01-product.md)'s ordering being right.
      **Whole-file failure only before anything is applied**, and only for: not CSV at all, no
      recognisable header, over the size or row limit, an already-imported hash without the override
      ([10.2.7](10b-import.md)).
      The report is the section's contract with the user and lists, per category with counts:
      rows applied · rows skipped and why · stubs created · columns ignored entirely
      (marketplace columns per finding 8, [3.2](03-collection.md)'s price) · values folded into the
      note ([3.4](03-collection.md)) · unmapped conditions and format descriptors
      ([10.2.5](10b-import.md)) · ambiguous label/catno fields. Every one of those categories exists
      because some invariant says the user must be told rather than surprised later.
- [x] 10.2.10 Importing the **catalogue** (not a collection) — is that an admin/offline operation ([section 2.5](02-catalogue-model.md)) rather than a user feature? Let's separate the two paths explicitly.
      **Decision:** **there is no catalogue import path at all — not as a user feature, not as an
      admin one, not offline.** The two paths are separated by one of them not existing.
      This is [1.5](01-product.md) restated at the operational level: an administrator loading a
      dump with `psql` is precisely the act the invariant forbids, and the shell it happens in does
      not change what it is. Catalogue rows appear in exactly two ways — a user's collection import
      minting stubs ([10.2.3](10b-import.md)) and hand editing
      ([section 4](04-editing.md)) — and that constraint is the reason the editing features are
      worth building at all.
      **One thing that looks like an exception and is not: vocabulary seed data.** Genres, styles,
      credit roles, format descriptors and country codes are our own hand-written lists loaded by a
      migration ([11.8](11-stack.md)), already tracked as a roadmap task in [NOTES.md](NOTES.md).
      A migration carrying a list we wrote is not an import channel.
- [x] 10.2.11 Two-way sync with Discogs (our changes → theirs) — do we declare it out of scope right away, or keep the option open?
      **Decision:** **out of scope permanently, and it is not this section's call to make.**
      Writing to Discogs means calling Discogs, which [1.5](01-product.md) forbids for reads already;
      it would additionally need write-scoped credentials to a user's account at another service —
      the single largest thing we could hold and the one with no upside we can name. Nothing is kept
      open, no column is reserved, and the option does not need one: it would be a new service
      talking to a new API, not a change to this schema.
      Import stays what its name says — a one-way door in, with [10.3](10c-export.md) as the door
      out.

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

**2026-08-15 — what closing this section hands to [10.4](10d-model-requirements.md).** All four are
now determined rather than open, and none of them should be re-argued there:

- **10.4.3** — no unresolved items; `collection_item.release_id` is `NOT NULL`
  ([10.2.3](10b-import.md)).
- **10.4.8** — the uploaded file lives in a `bytea` column on the `import` row and is cleared when
  the job ends; nothing reaches [8.2](08-media.md)'s storage and nothing has a retention policy
  ([10.2.6](10b-import.md)).
- **10.4.5** — the queue is needed, and **import is its only justification**;
  [10.3.3](10c-export.md) declined it for export.
- **10.4.1** — the importer writes an `external_ids` row per stub (`discogs`, `release`, the id),
  and both [10.2.4](10b-import.md)'s matching and [10.3.2](10c-export.md)'s `discogs_release_id`
  column read it. The table is now load-bearing rather than speculative.

What is left genuinely open in 10.4 is provenance marking (10.4.2), public identifier shape (10.4.6)
and change-log reuse (10.4.7) — of which 10.4.6 is the urgent one, since
[10.3.2](10c-export.md) puts whatever we choose into files we cannot recall.

**2026-08-15 — the two schema additions this section makes outside its own table.**
`import_id` on `collection_item` and `wantlist_item` (nullable, for rollback), and
`source` + `source_instance_id` on `collection_item` (nullable, for idempotency). Both are additive
and neither is read by anything except the importer, which is the property that lets
[10.2.7](10b-import.md)'s unverified key question stay open without blocking the schema.

**2026-08-15 — noted and deliberately not decided: progress without a refresh.**
[10.2.8](10b-import.md) leaves the import page static because the no-real-time invariant is
system-wide and a progress bar is a poor reason to reopen it. If it ever proves genuinely bad in
use, the smallest possible change is a `<meta http-equiv="refresh">` on that one page — which holds
no connection and runs no script, and is arguably outside what the invariant refuses. That argument
belongs at [5.3](05-messaging.md)/[9.x](09-nfr.md) where the invariant lives, not here, and it should
be made explicitly rather than smuggled in with an implementation.

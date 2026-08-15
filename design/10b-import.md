# 10.2 Importing a user's collection (our own format, with converters in front)

**Priority:** P0 — build it into the architecture from the start ([why](10d-model-requirements.md#why-section-10-is-p0))
**Siblings:** [10.1 public API](10a-public-api.md) · [10.3 export](10c-export.md) · [10.4 model requirements](10d-model-requirements.md) · [10.5 legal](10e-legal-sources.md)

**Restructured 2026-08-15.** The section originally treated the Discogs CSV as the importer's
subject; it is now a **converter in front of an importer that knows only our own format**. The
decisions below are largely the same decisions — what changed is which side of that boundary each
one lives on, and every item records where it landed.

- [x] 10.2.1 Sources in priority order: Discogs collection CSV export, Discogs wantlist CSV, the Discogs API via the user's OAuth (live sync), our own JSON, an arbitrary CSV with manual column mapping.
      **Decision (revised 2026-08-15): one importer, over our own format, with converters in front of
      it. Not three importers.**

      ```
      uploaded file → detect → converter (pure) → our export format → importer (job) → rows
                                    ↑                                      ↑
                            Discogs CSV, and any                the only thing that
                            later source                        touches the database
      ```

      **The importer reads exactly one format: the one [10.3](10c-export.md) writes.** Everything
      third-party-shaped — comma-packed label fields, Goldmine prose, folder names, the literal
      sentinel `none`, retailer-specific format descriptors — lives in a pure converter in
      `musilka-domain` and **cannot reach the database**.
      **What makes this fit with nothing invented, and it is the reason to do it this way:
      [10.3.5](10c-export.md) already denormalises the catalogue crumbs into each exported row**
      (artist, title, label, catno, format, country, year) **and [10.3.2](10c-export.md) already
      carries `discogs_release_id`**. A converted Discogs row is therefore expressible in our own
      export format exactly as it stands — same fields, same meanings — so there is no third
      interchange format to specify, version or test. **The converter emits the bytes our exporter
      emits**, which means "convert then import" is provably the same path as "import a file we
      wrote", and [11.10](11-stack.md)'s round-trip property covers it.
      **What this buys, in order of weight:**
      1. **The importer is testable on day one with no third-party file.** It is the other half of
         the round trip [1.10](01-product.md) measures success by.
      2. **[10.2.2](10b-import.md)'s unverified column set stops being load-bearing.** A wrong guess
         breaks one pure module with property tests, not the importer, the job, the report or the
         schema.
      3. **A second source is a new converter** — no schema change, no job change, no report change.
         MusicBrainz, a spreadsheet, another service: same shape.
      4. The converter is pure, so it can later be exposed as a standalone "convert my file" page
         needing no account and touching no data. Not built; available.
      **What it costs, stated rather than glossed:** one more moving part than parsing Discogs CSV
      straight into rows, and an obligation — the converter's findings must survive the boundary, or
      the never-discard-silently invariant breaks at the seam. So a converter returns **rows *and*
      findings**, and they merge into [10.2.9](10b-import.md)'s single report. The user uploads once
      and reads one report; the layering is ours, not theirs.
      **Format detection is by header signature**, not by asking: our own export is recognised by its
      column names or its JSON `format_version`, a Discogs export by its. An unrecognised file is
      refused **before anything is applied** ([10.2.9](10b-import.md)) with a message naming what we
      support — that refusal is also where a mapping UI would attach if one is ever built.
      **The converters that exist: Discogs collection CSV and Discogs wantlist CSV.** The wantlist is
      nearly free once the collection converter exists — the same field mapping writing
      [3.5](03-collection.md)'s row instead. **Both are in the MVP and neither is optional**: our own
      format only exists after someone has used the site, so on day one the converter is the *only*
      entry point and [1.9](01-product.md)'s round trip has no start without it.
      **Struck out entirely: the Discogs API via OAuth.** It is not a scope call — the two-channels
      invariant ([1.5](01-product.md)) forbids the server calling an external music database at all,
      and OAuth against a third-party account is the largest credential liability in the design,
      bought for a feature nobody has asked for.
      **Deferred: arbitrary CSV with manual column mapping.** A mapping UI is a real feature (column
      preview, per-column target, saved mappings, type coercion) and we cannot design it well from
      one example. It is now **strictly less necessary** than it was: the answer to "some other
      service's CSV" is a spreadsheet producing our column names — which is a hand-written converter
      — and if the source is common enough to be worth automating, it is worth a real converter
      rather than a UI that makes every user do the mapping themselves.
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
      **Narrowed further 2026-08-15 by [10.2.1](10b-import.md)'s restructuring:** every guess on this
      list now lives **inside the Discogs converter**, a pure function with property tests and no
      database access. The blast radius of being wrong is one module. This item no longer touches the
      importer, the job, the report machinery or the schema, and it is the last `[~]` in the design
      that describes something we have not seen.
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
      **Generalised 2026-08-15:** the question is no longer Discogs-specific and neither is the
      answer. **Any row that arrives without an identifier we can resolve mints a stub from the
      denormalised crumbs in our own format** ([10.3.5](10c-export.md) puts them there). That is one
      rule for our files and every converter's output alike, and it is [10.2.4](10b-import.md)'s
      third rung.
- [x] 10.2.4 Matching, in decreasing order of reliability: `discogs_id` → barcode → catalogue number + label → fuzzy match (artist + title + year + format). Do we need a match-review screen before applying?
      **Decision (generalised 2026-08-15): three exact rungs and no inexact ones. No match-review
      screen.**
      | Rung | Key | Filled by |
      |---|---|---|
      | 1 | our own `release_id`, following `merged_into` | our own export files |
      | 2 | `external_id` ([10.4.1](10d-model-requirements.md)) — `discogs` today | a converter, or a hand-entered id |
      | 3 | nothing — mint a stub ([10.2.3](10b-import.md)) | everything else |
      This is [10.4.6](10d-model-requirements.md)'s re-import resolution order, and it is the same
      ladder for our own files and for converted ones — **a converted Discogs row simply arrives with
      rung 1 empty and rung 2 filled**. Nothing about the importer knows which happened.
      **The original decision — one rung, exact `discogs_release_id`, otherwise a stub — is
      unchanged in substance**; what the restructuring added is the rung above it, which exists
      because our own export now carries our own id.
      The lower rungs from the item's list are not merely weaker, they are actively harmful here, and
      each fails for its own reason.
      *Barcode* has nothing to match against: no import channel carries one (there is no barcode
      column in the export), so in a catalogue built by import the field is empty on both sides.
      *Catalogue number + label* runs straight into findings 3 and 5 — `catno` carries the literal
      sentinel `none` and, in one observed row, a barcode instead of a catalogue number, while
      `label` is a comma-joined list whose members may themselves contain commas. Matching on that
      attaches a user's copy to somebody else's release, which is **worse than a duplicate**: a
      duplicate is fixed by merge, a wrong link is silent and looks correct.
      *Fuzzy artist + title + year + format* has no year to work with in the observed export, and is
      the same failure mode with a similarity score in front of it.
      **No review screen**, and the reason is that with exact rungs only there is nothing to
      adjudicate: a row either carries a resolvable identifier or it does not. A screen would be a
      ten-thousand-row triage bought to confirm decisions that were never ambiguous. The review that
      does exist is the **report afterwards** ([10.2.6](10b-import.md), [10.2.9](10b-import.md)).
      The importer also does **not** run [2.4.9](02-catalogue-model.md)'s advisory duplicate
      detection per row — it would flag most of them by design ([2.1.2](02-catalogue-model.md)) and
      teach the user to ignore it. Duplicates surface in the merge tooling, where somebody is
      actually looking.
- [x] 10.2.5 Vocabulary mapping: Goldmine grading (`Mint (M)`, `Near Mint (NM or M-)`, …) → our condition codes; Discogs folders → our folders/tags ([3.3](03-collection.md)); rating 1–5; Discogs formats → our format structure ([2.4.2](02-catalogue-model.md)).
      **Decision: four table-driven mappings, all one-way, all reporting what they could not map —
      and as of 2026-08-15 all four live in the Discogs converter, not in the importer.**
      **This item is the clearest case for [10.2.1](10b-import.md)'s restructuring**: every mapping
      below is a statement about somebody else's vocabulary. The importer receives values already in
      our own seeded vocabularies ([3.2](03-collection.md), [2.4.2](02-catalogue-model.md)) and does
      no translation at all — which is also what makes our own export re-importable without a mapping
      step that could lose something.
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
      custom-field columns folded into the note. **The converter carries those findings across the
      boundary** ([10.2.1](10b-import.md)) so they land in the same report as the importer's own.
- [x] 10.2.6 The import process as an entity: uploaded file → job → report/preview → apply. Do we keep the source file and the matching log? Can an import be **rolled back as a whole**?
      **Decision:** **upload → job → apply → report. The preview step is cut, the source file is not
      kept, and rollback covers the collection side only.**

      ```
      import(id, user_id, source, filename, file_bytes, sha256, status,
             rows_total, rows_applied, created_at, finished_at, report)
      collection_item.import_id   nullable FK
      wantlist_item.import_id     nullable FK
      ```

      `source` records **which converter ran, or `native` for our own format**
      ([10.2.1](10b-import.md)) — one column, and it is what makes the report legible a year later.
      **No preview/apply split.** It means holding a parsed ten-thousand-row intermediate somewhere
      — a staging table needing its own garbage collection, or the file again — plus a second
      confirmation most people click through without reading. It buys the ability to say no to a
      result that is already fully reversible on the side that matters.
      **The conversion is a stage of the job, not a separate user step.** The user uploads one file
      and reads one report; converting is not something we make them do in a spreadsheet and paste
      back. The purity of the converter is an internal property and must not be paid for at the door.
      **The uploaded file lives in `file_bytes` on the import row and is cleared when the job
      finishes.** In Postgres because Postgres is the only datastore; on the row because the job
      needs it and nothing else does; cleared because it is the user's personal data
      ([10.5.3](10e-legal-sources.md), [13.3](13-legal.md)), they still hold the original, and a
      retained copy is a retention policy we would have to write. `sha256` survives the clearing —
      [10.2.7](10b-import.md) needs it. **This answers
      [10.4.8](10d-model-requirements.md): nothing goes to [8.2](08-media.md)'s storage, and nothing
      outlives the job.** The **converted** bytes are never stored at all — they exist inside the job
      and are re-derived on restart, which is one fewer thing holding personal data.
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
      **Decision (extended 2026-08-15): three defences, because our own format can carry a good key
      and a third party's may not.**
      Note first what makes this hard and is easy to forget: [3.1](03-collection.md) deliberately
      allows several copies of the same release, so "this user already owns this release" is **not**
      a duplicate signal. Row-level idempotency is impossible without a stable per-instance
      identifier.
      1. **Our own format carries `item_id`** — [10.3](10c-export.md) now exports the collection
         item's own identifier ([10.4.6](10d-model-requirements.md)), so re-importing a file we wrote
         **skips items that still exist** and reports the count. **Guarded by provenance, because a
         bare id from a file is not trustworthy:** the skip applies only when the file's
         `exported_by` account matches the importing user. Importing somebody else's export — or your
         own into a fresh account — creates every row, which is correct, and cannot collide with an
         unrelated item that happens to share a number.
      2. **When a converter's source carries an instance id** — `collection_item.source` +
         `source_instance_id`, unique per user. A re-import then **skips** rows it has already
         created and reports the count. Skip rather than update: updating would let a re-upload
         silently revert edits the user made in our UI afterwards, which is a worse failure than a
         no-op and an invisible one.
      3. **When neither exists** — a `sha256` of the uploaded bytes, per user. Uploading a file
         already imported is refused with "you imported this file on <date>; it created N items", a
         link to that import, and an explicit override for the user who means it. It costs one column
         and catches the mistake that actually happens: the double submit and the re-upload after a
         browser back. **This defence is format-independent and always runs**, which is why it is the
         one that never fails.
      **What we do not pretend to support, stated plainly on the upload page:** without a per-item
      identifier there is no incremental re-import. A user who adds twenty records on Discogs and
      re-exports cannot merge that file into what they already have — they import into an empty
      collection, or they roll back ([10.2.6](10b-import.md)) and import afresh. Saying so is better
      than a heuristic that silently duplicates ten thousand rows. **Our own format is exempt from
      that limitation** by defence 1, which is a real argument for round-tripping through it.
      Whether defence 2 ever activates is [10.2.2](10b-import.md)'s verification question; the design
      accommodates both answers, which is why this item can close while that one cannot.
- [x] 10.2.8 Volumes: a 10,000-item collection means a background job with progress (see [11.7](11-stack.md)), a file size limit, timeouts.
      **Decision:** **a background job on [11.7](11-stack.md)'s queue table, 10 MB and 20,000 rows as
      hard limits, a streaming parse, and a progress counter on a page the user reloads themselves.**
      The limits are checked before anything is applied and are refusals, not truncations —
      truncating a user's collection at row 20,000 is the silent data loss the whole section exists
      to avoid. They are generous against [1.4](01-product.md)'s volumes and exist to bound the job,
      not to ration. **They apply to the uploaded file**, whatever format it is in; a converter runs
      inside those bounds and cannot expand a file past them.
      **Streaming parse**, never the whole file in memory: a 10 MB CSV read incrementally, rows
      applied in batches with a commit per batch. Not one transaction for the file — a transaction
      that fails on row 9,999 discards the other 9,998 and holds locks for minutes, and
      [10.2.9](10b-import.md) accepts partial success anyway. **A converter is row-wise for the same
      reason** — it maps a row to a row, so it composes with the streaming parse instead of
      demanding the whole file be materialised first.
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
      completion ([10.2.6](10b-import.md)) — re-runs the converter, which is pure and therefore
      yields the identical rows, skips `rows_applied` rows and continues. That is why the counter is
      persisted per batch rather than held in memory: it is the resume point first and a progress
      display second. **A pure converter is what makes resume trivially correct** — a stateful one
      would have to be resumable itself.
- [x] 10.2.9 Error handling: is partial success acceptable? A "N rows unrecognised" report + exporting the problematic rows back to CSV for manual fixing.
      **Decision:** **yes — partial success is the normal outcome, and the unit of failure is the
      row.** Three bad rows must not cost the other 9,997; a collector's file is thirty years of
      typing and an all-or-nothing importer teaches them to distrust it.
      A row that cannot be parsed, converted or applied is skipped and recorded with its number and
      its raw text. **The failed rows are downloadable as a CSV in the original column layout** — fix
      in a spreadsheet, re-upload just those — which costs nothing because
      [10.3](10c-export.md) already built the CSV writer. That the export machinery serves the
      importer is a small argument for [1.9](01-product.md)'s ordering being right.
      **Whole-file failure only before anything is applied**, and only for: **no recognised format**
      ([10.2.1](10b-import.md)'s detection failing), no recognisable header, over the size or row
      limit, an already-imported hash without the override ([10.2.7](10b-import.md)).
      **One report, whichever side a finding came from** — this is the obligation
      [10.2.1](10b-import.md)'s boundary creates, and honouring it is what keeps the layering
      invisible to the user. The report lists, per category with counts:
      rows applied · rows skipped and why · stubs created · columns ignored entirely
      (marketplace columns per finding 8, [3.2](03-collection.md)'s price) · values folded into the
      note ([3.4](03-collection.md)) · unmapped conditions and format descriptors
      ([10.2.5](10b-import.md)) · ambiguous label/catno fields · **which converter ran**. Every one
      of those categories exists because some invariant says the user must be told rather than
      surprised later.
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
      **A second thing that looks like one after 2026-08-15's restructuring, and is not:** our own
      format being importable does not make it a catalogue channel. Our export holds a user's
      collection and wantlist ([10.3.1](10c-export.md)), never the catalogue, so a hand-crafted file
      is still a **collection** import — bounded by [10.2.8](10b-import.md)'s 20,000 rows and
      [14.5](14-security.md)'s three imports a day, minting ordinary stubs that anyone can merge or a
      moderator can delete. It is the same surface the Discogs path always had, not a new one.
- [x] 10.2.11 Two-way sync with Discogs (our changes → theirs) — do we declare it out of scope right away, or keep the option open?
      **Decision:** **out of scope permanently, and it is not this section's call to make.**
      Writing to Discogs means calling Discogs, which [1.5](01-product.md) forbids for reads already;
      it would additionally need write-scoped credentials to a user's account at another service —
      the single largest thing we could hold and the one with no upside we can name. Nothing is kept
      open, no column is reserved, and the option does not need one: it would be a new service
      talking to a new API, not a change to this schema.
      Import stays what its name says — a one-way door in, with [10.3](10c-export.md) as the door
      out. **A converter does not soften this**: it reads a file the user already has, and it never
      talks to anyone.

## Working notes

**2026-08-15 — why the section was restructured, recorded because the items now read as if it were
always this way.** The original shape made the Discogs CSV the importer's subject, which put an
unverified third-party format ([10.2.2](10b-import.md)) on the critical path of the schema, the job,
the report and the round-trip test. Moving it behind a converter changes the blast radius of every
remaining unknown from *the import feature* to *one pure module*, and it makes the importer testable
with no third-party file at all. **The decisions themselves barely moved** — conditions still parse
the bracketed abbreviation, folders still become tags, matching is still exact-or-mint — they simply
sit on one side of a boundary now.
The trap avoided: **a converter is not "later".** Our own format only exists after someone has used
the site, so the Discogs converter is the MVP's only entry point and ships with it
([15.1](15-roadmap.md) gives it its own slice).

**2026-08-14 — verified against a real Discogs export.** A genuine export was inspected (18 rows,
CD-heavy) and then deleted; only structural facts are recorded here, no collection contents. The file
was an **inventory** (marketplace listings) export, *not* a collection export — so this settles the
shape of Discogs' CSV conventions but leaves the collection-specific columns still unverified.
*(As of 2026-08-15 every finding below is a constraint on the **Discogs converter**, not on the
importer.)*

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
- **10.4.1** — a converter writes an `external_ids` row per stub (`discogs`, `release`, the id), and
  both [10.2.4](10b-import.md)'s rung 2 and [10.3.2](10c-export.md)'s `discogs_release_id` column
  read it. The table is now load-bearing rather than speculative — and it is the **only** place a
  third party's identifier lives, which is what lets the importer stay format-agnostic.

**2026-08-15 — the schema additions this section makes outside its own table.**
`import_id` on `collection_item` and `wantlist_item` (nullable, for rollback), and
`source` + `source_instance_id` on `collection_item` (nullable, for idempotency). Both are additive
and neither is read by anything except the importer, which is the property that lets
[10.2.7](10b-import.md)'s unverified key question stay open without blocking the schema.
**Added by the restructuring:** none. [10.2.7](10b-import.md)'s defence 1 reads `collection_item.id`
and [10.3](10c-export.md)'s new `item_id` column, both of which exist already.

**2026-08-15 — noted and deliberately not decided: progress without a refresh.**
[10.2.8](10b-import.md) leaves the import page static because the no-real-time invariant is
system-wide and a progress bar is a poor reason to reopen it. If it ever proves genuinely bad in
use, the smallest possible change is a `<meta http-equiv="refresh">` on that one page — which holds
no connection and runs no script, and is arguably outside what the invariant refuses. That argument
belongs at [5.3](05-messaging.md)/[9.x](09-nfr.md) where the invariant lives, not here, and it should
be made explicitly rather than smuggled in with an implementation.

# 10.3 Export

**Priority:** P0 — build it into the architecture from the start ([why](10d-model-requirements.md#why-section-10-is-p0))
**Siblings:** [10.1 public API](10a-public-api.md) · [10.2 import](10b-import.md) · [10.4 model requirements](10d-model-requirements.md) · [10.5 legal](10e-legal-sources.md)

- [x] 10.3.1 What we export: collection, wantlist, profile, messages, one's own catalogue edits; separately — a full dump of all the user's data for GDPR/152-FZ ([13.3](13-legal.md)).
      **Decision:** **two exports, one set of serialisers.** The *list export* (collection, wantlist)
      is the collector-facing one [3.9](03-collection.md) asked for; the *account dump* is
      [13.3](13-legal.md)'s, and it is the list export plus more sections rather than a second
      exporter with its own bugs.
      The account dump holds: profile (nickname, email address, bio, the three settings from
      [6.6](06-accounts.md), registration date); the collection with its owner-only fields and tags;
      the wantlist; every conversation the user takes part in, both sides' bodies verbatim
      ([5.5](05-messaging.md)); and the revisions they authored ([4.2](04-editing.md)) as
      entity/id/timestamp. **Not** the revision snapshots themselves — those are catalogue content,
      public by definition, and listing what someone edited and when is the part that is about the
      person. No sessions, no rate-limit counters, no logs ([14.6](14-security.md) governs those).
      **Export is owner-only, and there is no export button on someone else's public collection.**
      The argument [3.9](03-collection.md) makes for export is trust in getting *your own* data
      out. A visitor bulk-copying a public list is a different feature with a licence question
      behind it, and it belongs to [10.3.4](10c-export.md) and [13.1](13-legal.md), not here.
      Additive later at no cost, since the field filter it would need is the one
      [3.7](03-collection.md) already requires.
- [x] 10.3.2 Formats: CSV (do we try to be Discogs-compatible so there is a way "back" too?), JSON (full, ours), XLSX — is it needed. The export format is a public contract; changing it hurts.
      **Decision:** **CSV and JSON. No XLSX, and no claim of Discogs compatibility.**
      **JSON is the lossless one** — our own shape, a top-level `format_version`, everything the
      model holds including tags and owner-only fields. It exists to be re-imported: it is the
      format [11.10](11-stack.md)'s round-trip property runs on, and an export nobody can re-import
      is a promise nobody checks.
      **CSV is the human one** — one flat row per item, for spreadsheets and for feeding another
      service. Lossy by construction (tags collapse into one `;`-joined column, and a tag containing
      `;` cannot survive that — [3.3](03-collection.md) lets users name tags freely).
      **The exporter reports what it flattened**, on the export page and in the file's own header
      comment where the format allows one. This is [10.2](10b-import.md)'s never-discard-silently
      obligation pointed the other way, and it costs one counter.
      **No XLSX**: a zip of XML needing a library, for a file every spreadsheet already opens from
      CSV.
      **Not Discogs-compatible** — decided, not skipped. Matching their importer means tracking an
      undocumented third-party format we cannot test against, and we have not even verified their
      *collection export's* own columns ([10.2.2](10b-import.md)). What we do instead is carry
      `discogs_release_id` as a column whenever [10.4.1](10d-model-requirements.md)'s `external_ids`
      has one. That single field is what actually lets a user rebuild their collection somewhere
      else; the rest of the column names are decoration.
      **The contract, stated so that breaking it is a decision rather than an accident:** appending
      a column is compatible, renaming/removing/reordering is not; JSON changes bump
      `format_version`. And whatever identifier appears in these files is public forever — which is
      what makes [10.4.6](10d-model-requirements.md) a P0 item rather than a taste question.
- [x] 10.3.3 Synchronous (download a file) or job + link/email — depends on the sizes ([11.7](11-stack.md)).
      **Decision:** **synchronous and streamed, always. No job, no generated file, no emailed link.**
      Ten thousand items at a couple of hundred bytes is about 2 MB: one indexed query streamed
      straight into the response body, well under a second on [1.4](01-product.md)'s box. A job would
      buy the queue ([10.4.5](10d-model-requirements.md)), a produced file to store and expire
      ([10.4.8](10d-model-requirements.md)) holding personal data, a download URL with its own
      authorisation story, and a mail template — to avoid a request that is already fast.
      **The asymmetry is worth naming: import needs the queue, export does not.**
      [10.4.5](10d-model-requirements.md) is justified by [10.2.8](10b-import.md) alone, so if
      import ever moved out of scope the queue would go with it. [11.7](11-stack.md) listed "export
      generation" among the queue's expected consumers; that expectation is **not exercised**, and
      the item stands unchanged — the queue is mandatory for import and email regardless.
      The account dump takes the same path — it is the same order of magnitude, and a dump is
      requested once.
      Export is a heavier-than-average query behind a login, so it takes an ordinary per-user rate
      limit ([14.5](14-security.md)). That is the whole of its protection.
- [x] 10.3.4 Exporting the whole catalogue (our own "dump", like MusicBrainz/Discogs) — is that planned? Directly tied to the data licence ([13.1](13-legal.md)).
      **Decision:** **not in the MVP — deferred and blocked on [13.1](13-legal.md)**, not refused on
      merit.
      Three reasons, in order of weight. (1) Publishing a dump *is* the licensing act: it commits
      [13.1](13-legal.md) before [13.1](13-legal.md) has been answered, and the terms are much harder
      to tighten afterwards than to loosen. (2) The EU database-rights note in
      [NOTES.md](NOTES.md) bites here and nowhere else — one user's export coming *in* is not a
      substantial part of anyone's database, but an accreted catalogue republished as a dump is
      exactly the redistribution the risk was never accepted for; ingest was.
      (3) It needs the schedule-and-storage machinery [10.3.3](10c-export.md) just declined, for
      zero consumers at ~12,000 releases ([1.4](01-product.md)).
      When it is reopened it is JSON Lines per entity from the same serialisers plus a cron entry —
      a licence decision with a small job attached, not an engineering project. Nothing needs to be
      built now to keep it possible.
- [x] 10.3.5 Export privacy: what is excluded from a public collection export (prices, notes, purchase places), what is available only to the owner.
      **Decision:** **the only export is the owner's, so the filter is trivial: everything the owner
      can see, they get** — note, `purchased_on` and `purchased_at` included, and price too if
      [3.2](03-collection.md) ever reopens. There is no reduced public variant to specify because
      [10.3.1](10c-export.md) gives no one else an export.
      Two rules do bite, and both concern *other people's* data inside the account dump: a
      conversation partner appears as a nickname and as message bodies the requester already has in
      their mailbox, and **never as an email address**; a deleted account appears as its tombstone
      ([6.5](06-accounts.md)), not as the nickname it used to hold.
      One shape rule, stated because it looks like a contradiction: the export **denormalises
      catalogue fields into the row** — artist, title, label, catalogue number, format, country,
      year — although [3.1](03-collection.md) forbids storing any of them on the item. A file has no
      foreign keys and is read once, detached from us; a row has both. That is the entire difference,
      and it is why denormalising is right in one place and a synchronisation bug in the other.

## Working notes

**2026-08-15 — the column sets, as decided.** Recorded here rather than in the items because they
are the thing an implementation will actually read, and because
[10.3.2](10c-export.md)'s contract rule applies to them from the day the first file leaves the box.

Collection CSV:

```
release_id, discogs_release_id, artist, title, label, catno, format, country, released,
media_condition, sleeve_condition, rating, purchased_on, purchased_at, note, tags, added_at
```

Wantlist CSV:

```
release_id, discogs_release_id, artist, title, label, catno, format, country, released,
priority, note, added_at
```

`media_condition` and `sleeve_condition` carry our own codes (`NM`, `VG+`, …), not Goldmine prose —
[10.2.5](10b-import.md) parses the abbreviation on the way in, and writing the prose back out would
be pretending to a compatibility [10.3.2](10c-export.md) declined. `tags` is `;`-joined. Empty is
empty: no `n/a`, no sentinel, because [10.2](10b-import.md) records what a sentinel like Discogs'
literal `none` costs the reader.

**2026-08-15 — what "lossless" is claimed for, precisely.** JSON export → import reproduces the
collection, the wantlist and the tags exactly, and that is the property
[11.10](11-stack.md) tests. CSV makes no such claim: tags containing `;` and any future multi-valued
field flatten. Both facts belong on the export page in one sentence each, since a user choosing a
format is choosing which of these they get.

**2026-08-15 — deliberately not built, so that a later reader does not mistake it for an oversight.**
No scheduled or repeat export, no "email me my collection monthly", no per-list format preference,
no partial export by filter or tag. Each is a setting or a job for a page that is one click and one
second; [6.6](06-accounts.md) already closed the settings question against exactly this kind of
addition.

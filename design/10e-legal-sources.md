# 10.5 Legal aspects and the ToS of external sources

**Priority:** P1, but check before implementing
**Siblings:** [10.1 public API](10a-public-api.md) · [10.2 import](10b-import.md) · [10.3 export](10c-export.md) · [10.4 model requirements](10d-model-requirements.md)

**What this section is not free to decide.** [1.5](01-product.md) predicted this section would
"shrink to almost nothing" and it has: our server never calls an external music database, downloads
a dump or scrapes, so most of what the items ask about — rate limits, User-Agent requirements, bulk
mirroring terms — describes activities we do not perform. What survives is narrow and worth getting
right: a user uploading their own file, and images.

- [x] 10.5.1 Discogs API: the current terms (mandatory User-Agent, limits on the order of tens of requests per minute, restrictions on bulk mirroring) — re-read them before committing to live sync.
      **Decision: moot as written, because there is no live sync to commit to — and the question that
      replaces it has a clear enough answer to act on.**
      **The API terms bind API callers and we are not one.** [1.5](01-product.md) bars the server
      from calling an external music database at all; [10.2.1](10b-import.md) and
      [10.2.11](10b-import.md) closed live sync and write-back permanently. A User-Agent
      requirement, a per-minute limit and a bulk-mirroring restriction are conditions on a thing we
      never do, and re-reading them would be diligence about someone else's product.
      **What actually remains, per [NOTES.md](NOTES.md): may a user do what they like with their own
      export?** Checked 2026-08-15, as far as is possible without a lawyer:
      - The export is an **official, supported Discogs feature** — *Export My Collection* →
        *Request Data Export* — documented in their own help centre. It is offered to the user as
        their data.
      - The licensing language in their ToS runs the other way from what this item feared: it is the
        licence a **user grants Discogs** over their contributions, plus Discogs' onward licensing
        through the Data Dump and the API. We found nothing purporting to restrict what a user does
        with a copy of their own collection.
      - Their ToS page refuses automated fetching (HTTP 403), which is consistent with a service that
        does not want to be crawled — and we do not crawl it.
      **The framing that decides it, and it is more useful than the reading: their ToS is not our
      contract.** We are not party to it; it binds the user. Our exposure is at most inducing someone
      to breach their own agreement, which is thin where the export is a feature that service built
      for exactly this. And for an EU user, GDPR data portability is a **right to their own data**
      that no terms of service can remove — a firmer footing than any clause we could read.
      **The residual risk is not here and must not be confused with it.** The live one is
      [NOTES.md](NOTES.md)'s EU database-rights note — *we* accreting many users' exports into one
      public catalogue — accepted at portfolio scale by [1.5](01-product.md) and reaching its limit
      at [10.3.4](10c-export.md)'s dump. One user's upload is safe on every reading; republishing the
      accretion is the act that would want an actual lawyer.
      **Not lawyer-reviewed**, and it stays on [NOTES.md](NOTES.md)'s verification list as a ten-minute
      read before the ToS is published — alongside [13.5](13-legal.md), which is in the same
      position.
- [x] 10.5.2 Monthly Discogs dumps vs the API: which suits us by volume and what is permitted (the data itself is CC0, but the terms of access to the service are a separate story).
      **Decision: neither, permanently. This is [1.5](01-product.md)'s question and it was already
      answered.**
      The item asks us to choose between two ingestion channels, and [1.5](01-product.md) removed the
      category: data enters only as a file a user uploads or a person typing into our UI.
      [10.2.10](10b-import.md) closed the loophole the dump would most naturally use — **an
      administrator loading it through `psql` is exactly what [1.5](01-product.md) forbids**, and the
      shell it happens in does not change the act.
      **That the data is CC0 does not make it permitted here, and the distinction is the item's own.**
      The licence answers *may we redistribute these facts*; [1.5](01-product.md) answers *may this
      system acquire them that way*, and it says no — for reasons that are ours rather than legal:
      it would make [section 4](04-editing.md), the interesting half of a portfolio piece
      ([1.1](01-product.md)), largely redundant, and it buys a sync problem.
      **The one thing worth recording for a reopening**, since NOTES keeps rejected approaches so we
      do not re-litigate them: **if [1.5](01-product.md) is ever reversed, the dump is the better of
      the two options, not the API** — no rate limit, no availability dependency on someone's
      service, an explicit licence on the landing page, and a volume ([1.4](01-product.md)'s ~12,000
      releases) that a monthly file overwhelms rather than strains. Reversing [1.5](01-product.md)
      also means [13.1](13-legal.md)'s CC0 dedication must be re-examined, since we would then be
      redistributing material we did not receive under our own ToS grant.
- [x] 10.5.3 A user importing their own collection is their own data — legally the simplest case; but the source file contains personal data (see [10.4.8](10d-model-requirements.md), [13.3](13-legal.md)).
      **Decision: confirmed as the simplest case, with three points fixed — the lawful basis, the
      warranty in the ToS, and the line between the file and what the import extracts from it.**
      **Basis: performance of the contract.** The user asked us to import their file; we process it
      to do the thing they requested. No consent banner, nothing to withdraw beyond deleting the
      import. Retention is [10.4.8](10d-model-requirements.md)'s — the file lives in
      `import.file_bytes` for the duration of the job, at most seven days, and only its `sha256`
      survives ([10.2.7](10b-import.md)).
      **The file contains the uploader's data and nobody else's**, which is a finding rather than an
      assumption and it simplifies [13.3](13-legal.md) considerably. The observed columns
      ([10.2](10b-import.md)'s working notes) are catalogue facts about objects — artist, title,
      label, catno, format, `release_id` — plus facts about this user's own copy: folder, rating,
      conditions, notes, date added. **No third party's personal data enters the system through
      import**, so there is no case where we hold data about someone who never dealt with us.
      **The ToS carries a warranty**, one sentence, alongside the CC0 grant
      [13.1](13-legal.md) already requires: the uploader confirms the file is theirs to upload. It
      is what we can honestly ask for and it puts the question where the facts are.
      **The line that matters for erasure, because it looks like a contradiction and is not.**
      Deleting the file deletes the *file*; it does not retract the catalogue rows the import minted.
      Those are facts about physical objects — [13.1](13-legal.md) dedicates them to the public
      domain and the ToS grant makes each contribution match — and they are not personal data about
      the uploader. So an account deletion ([6.5](06-accounts.md)) removes the collection, the
      wantlist and the tags, and leaves the releases standing, attributed to a tombstone. That is
      the same rule [4.2](04-editing.md)'s revisions already follow, and [13.3](13-legal.md) must
      state it plainly rather than implying that deletion unwinds an import.
- [x] 10.5.4 Cover art and images from external sources remain a separate question ([2.5.3](02-catalogue-model.md)) — importing a collection does not legalise them.
      **Decision: agreed, and the design already forecloses it in two independent ways — recorded
      here as one explicit importer rule.**
      **Images enter only as uploads.** [8.1](08-media.md) fixed a single path: a person chooses a
      file, it becomes two WebP derivatives, the original is discarded. There is no other way for a
      byte of image data to reach the bucket.
      **The importer ignores any image column or URL, and this is the rule worth writing down**, even
      though the observed Discogs export carries neither. It is doubly barred: fetching a URL from a
      user-supplied file is what [1.5](01-product.md) forbids and what [14.3](14-security.md) calls
      out by name as the SSRF surface this system does not have — and *if* it were fetched, the
      rights position would be exactly the one this item warns about. A future export format that
      grows an image column changes nothing; the parser skips it and
      [10.2](10b-import.md)'s report says it did, per the never-discard-silently rule.
      **Rights for what users do upload** are handled where they belong: the ToS warranty
      ([13.1](13-legal.md)) that the uploader may share it, [13.1](13-legal.md)'s exclusion of
      artwork from the CC0 dedication — we never held rights in it, which is precisely why
      [8.4](08-media.md) *removes* rather than argues — and [8.4](08-media.md)'s takedown, the one
      hard delete in the system, with a `blocked_at` on the `sha256` that makes re-upload refusal
      free.
      **What we do not do:** screen uploads automatically ([8.4](08-media.md) declined that — every
      option is an external API or a model on a 2 GB box, and sleeve art is where a classifier is
      worst), or claim any licence in an uploaded image beyond displaying it.

## Working notes

- **2026-08-15 — Section closed. All 4 items decided, and [1.5](01-product.md)'s prediction held:
  three of them dissolve rather than resolve.** [10.5.1](10e-legal-sources.md) and
  [10.5.2](10e-legal-sources.md) ask about terms governing activities we do not perform, and
  [10.5.4](10e-legal-sources.md) asks about a channel that does not exist. The only item with real
  content is [10.5.3](10e-legal-sources.md), and it is the easy one.
- **2026-08-15 — What was checked against reality and what was not.** Checked: that the collection
  export is an official Discogs feature and that their ToS's licensing clauses run user→Discogs
  rather than restricting a user's own copy ([10.5.1](10e-legal-sources.md)); that the observed
  export columns carry no third party's personal data ([10.5.3](10e-legal-sources.md)). Not checked
  by a lawyer, and both still on [NOTES.md](NOTES.md)'s verification list: the ToS reading itself,
  and [13.1](13-legal.md)'s assumption that a ToS clause waives the EU sui generis database right in
  a contribution.
- **2026-08-15 — Two obligations this section hands on**, both one sentence each rather than
  features. [13.1](13-legal.md)'s ToS gains an upload warranty — *this file is yours to upload*
  ([10.5.3](10e-legal-sources.md)) — beside the CC0 grant it already carries.
  [13.3](13-legal.md)'s privacy page gains the erasure line: deleting an account removes the
  collection, wantlist and tags and does **not** retract the catalogue rows an import minted, because
  those are facts about objects rather than about the person.
- **2026-08-15 — Where the real legal risk sits, so it is not looked for here.** It is not in
  anyone's terms of service. It is [NOTES.md](NOTES.md)'s EU database-rights note — repeated
  extraction of insubstantial parts, accepted for ingest at portfolio scale by
  [1.5](01-product.md) and never for redistribution — and it becomes live only at
  [10.3.4](10c-export.md)'s public dump, which [13.1](13-legal.md) has now unblocked on licence
  grounds while leaving that risk exactly where it was.

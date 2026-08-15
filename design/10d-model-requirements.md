# 10.4 What this already demands from the data model (P0 checklist)

**Priority:** P0 — build it into the architecture from the start ([why](#why-section-10-is-p0))
**Siblings:** [10.1 public API](10a-public-api.md) · [10.2 import](10b-import.md) · [10.3 export](10c-export.md) · [10.5 legal](10e-legal-sources.md)

## Why section 10 is P0

Even if we write the API and the importers much later, three things have to be settled **before**
designing the schema: external identifiers and data provenance (10.4.1–10.4.2), whether
"unresolved" collection items are allowed (10.4.3), and the rule that all mutations go through a
shared service layer — UI, API and importer are three callers of the same function (10.4.4).
Bolting this onto a finished database is far more expensive than designing it in now. That is why
10.1–10.4 are P0 despite being implemented late.

## Checklist

- [x] 10.4.1 External IDs ([2.5.4](02-catalogue-model.md)): a separate table `external_ids (entity_type, entity_id, source, external_id, url, synced_at)` instead of `discogs_id`/`musicbrainz_id` columns on every entity — agreed? (more flexible, supports several sources and sync history right away)
      **Decision: agreed on the table, with two of the proposed columns deleted and a unique index
      added in their place.**

      ```
      external_id
        id           bigint identity
        entity_type  enum (release, master, artist, label)
        entity_id    bigint
        source       enum (discogs, musicbrainz)   -- seeded, closed, per the curated-set invariant
        external_id  text
        added_by     bigint → user
        added_at     timestamptz
        unique (entity_type, source, external_id)
        index  (entity_type, entity_id)
      ```

      **`synced_at` is deleted rather than renamed.** There is no sync and there never will be —
      [1.5](01-product.md) bars the server from calling an external database at all, and
      [10.2.1](10b-import.md)/[10.2.11](10b-import.md) closed the live-sync and write-back options
      permanently. A column whose only possible value is the insert timestamp is `added_at` wearing a
      misleading name, and a column called `synced_at` will eventually get a sync to justify it.
      **`url` is deleted too**, for a reason worth keeping: it is a pure function of
      `(source, entity_type, external_id)`, so storing it means keeping ~12,000 copies of a third
      party's URL scheme and migrating them when that scheme changes. The function lives in
      `musilka-domain`. The second reason is sharper — **a stored URL is an invitation for some later
      code path to fetch it**, and [14.3](14-security.md) forbids exactly that by name. Nothing in
      this system resolves a URL it did not write.
      **The unique index is the point of the table.** [10.2.4](10b-import.md) matches on an exact
      external id and on nothing else, so import is one indexed lookup per row rather than a
      similarity search. Hand entry that collides shows the user the entity that already holds that
      id instead of erroring — the same "here is your duplicate, merge it" shape as everywhere else.
      **No foreign key, and the cost is named.** `(entity_type, entity_id)` is polymorphic, so
      referential integrity is the service layer's job ([10.4.4](10d-model-requirements.md)) rather
      than the database's. The concrete obligation is on merge ([4.4](04-editing.md)), which must
      repoint these rows like every other reference — **including the one collision case**: when
      winner and loser both carry the same `(source, external_id)`, which is frequently *why* they
      were duplicates, the unique index makes repointing impossible and merge deletes the loser's
      row. That is the only row merge ever deletes, and it is safe because the fact survives intact
      on the winner and an `external_id` row is not user content.
      **Who may write one:** any verified user editing the entity ([6.1](06-accounts.md)), under
      [14.5](14-security.md)'s catalogue-edit limit, and the importer acting as the uploading user.
      **The MVP has exactly one source**, `discogs`, from the CSV's `release_id`
      ([10.2.4](10b-import.md)); a second source is a seed row, not a migration.
- [x] 10.4.2 Data provenance: do we mark where an entity/field came from (`user`, `import:discogs`, `import:musicbrainz`), and may that value be overwritten on a repeated sync?
      **Decision: no field-level provenance anywhere. Provenance is [4.2](04-editing.md)'s revision
      plus two columns on it, and the second half of the question is moot.**
      **The second half first, because it dissolves the item.** "Overwritten on a repeated sync"
      presupposes a sync; [1.5](01-product.md) forbids one and [10.2.11](10b-import.md) closed the
      write-back direction as well. Nothing in this system ever refetches a value, so no value is
      ever at risk of being replaced by a machine.
      **What we store instead**, all of it already required for other reasons:
      1. **The revision** ([4.2](04-editing.md)) — who changed the aggregate, when, and what it
         looked like afterwards. Two columns are added here: `revision.source`
         (`ui` | `import` | `merge` | `moderation`) and a nullable `revision.import_id`. Together
         they answer "where did this release come from" with *this release was minted by Dmitry's
         import #7 on 3 March and has been edited twice by hand since*.
      2. **`external_id.added_by` / `added_at`** ([10.4.1](10d-model-requirements.md)) — provenance
         of the identifier itself, which is the only field that genuinely came from somewhere else.
      3. Nothing else.
      **Why not a provenance column beside every field.** It is a parallel schema that must be
      written on every edit and is read to answer one question — *is this value trustworthy?* —
      which [2.4.7](02-catalogue-model.md) already refused to answer with a stored status. The
      quality signal in this design is the edit history, which is data, not an assertion parked next
      to the data. Field-level provenance is that assertion with a per-column tax attached.
      **The question it is actually asked to answer is derivable and cheap:** "is this an untouched
      import stub?" is *exactly one revision, `source = import`* — one query, no column, and it
      stays correct without anyone maintaining it. That predicate is what the merge tooling
      ([4.4](04-editing.md)) and any "needs work" listing should use.
      **The overwrite rule, stated from this side because it is a model rule rather than an importer
      detail: nothing in the system ever replaces a non-empty field automatically.** Whether the
      importer *fills a blank* on a release it matched by external id is [10.2](10b-import.md)'s to
      say; what is fixed here is that it may never overwrite a value a human typed, because
      [1.5](01-product.md) makes a human the only possible author of one.
- [x] 10.4.3 Do we allow an "unresolved" collection item — with no `release_id`, holding raw text from the import? (see [10.2.3](10b-import.md); this is directly a schema question)
      **Decision: no, and this was answered by [10.2.3](10b-import.md) — recorded here because the
      item that asked it is the schema one.**
      `collection_item.release_id` is `NOT NULL`, and so is `wantlist_item.release_id` and
      `release.master_id` ([2.1.2](02-catalogue-model.md)). An imported row that matches nothing
      mints a stub release and its master rather than parking as raw text.
      The reasoning is [10.2.3](10b-import.md)'s and is not restated; what belongs here is the
      **general rule it belongs to**, now an invariant in [NOTES.md](NOTES.md): this design never
      introduces a half-linked row awaiting human triage. The queue would not be drained
      ([4.3](04-editing.md)), every query in [section 3](03-collection.md) would carry an "and the
      resolved ones" clause forever, and every listing would need a second rendering for an item with
      nothing behind it. Corrections happen through merge ([4.4](04-editing.md)), which exists.
      **The one nullable link in the neighbourhood, so it is not mistaken for an exception:**
      `revision.import_id` ([10.4.2](10d-model-requirements.md)) is null for hand edits. That is an
      optional *annotation*, not an unresolved *reference* — nothing renders differently and no query
      filters on it.
- [x] 10.4.4 Service layer: every mutation (create a release, add to collection, change condition) is a single function called by the UI, the API and the importer alike. No domain logic in controllers or templates. Do we fix this as an architectural rule?
      **Decision: yes, and [11.11](11-stack.md) makes the compiler enforce it rather than the
      reviewer.**
      **The rule.** Every mutation is one function in `musilka-app` taking an explicit `Actor` — a
      user id and role ([4.7](04-editing.md)), or `System` for a timer job. Handlers decode
      ([11.9](11-stack.md)), call exactly one service function, and render. Templates receive domain
      values and query nothing.
      **SQL lives only in `musilka-app`, and this is a refinement of [11.11](11-stack.md)'s table
      rather than a restatement of it:** `musilka-web` does not depend on `hasql` in its
      `build-depends` at all, so a query written in a handler is a build error rather than a review
      comment. [11.11](11-stack.md) lists "sessions" under `musilka-web`; that means the cookie, the
      middleware and the CSRF token ([14.3](14-security.md)) — the session *store* is a service like
      any other.
      **Four rules live inside these functions and therefore cannot be skipped by a caller**, which
      is the entire argument for the layer:
      - authorisation as a clause in the query, never a check after the load ([14.3](14-security.md));
      - the rate-limit check ([14.5](14-security.md)), which is why [9.4](09-nfr.md) could promise
        that a future API inherits the numbers instead of reinventing them;
      - the revision write ([4.2](04-editing.md)) — a mutation that skipped the layer would be a
        mutation with no history;
      - the verified-email barrier ([6.1](06-accounts.md)).
      **There are three callers today and the third is the one that keeps the layer honest:** the web
      handler, the job runner (the importer, [10.2](10b-import.md), and outbound mail), and
      **the test suite** — [11.10](11-stack.md)'s level 2 tests service operations against a real
      Postgres, which is only a meaningful test of the rights model if the rights model is in the
      service. A fourth caller, the API ([10.1](10a-public-api.md)), inherits all four rules for
      free; that is [11.5](11-stack.md)'s reuse boundary and the reason it is the service layer
      rather than HTTP.
- [x] 10.4.5 A background job queue is needed as early as import/export — do we build it into E1, even if we start with a primitive one (see [11.7](11-stack.md), [12.x](12-infrastructure.md))?
      **Decision: yes — in E0 rather than E1, and the primitive version is the final version.**
      [11.7](11-stack.md) fixed the mechanism (one Postgres table, `SELECT … FOR UPDATE SKIP
      LOCKED`, an attempt count and a last error, an in-process worker, every job restartable). What
      this item adds is **the stage**, and it is one earlier than the question assumes: the table and
      the runner land in E0 beside the migration runner, because the *first* consumer is
      [6.1](06-accounts.md)'s verification email in E1's first slice — the barrier every anti-abuse
      decision in the design leans on. A queue that arrives with the importer would arrive after the
      thing that needs it.
      **"Even if we start with a primitive one" is declined as a framing.** One table, a claim query
      and a retry column is not a prototype of a queue to be replaced later; at [1.4](01-product.md)'s
      volume it is the queue, and [11.7](11-stack.md) already refused the broker that would be the
      upgrade path.
      **The consumer list is final and short: the importer ([10.2.8](10b-import.md)) and outbound
      mail (four templates, [section 6](06-accounts.md)).** Export is not a consumer
      ([10.3.3](10c-export.md) is synchronous) and neither are image derivatives
      ([8.3](08-media.md) is a subprocess in the request), which between them retired both of
      [11.7](11-stack.md)'s other expected users.
      **Recurring work is not queue work, and conflating the two is how a queue grows a scheduler.**
      The nightly backup ([9.3](09-nfr.md)), the error digest ([9.5](09-nfr.md)),
      [14.5](14-security.md)'s rate-limit sweep and [10.4.8](10d-model-requirements.md)'s file
      sweeper are systemd timers running a command ([12.2](12-infrastructure.md)) — no row, no
      claim, no retry. **The queue is for work a user action started**, and both its consumers are
      that.
- [x] 10.4.6 Public entity IDs (numeric autoincrement / UUID / slug): once they land in someone's export or in an API client, they are forever. What do we expose outwards?
      **Decision: `bigint`, generated by the database, exposed exactly as it is — one sequence per
      entity type, no UUID, no slug in the identifier position, no opaque code.**
      This was the oldest unblocked P0 item in the design and it has two callers waiting:
      [10.3.2](10c-export.md), where the first exported file makes the choice permanent, and
      [7.5](07-search-ux.md), which fixed every URL as `/<entity>/<id>/<slug>` and left `<id>` as its
      one hole.
      **Unguessability is not a requirement here, and that is a decision already taken rather than an
      assumption.** [14.3](14-security.md) puts authorisation in the `WHERE` clause of every
      user-scoped query and states in terms that sequential integer ids are therefore not a
      vulnerability. The one thing in the system that must be unguessable is
      [3.7](03-collection.md)'s share token, and it is a capability rather than an identifier
      precisely so that this question stays separate.
      **What sequential ids leak, stated so nobody has to wonder:** how many releases exist — which
      the catalogue's own pages publish — and how fast the collection table grows. Neither is a
      secret at [1.4](01-product.md)'s scale, and neither becomes one later.
      **UUIDv7 was the real alternative and it loses on price.** It costs 36 characters in every URL,
      every export cell and every log line, a wider index on every foreign key, and a value no human
      can read back, remember or type — against hiding a row count the design publishes. Nothing in
      [1.4](01-product.md)'s one-process, one-writer shape needs client-side or distributed id
      generation, which is UUIDv7's actual reason to exist.
      **The slug is never the identifier.** [7.5](07-search-ux.md) made it cosmetic and free to
      change; promoting it would make a retitle break every link and destabilise an export column
      that [10.3.2](10c-export.md) has just declared a contract. `/release/42/kino-gruppa-krovi`
      and `/release/42/blood-type` are the same page.
      **Per entity type, and never a bare id.** `/release/42` and `/master/42` are different objects;
      there is no global id space and no attempt at one. Every export column is named for its type
      (`release_id`, [10.3](10c-export.md)'s column sets), every URL carries the type before the
      number, and any future API path does too. An id travelling without its type is a bug.
      **What becomes permanent the moment the first file leaves the box:** that the identifier is an
      integer, that it is *this* integer, and that `release_id` in a file means our release. That is
      [10.3.2](10c-export.md)'s contract and it now has a value in it.
      **Old identifiers keep resolving, which is what makes an old file still work.** A merged entity
      keeps its id and follows `merged_into` ([4.4](04-editing.md), [7.8](07-search-ux.md)'s
      redirect); a deleted one resolves to its tombstone ([4.5](04-editing.md)). So a two-year-old
      export still lands somewhere, and **JSON re-import resolves in this order**: our `release_id`
      (following the merge pointer) → `external_id` ([10.4.1](10d-model-requirements.md)) → mint a
      stub. Exact or nothing at every rung, per [10.2.4](10b-import.md).
      **Mechanics:** `bigint GENERATED ALWAYS AS IDENTITY` on every entity table, no `serial`, no
      shared sequence, `bigint` even where `int` would obviously do — the four bytes are cheaper than
      the migration.
      **Not identifiers, and never exposed as such:** session tokens, CSRF tokens, verification and
      reset tokens, and [3.7](03-collection.md)'s share token. Those are random capabilities
      ([14.2](14-security.md) stores them hashed), and no page ever shows a user a token where an id
      belongs.
- [x] 10.4.7 The change log ([4.2](04-editing.md)) — do we expect to reuse it for incremental "what changed since X" delivery (API/webhooks/dumps)?
      **Decision: we design nothing for it and get it anyway, because two properties we want for
      other reasons are the whole requirement.**
      **The two properties, both cheap and both worth stating so they are not broken by accident:**
      1. `revision.id` is a `bigint` identity, monotonic and never reused, so "everything since
         revision N" is one indexed query.
      2. **Every mutation writes a revision — including merge and delete**, the two that look like
         exceptions. [4.2](04-editing.md) already says every mutation; naming these two matters
         because a consumer replaying the log would otherwise never learn that an entity had
         disappeared or been absorbed, which is the failure that makes a change feed untrustworthy.
      **Nothing else is built and nothing is reserved:** no outbox table, no `changed_fields`
      column, no per-entity feed, and no webhooks — [10.1.10](10a-public-api.md) refuses those
      outright. This is NOTES' rule that we **store what cannot be reconstructed and defer what can
      be built later from it**, applied to the case it was written for.
      **One consequence worth banking:** if [10.1.8](10a-public-api.md) ever wants an `ETag` for a
      catalogue resource, the newest revision id for that entity is one, with no new state to
      maintain.
      **The shape a consumer would get is a snapshot, not a diff** — [4.2](04-editing.md) stores the
      aggregate as it stood. That is the right shape for delivery anyway (a consumer wants the
      current truth, not a patch to apply), and computing a diff from two snapshots is available to
      whoever needs one.
      **One honest limit, recorded so it is not discovered as a bug:** ids come from a sequence, so a
      transaction can commit revision 100 after a reader has already seen 101, and a naive "since N"
      poller could skip it. At one process and a handful of edits a day this is theoretical; the fix,
      if it ever matters, is to page by `created_at` with an overlap window, not to add a table.
- [x] 10.4.8 Storing uploaded import files: where ([8.2](08-media.md)), for how long, do we delete them after a successful apply — they contain the user's personal data.
      **Decision: [10.2.6](10b-import.md) and [14.4](14-security.md) answered where and how long;
      this item adds the sweeper that makes "how long" true, and the sentence
      [13.3](13-legal.md) needs.**
      **Where: `import.file_bytes` (`bytea`) on the import row — not object storage.**
      [8.2](08-media.md)'s bucket is a public prefix with content-addressed, immutable keys, built
      for objects that are served; a private prefix for one temporary blob would be a second
      access-control story, a second lifecycle and a second thing to back up, for a file that lives
      for minutes. In the database it is inside the same transaction as the job that reads it and
      inside [9.3](09-nfr.md)'s existing backup.
      **How long: until the job reaches a terminal state**, then set to `NULL` in the same
      transaction that writes the final status. Only `file_sha256` survives, for
      [10.2.7](10b-import.md)'s re-import defence. [10.2.6](10b-import.md) already refused to keep
      the file: it is personal data, the user still has the original, and keeping it means writing a
      retention policy for something nobody reads.
      **The sweeper, which is this item's only new content.** A deploy kills a running job
      ([11.7](11-stack.md)), so an import can sit in a non-terminal state indefinitely if it is never
      resumed — and for exactly those rows, "deleted when the job ends" silently means "kept
      forever". A daily timer ([10.4.5](10d-model-requirements.md)) nulls `file_bytes` on any import
      row untouched for **7 days** and marks it failed with that reason. The retention claim is only
      as good as the case that goes wrong.
      **The sentence [13.3](13-legal.md) can state:** *an uploaded collection file is kept for the
      duration of the import and deleted when it finishes — at most seven days.*
      **Two things that follow.** [9.3](09-nfr.md)'s observation stands — a dump taken mid-import
      carries those megabytes and the next one does not — and the sweeper is what bounds it.
      And **there is no other retained upload anywhere in the system**: images are re-encoded into
      derivatives and the original is discarded ([8.1](08-media.md)), and [5.4](05-messaging.md)
      refused message attachments, so these two paths are the complete set.

## Working notes

- **2026-08-15 — Section closed. All 8 items decided.** Five were consequences of decisions taken
  elsewhere and are recorded here because this is the file that asked the schema question:
  [10.4.3](10d-model-requirements.md) belongs to [10.2.3](10b-import.md),
  [10.4.4](10d-model-requirements.md) to [11.5](11-stack.md)/[11.11](11-stack.md),
  [10.4.5](10d-model-requirements.md) to [11.7](11-stack.md), and
  [10.4.8](10d-model-requirements.md) to [10.2.6](10b-import.md)/[14.4](14-security.md). The
  genuinely new material is **[10.4.6](10d-model-requirements.md)'s identifier**, the two columns
  removed from [10.4.1](10d-model-requirements.md)'s table, and
  [10.4.2](10d-model-requirements.md)'s refusal of field-level provenance.
- **2026-08-15 — The schema deltas this section creates**, gathered because they are what an
  implementation reads:
  - `external_id` as specified in [10.4.1](10d-model-requirements.md) — unique on
    `(entity_type, source, external_id)`, no `url`, no `synced_at`.
  - `revision.source` and `revision.import_id` ([10.4.2](10d-model-requirements.md)).
  - `bigint GENERATED ALWAYS AS IDENTITY` on every entity table
    ([10.4.6](10d-model-requirements.md)).
  - `import.file_bytes` nulled at terminal state, plus a 7-day sweeper
    ([10.4.8](10d-model-requirements.md)).
  - `NOT NULL` on `collection_item.release_id`, `wantlist_item.release_id`, `release.master_id`
    ([10.4.3](10d-model-requirements.md)).
- **2026-08-15 — Three obligations landing on other people's closed sections.** None reopens a
  decision; each changes a column list or names a rule that was implied.
  [4.2](04-editing.md)'s revision gains two columns and one restated guarantee — merge and delete
  write revisions too ([10.4.7](10d-model-requirements.md)).
  [4.4](04-editing.md)'s merge gains the `external_id` collision case: delete the loser's row rather
  than repoint it ([10.4.1](10d-model-requirements.md)).
  [11.11](11-stack.md)'s library split gains a build-level rule: `musilka-web` does not depend on
  `hasql` ([10.4.4](10d-model-requirements.md)).
- **2026-08-15 — What [10.4.6](10d-model-requirements.md) unblocks.** [10.3](10c-export.md)'s
  column sets now have a defined type in their `release_id` position and can ship;
  [7.5](07-search-ux.md)'s URL shape is complete; and NOTES' oldest open question is closed. The
  decision is cheap to *keep* and expensive to reverse — from the first exported file onward,
  changing it means a migration plus every file already in a user's hands being wrong.

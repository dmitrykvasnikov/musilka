# 4. User editing of the catalogue

**Priority:** P0 — the most underestimated part

- [x] 4.1 Permission model: any registered user edits immediately (wiki style) / pre-moderation / reputation threshold / trusted editors only?
      **Decision:** **Wiki style — any user with a verified email edits immediately.** No
      pre-moderation, no reputation threshold, no trusted-editor tier.
      The one split: **ordinary field edits are open to everyone, merge and soft delete are
      moderator-only** ([4.4](04-editing.md), [4.5](04-editing.md), [4.7](04-editing.md)). Those two
      are the operations that move other people's collection items around, and with no revert UI in
      the MVP ([4.2](04-editing.md)) undoing a bad one is hand work.
      Why not pre-moderation: it needs someone draining a queue, and [1.9](01-product.md) puts any
      moderation queue outside the MVP. At our size the queue's only drainer is also its only filler.
      Why not a reputation threshold: [4.8](04-editing.md) declines to compute reputation, so there
      would be nothing to threshold on.
      Verified email is the only barrier to editing, and it is deliberate — see
      [4.9](04-editing.md).
- [x] 4.2 Edit versioning: do we keep the full change history of entities? Do we need diff viewing and rollback? (the change log is also a candidate feed for incremental delivery — see [10.4.7](10d-model-requirements.md), [10.1.10](10a-public-api.md))
      **Decision:** **Yes, from day one — but as data only. No diff viewer and no rollback in the
      MVP.** The service layer ([10.4.4](10d-model-requirements.md)) writes a snapshot on every
      mutation and nothing reads it yet.

      ```
      revision(id, entity_type, entity_id, editor_id, created_at, note, snapshot jsonb)
      ```

      **The versioning unit is the aggregate, not the row.** A release snapshot carries its formats,
      identifiers, company/label pairs, credits and tracklist, because that is where most edits
      actually land ([2.1.3](02-catalogue-model.md), [2.4.5](02-catalogue-model.md)) and a row-level
      change log would have to invent synthetic field names to describe them. At ~12,000 releases
      ([1.4](01-product.md)) the storage is irrelevant.
      **Why not more.** Three of the four arguments usually made for versioning do not survive our
      scale: vandalism assumes an attacker who will not come, audit assumes someone who will ask, and
      recommendation 3's "retrofitting is painful" overstates the case — an append-only table is
      additive, and adding it late loses only the history before that date. The argument that does
      hold is narrower: **a backup restores the whole database to a point in time and cannot undo a
      single edit noticed three days later**, and the data is users' own uploads, not re-fetchable
      from anywhere ([1.5](01-product.md)).
      **Why not less.** [2.4.7](02-catalogue-model.md) rejected a data-quality field on the ground
      that "the quality signal is the edit history". Dropping history would reopen that decision.
      **What deferring buys back.** The expensive part of versioning is never the table — it is the
      diff renderer, the revert flow and the "history" tab. Those are built when something needs
      them, and the data to build them on is already there. The cost of being wrong is asymmetric:
      snapshots nobody reads cost one INSERT per mutation; snapshots never written cost history that
      cannot be reconstructed.
      **Two things this table is the answer to later:** the mass-revert tool
      ([4.9](04-editing.md)), and the incremental-delivery feed of
      [10.4.7](10d-model-requirements.md)/[10.1.10](10a-public-api.md), whose cursor is
      `revision.id`.
- [x] 4.3 Voting on edits (like MusicBrainz) or instant application (like Discogs)?
      **Decision:** **Instant application. No voting, at any stage we can currently see.**
      Voting presumes an electorate. MusicBrainz's model works because hundreds of people vote;
      here the electorate is one person ([1.4](01-product.md)). A vote queue nobody drains is worse
      than no queue — edits stall, corrections never land, and the site reads as abandoned.
      Reopen only if there are ever enough simultaneous editors to actually disagree, which is a
      problem we would be lucky to have.
- [x] 4.4 Duplicates: how do we find them and how do we **merge** entities while preserving references from users' collections.
      **Decision:** **Advisory detection at create time, merge as a moderator operation, and merge is
      non-destructive by construction.**
      **Finding them.** On create, we search existing entries and show likely matches before the
      save — advisory only, never blocking, and never a uniqueness constraint
      ([2.4.9](02-catalogue-model.md) is an invariant). Signals in descending confidence: a matching
      identifier (barcode, matrix/runout) → normalised artist + title + catalogue number →
      normalised artist + title. Normalisation is case and punctuation only; it never transliterates
      ([2.2.1](02-catalogue-model.md)).
      **Merging.** The loser keeps its row and gains `merged_into`; its ID keeps resolving and the
      page redirects. Every reference is repointed: collection items, wantlist items, images,
      credits, and `release.master_id` for a master merge. A revision is written on both entities.
      **No field is ever copied from loser to winner.** Want the loser's better catalogue number?
      Edit the winner first, then merge. This keeps merge to a pointer plus repointing, keeps unmerge
      to the reverse, and means a wrong merge never destroys a value.
      **Master merge is early scope, not late** — [2.1.2](02-catalogue-model.md) makes duplicate
      masters the normal state after an import, so merge is the mechanism by which the catalogue
      becomes correct at all.
      **Accepted consequence of moderator-only.** Post-import dedup is moderator work, and in the
      MVP an ordinary user who spots a duplicate has no lever at all. They gain one when
      [4.6](04-editing.md)'s reports are built, where "duplicate of X" is expected to be the most
      used reason.
- [x] 4.5 Entity deletion: hard or soft? What if the release is in somebody's collection.
      **Decision:** **Soft only.** Nothing a user's collection can point at is ever hard-deleted —
      already an invariant from [2.4.9](02-catalogue-model.md), fixed here in its details.
      `deleted_at`, `deleted_by` and a reason on the entity. The URL keeps resolving to a tombstone
      that says the entry was removed; collection items pointing at it keep working and are shown as
      deleted rather than silently vanishing ([3.11](03-collection.md)). Moderator-only, and
      reversible — undelete clears the columns.
      **Delete is not merge.** Merge says "this is the same object as that one" and redirects; delete
      says "this object does not exist" and does not. Using merge as a tidier delete would put a
      user's item on a release they do not own.
      Hard deletion exists only as database surgery outside the application, for content we are
      legally obliged to remove ([section 13](13-legal.md)). It is not a feature and has no UI.
- [x] 4.6 Reports/flags for incorrect data, a moderation queue.
      **Decision:** **Shape fixed now, built after the MVP.** [1.9](01-product.md) puts the
      moderation queue outside the MVP; deciding the table now costs nothing and stops it being
      designed under pressure later.

      ```
      report(id, target_type, target_id, reporter_id, reason, note,
             status, resolved_by, resolved_at)
      ```

      `reason` is a curated set with a free-text `note` beside it rather than inside it — the
      vocabulary invariant applies here as everywhere. `status` is `open` or `closed`; no assignment,
      no priority, no SLA. Expected initial reasons: duplicate of X, wrong data, wrong image,
      offensive content.
      Resolving a report is a moderator action ([4.7](04-editing.md)). The queue is a filtered list,
      not a workflow engine.
- [x] 4.7 Roles: user / trusted / moderator / admin. What rights does each have.
      **Decision:** **Three roles, not four: `user`, `moderator`, `admin`.**
      | role | rights |
      |------|--------|
      | `user` | create and edit any catalogue entity, upload images, manage own collection, file reports |
      | `moderator` | + merge, soft delete and undelete, resolve reports, approve vocabulary additions, block a user |
      | `admin` | + grant roles, configuration, operational access |
      **`trusted` is dropped.** It only makes sense as a tier you are promoted into automatically,
      and [4.8](04-editing.md) declines to compute the reputation that would promote you. A manually
      granted `trusted` is a moderator under a quieter name.
      Rights are enforced in the service layer ([10.4.4](10d-model-requirements.md)), never in the
      UI alone — [10.1](10a-public-api.md) must apply exactly the same rules.
      In the MVP every account is a `user` and the maintainer is `admin`. The two upper tiers are
      schema and a permission check, not a feature to build.
- [x] 4.8 Reputation and contribution (how many edits were accepted) — do we need gamification?
      **Decision:** **No gamification.** No points, badges, levels or leaderboards, and nothing
      anywhere is gated on a contribution count — [4.1](04-editing.md) already decided that rights
      come from roles, not from history.
      What we do show: a plain **edits made** count on the profile, derived from `revision`
      ([4.2](04-editing.md)). It costs one query, it is honest, and it is not a score worth farming.
      Note that "how many edits were **accepted**" is not a question our model can ask — with instant
      application ([4.3](04-editing.md)) there is no acceptance step.
      Rejected: Discogs-style contributor ranks. They exist to allocate moderation privileges by
      earned standing, which is exactly the mechanism [4.1](04-editing.md) declined; at our volume
      the leaderboard would have one entry on it.
- [x] 4.9 Protection against vandalism and mass spam: rate limits, verified-email requirement, honeypot, reverting all of a user's edits.
      **Decision:** **Verified email to edit, per-user rate limits, blocking — and mass revert kept
      as a capability rather than built.**
      - **Verified email is required to edit**, not merely to register. It is the only barrier
        [4.1](04-editing.md) imposes, so it carries real weight; the verification mechanics belong to
        [section 6](06-accounts.md).
      - **Rate limits** per user per hour on entity creation and on edits, set generously enough that
        a genuine cataloguing session never notices them. They exist to bound damage, not to pace
        contributors.
      - **Blocking** a user is a moderator action ([4.7](04-editing.md)). It stops writes, not reads.
      - **No captcha and no honeypot in the MVP.** Both are a real cost against a threat that a
        portfolio-scale site with a verified-email wall does not have ([1.4](01-product.md)).
        Revisit if open registration ever attracts anything.
      - **"Revert all edits by user X since T" is deliberately not built.** [4.2](04-editing.md)'s
        snapshots make it a day's work the moment it is needed, and that option is precisely what
        storing them buys. Building the tool before the vandal is spending against an imaginary
        threat; storing the data is not.
- [x] 4.10 Audit log: who, what, when, from where (IP) — what is the retention period.
      **Decision:** **Two separate logs, with different contents, audiences and retention.**
      1. **Content history** — `revision` ([4.2](04-editing.md)). Who changed what, when, and the
         resulting state. **Permanent, never pruned, and carries no IP address.** It is catalogue
         data, and it becomes public the day a history page ships.
      2. **Security log** — authentication events, role grants, blocks, deletions and merges. Carries
         IP and user agent. **Admin-only, retained about 90 days**, then dropped.

      The split is the actual decision. Putting IPs on content revisions makes personal data both
      permanent and eventually public, and mixing login events into catalogue history makes both
      logs harder to read. The 90 days is a starting figure chosen to be short; it interacts with
      [section 13](13-legal.md) and [section 14](14-security.md) and is not a legal analysis.
- [x] 4.11 Data entry guidelines (our own equivalent of the Discogs Database Guidelines) — are they needed and who writes them.
      **Decision:** **Yes — one short page, written by the maintainer, as a
      [section 15](15-roadmap.md) task rather than something to write now.**
      They are needed because the decisions already taken are worthless if only their author knows
      them: transcribe from the object and never translate or transliterate
      ([2.2.1](02-catalogue-model.md)), object facts on the release and work facts on the master
      ([2.1.1](02-catalogue-model.md)), a stub artist with nothing but a name is fine
      ([2.2.6](02-catalogue-model.md)), when to merge rather than edit ([4.4](04-editing.md)), and
      how to ask for a vocabulary addition instead of forcing a value.
      Explicitly **not** Discogs' guidelines. Theirs run to tens of thousands of words because they
      are the constitution of a marketplace with a voting system and real money at stake. Ours is one
      page that grows a paragraph each time a moderator decision proves it needed one.
      Same roadmap bucket as seeding the vocabularies — both are hand-written content that
      [1.5](01-product.md) forbids fetching, and neither has been estimated.

## Working notes

- **2026-08-15 — What "no revert UI" actually means in practice.** Until a diff viewer exists, fixing
  a bad edit means editing the entity again by hand, and fixing a bad merge means clearing
  `merged_into` and repointing — a moderator with database access, not a button. That asymmetry is
  the reason [4.1](04-editing.md) keeps merge and delete on the moderator side; if a revert UI is
  ever built, reopening the question of who may merge is reasonable.
- **2026-08-15 — The revision table has three future customers, and they disagree about shape.**
  A diff viewer wants adjacent snapshots, mass revert wants everything by one editor, and the
  incremental feed ([10.4.7](10d-model-requirements.md)) wants everything after a cursor. All three
  are served by the same table, but the feed one implies `revision.id` must be monotonic and stable —
  worth remembering when the storage engine is chosen ([section 11](11-stack.md)).

# 15. Roadmap

**Priority:** P0 — the outcome of the discussion

**What this section is not free to decide.** [1.9](01-product.md) already wrote the MVP scope and
its out list, [1.10](01-product.md) the success criteria, [1.11](01-product.md) the pace (solo,
2–3 hours a day, no deadline, stages sized in weeks and tasks sized to one sitting), and
[1.12](01-product.md) four known gaps to budget for rather than discover. Two sequencing rules were
fixed elsewhere and are not this section's to revisit: **export ships before import**, and **master
merge is early, not late**. What this section does is cut the agreed scope into stages, put them in
an order that is defensible line by line, and name the risks.

- [x] 15.1 Slice everything above into stages: E0 (skeleton), E1 (MVP), E2, E3…
      **Decision: four stages and a fifth that is not scheduled. E1 is [1.9](01-product.md)'s round
      trip, cut into nine slices whose order is itself a decision.**

      | Stage | What it is | Rough size |
      |---|---|---|
      | **E0** | Skeleton: it is on the internet, CI is green, backups restore | 3–4 weeks |
      | **E1** | The MVP — [1.9](01-product.md)'s collector's round trip | 4–5 months |
      | **E2** | Messaging ([section 5](05-messaging.md)) | 3–4 weeks |
      | **E3** | Depth: moderation tooling and the parts of the schema shipped switched off | open-ended |
      | **E4** | Not scheduled: the public API ([10.1](10a-public-api.md)), the catalogue dump ([10.3.4](10c-export.md)) | — |

      **The sizes are guesses with [1.11](01-product.md)'s 2–3× multiplier already inside them**, at
      15–20 hours a week. They exist to make "E1 is months, not weeks" explicit; nothing is
      date-driven ([1.11](01-product.md)) and scope is cut by interest and value rather than by a
      calendar.

      **E0 — skeleton.** Repository layout and the three libraries ([11.11](11-stack.md)); config
      decoded at startup ([12.6](12-infrastructure.md)); the migration runner and migration 001;
      **the job queue table and runner** ([10.4.5](10d-model-requirements.md) puts it here, before
      its first consumer); the base layout, the stylesheet and **the form helper**
      ([14.3](14-security.md) — before the first form exists, or it is retrofitted forever); CI's
      five checks ([12.3](12-infrastructure.md)); the box, nginx, TLS, systemd, journald;
      the backup job and **[9.3](09-nfr.md)'s first restore drill**; the runbook; `plan/`
      ([12.8](12-infrastructure.md)). Exit: a deployed application, a green pipeline, and a restore
      that has actually been performed.

      **E1 — the MVP, in this order:**
      1. **Accounts** ([section 6](06-accounts.md)) — register, verify, log in, reset, change email,
         profile, the three settings ([6.6](06-accounts.md)), deletion by anonymisation
         ([6.5](06-accounts.md)). Plus [14.2](14-security.md)'s password answer and
         [14.5](14-security.md)'s limits.
      2. **Catalogue** ([section 2](02-catalogue-model.md), [section 4](04-editing.md)) — the four
         entities, the vocabularies as seed data ([11.8](11-stack.md)), pages and URLs
         ([7.5](07-search-ux.md)), create and edit forms, revisions on every mutation
         ([4.2](04-editing.md)), `external_id` ([10.4.1](10d-model-requirements.md)).
      3. **Collection and wantlist** ([section 3](03-collection.md)) — add from a release page,
         listing, tags, note, conditions, statistics, visibility and the share link.
      4. **Merge and delete** ([4.4](04-editing.md), [4.5](04-editing.md)).
      5. **Export** ([10.3](10c-export.md)) — both formats, both files, `item_id` and the JSON
         header ([10.3.2](10c-export.md)), and [11.10](11-stack.md)'s round-trip property as far as
         it can go without an importer.
      6. **Importer, our own format only** ([10.2](10b-import.md)) — the background job with
         progress, [10.2.4](10b-import.md)'s three-rung ladder, stub minting, the report, rollback,
         idempotency. **Closes the round-trip property with no third-party file involved.**
      7. **Discogs converters** ([10.2.1](10b-import.md)) — collection CSV and wantlist CSV to our
         format: pure, property-tested, no database access, carrying their findings into the same
         report. This is the slice that lets a real person start.
      8. **Search** ([section 7](07-search-ux.md)) — FTS, the `search_tsv` column, facets,
         autocomplete.
      9. **Images** ([section 8](08-media.md)) — bucket, upload, `vips` derivatives, eight per
         release, the avatar.

      **Why that order, since it is the part worth arguing.** Accounts first because everything else
      is user-scoped and [6.1](06-accounts.md)'s barrier is load-bearing for four other decisions.
      Catalogue before collection because `collection_item.release_id` is `NOT NULL`
      ([10.4.3](10d-model-requirements.md)) — there is nothing to collect until releases exist.
      **Merge before export and import**, which is [1.9](01-product.md)'s "early, not late" made
      concrete: the importer mints a master per row ([2.1.2](02-catalogue-model.md)), so the first
      real import lands in a system that must already be able to repair itself, and export must
      serialise merge pointers correctly anyway. Export before import is
      [1.9](01-product.md)'s rule — it needs no external format knowledge and delivers
      [1.3](01-product.md)'s "never locked in" immediately — and after
      [10.2.1](10b-import.md)'s restructuring it is also a hard dependency, since the export file
      **is** the importer's input format. **The importer before the converters**, which is the same
      reasoning one step further on: the job, the ladder, the report and rollback must be provably
      correct against a format we control before a third party's comma-packed, sentinel-bearing file
      is pointed at them, and slice 6 closes [1.10](01-product.md)'s round-trip criterion on its
      own. Search after converters because tuning a
      search over an empty catalogue is guesswork. **Images last, deliberately**: they are the one
      irreversible decision in the design ([8.1](08-media.md) discards originals) and
      [1.4](01-product.md)'s predicted first cost overrun, so they are the thing least suited to
      being half-built while everything else moves.
      **Exit from E1 is [1.10](01-product.md)'s two criteria**, not a feature list.

      **E2 — messaging.** [1.9](01-product.md) put it out of the MVP because at tens of users there
      is nobody to message; it is the natural next vertical because [section 5](05-messaging.md)
      needs **nothing new from the platform** — five tables and one email template, no real-time
      transport ([5.3](05-messaging.md)), no storage, no search.
      **E3 — depth.** [4.6](04-editing.md)'s report queue and moderator tooling; the four things
      [1.9](01-product.md) ships switched off (artist photos, the companies UI, track-level credits,
      membership editing); tracklists by hand; vocabulary expansion. Open-ended by nature — this is
      where the project lives after the round trip works.
      **E4 — not scheduled.** [10.1](10a-public-api.md) found no audience for the API and
      [10.3.4](10c-export.md)'s dump is a licence decision with a small job attached. Both are listed
      so that "later" has a name, not because either is planned.
- [x] 15.2 Record an explicit **out-of-scope** list for the first version.
      **Decision: two lists, because "never" and "not yet" are different promises and conflating them
      is how a permanent refusal quietly becomes a backlog item.**
      **Permanently out — decided, with the item that decided it:** a marketplace or anything
      order-shaped ([1.7](01-product.md)); monetisation of any kind ([1.6](01-product.md)); any
      server call to an external music database, including dumps and an admin with `psql`
      ([1.5](01-product.md), [10.2.10](10b-import.md), [10.5.2](10e-legal-sources.md)); Discogs live
      sync and write-back ([10.2.1](10b-import.md), [10.2.11](10b-import.md)); social features
      beyond messaging — comments, reviews, ratings, following, feeds, a forum
      ([5.11](05-messaging.md)); real-time transport of any kind ([5.3](05-messaging.md),
      [5.9](05-messaging.md)); OAuth login ([6.1](06-accounts.md)); a second UI language
      ([1.8](01-product.md)); reputation, ranks and voting on edits ([4.3](04-editing.md),
      [4.8](04-editing.md)); CAPTCHA ([14.5](14-security.md)); third-party scripts, analytics and a
      cookie banner ([13.4](13-legal.md)); a JavaScript build step ([11.3](11-stack.md)); catalogue
      writes through the API and webhooks ([10.1.7](10a-public-api.md),
      [10.1.10](10a-public-api.md)); audio previews and streaming links ([8.5](08-media.md)).
      **Out of the first version but not refused:** messaging (E2); the public API
      ([10.1.1](10a-public-api.md) — and its realistic substitute is a token on the export URL);
      the catalogue dump ([10.3.4](10c-export.md), blocked on a real legal read at volume); 2FA
      ([6.2](06-accounts.md), deferred not refused — TOTP for everyone if reopened); anything
      money-shaped on a collection item ([3.2](03-collection.md), two nullable columns if reopened);
      a moderation queue and moderator tooling (E3); artist photos, the companies UI, track-level
      credits and membership editing — **already in the schema and switched off**
      ([1.9](01-product.md)); tracklists by hand; `master_title` variant rows, the additive fix for
      [7.4](07-search-ux.md)'s known search gap; WAL archiving and PITR ([9.3](09-nfr.md)); error
      tracking ([9.5](09-nfr.md), the most likely of all of these to be reopened first); a staging
      environment ([12.4](12-infrastructure.md)); the diff viewer, revert flow and change feed over
      [4.2](04-editing.md)'s history, which is **stored from day one and shown to nobody**.
      **The asymmetry that runs through the second list, worth stating once**: where the data is
      unrecoverable and the tool is a day's work, we store the data and skip the tool — edit history
      ([4.2](04-editing.md)), mass revert ([4.9](04-editing.md)), `last_read_at`
      ([5.6](05-messaging.md)). Every one of those is in the second list, not the first.
- [x] 15.3 Define the "vertical slice" for the first week of work (for example: artist → album → release → add to collection, without edits and messages).
      **Decision: not the item's example — that one needs accounts and forms and is a month. Week one
      is *one release page, rendered from Postgres, through every layer, with CI green*.**
      The example slice is right in spirit and wrong in size: "add to collection" requires
      [section 6](06-accounts.md), sessions, forms and CSRF, which is E1.1 plus E1.2 plus E1.3.
      **What week one actually contains** (15–20 hours, [1.11](01-product.md)):
      1. Repository, `cabal.project` with a pinned GHC and `index-state`, the three libraries and the
         executable ([11.11](11-stack.md)).
      2. `docker compose` with Postgres and MinIO ([12.2](12-infrastructure.md)); `hasql`
         connection; config decoded at startup and refusing to boot on a bad value
         ([12.6](12-infrastructure.md)).
      3. The migration runner and **migration 001** — `artist`, `master`, `release`, minimal
         columns, `bigint GENERATED ALWAYS AS IDENTITY` ([10.4.6](10d-model-requirements.md)).
      4. **One service function** in `musilka-app` ([10.4.4](10d-model-requirements.md)), one Servant
         route `/release/:id/:slug` ([7.5](07-search-ux.md)), one Lucid page, one stylesheet, and
         `/healthz` ([9.5](09-nfr.md)).
      5. CI: build, format, lint, and one integration test against a real Postgres
         ([11.10](11-stack.md) level 2, [12.3](12-infrastructure.md)).
      6. Deploy to the box: systemd, nginx, TLS.
      **Why this slice and not a bigger one.** It touches every layer of
      [11.11](11-stack.md)'s split exactly once — migration, codec, service, route, template — and
      three of [1.12](01-product.md)'s four known gaps are in it (`ReaderT AppEnv IO` in the large,
      the database layer, build and deploy). It is a rehearsal of the whole architecture with one
      column of business logic, which is the cheapest possible way to find out that the shape is
      wrong.
      **Two honest caveats.** Step 6 is blocked while [12.1](12-infrastructure.md) has no provider,
      so if that is still open, week one ends at green CI and the deploy slips — the work does not.
      And **production will show an empty catalogue**, because [10.2.10](10b-import.md) forbids
      loading catalogue rows by any channel including `psql`: the release page is exercised by
      [11.8](11-stack.md)'s development fixtures and by the CI test, and the first real row arrives
      through a form in E1.2.
- [x] 15.4 Risks and open questions requiring research (data import, search, volumes).
      **Decision: eight, each with what would tell us it is happening and what we do about it.
      Ranked by how much of the project each one threatens.**
      1. **Haskell pace, and [1.12](01-product.md)'s four gaps.** The largest risk by far, and E0
         concentrates three of the four. *Signal:* E0 running well past four weeks, or E1.1 not
         converging. *Response:* [11.2](11-stack.md) fixed the rule — reopening the language happens
         on evidence from a built E0/E1, never on a bad week, and switching means restarting the
         codebase.
      2. **Transactional mail deliverability.** [6.1](06-accounts.md)'s verified-email barrier is
         what [4.9](04-editing.md), [5.7](05-messaging.md) and [14.5](14-security.md) all lean on
         instead of a captcha; a fresh domain landing in spam breaks all four at once.
         *Signal:* the check [12.5](12-infrastructure.md) requires before
         [1.10](01-product.md)'s stranger is invited. *Response:* SPF, DKIM, DMARC published with the
         first deploy setup, and reputation given time.
      3. **The Discogs collection export's real columns** — still unverified
         ([10.2.2](10b-import.md)), along with the default folder name ([10.2.5](10b-import.md)) and
         whether an instance id exists ([10.2.7](10b-import.md)). *Mitigated twice over:* the parser
         resolves columns by header through a lookup table and treats every column as optional, so a
         wrong guess costs a table row and surfaces in the report — and since
         [10.2.1](10b-import.md), the whole guess lives inside a pure converter that the importer,
         the job, the schema and the round-trip test do not depend on. **Get a real collection export
         before E1.7** — it is the cheapest research task on this list, and it is now the only thing
         that slice is waiting on.
      4. **Vocabulary seed data is unestimated.** [NOTES.md](NOTES.md) has flagged it as a real
         roadmap task nobody has sized: genres, styles, credit roles, company roles, format
         descriptors, identifier types and country codes including `SU`/`YU`/`DD`/`CS`, all
         hand-written because [1.5](01-product.md) forbids fetching them. *Response:* size it during
         E1.2, and start deliberately small — [4.x](04-editing.md)'s moderation path exists so users
         can request what is missing.
      5. **Images breaking the cost estimate**, which [1.4](01-product.md) predicted and
         [9.3](09-nfr.md) attached a weekly sync to. *Signal:* the bucket approaching
         [8.2](08-media.md)'s ~6.5 GB, and the sync's time and egress rather than its storage bill.
         *Response:* the caps are already in place; [8.1](08-media.md)'s discarding of originals is
         the irreversible part and is why images are E1's last slice.
      6. **Search quality on a thin, multi-script catalogue.** [7.4](07-search-ux.md)'s known gap —
         artists are findable in any script but titles have no variant rows — is a recorded
         limitation, not a surprise. *Response:* the additive `master_title` table if a real user
         actually hits it.
      7. **`vips` on a small box.** [14.4](14-security.md) calls it the largest attack surface in the
         system and [8.3](08-media.md) runs it in the request. *Response:* already specified —
         subprocess, dimension cap, memory limit, timeout, systemd hardening
         ([12.2](12-infrastructure.md)).
      8. **[1.10](01-product.md)'s second success criterion depends on finding a willing collector**,
         which is outside the author's control. [1.10](01-product.md) already says to distinguish
         that from failure; the roadmap consequence is to start looking during E1, not after it.
      **The legal residual is not on this list and is not forgotten**: [NOTES.md](NOTES.md)'s EU
      database-rights note is accepted for ingest and becomes live only at
      [10.3.4](10c-export.md)'s dump, which is E4 and gated on a real legal read.
- [x] 15.5 Decide separately at which stage the public API ([10.1](10a-public-api.md)), import ([10.2](10b-import.md)) and export ([10.3](10c-export.md)) appear, and which "hooks" for them already land in E0–E1 ([10.4](10d-model-requirements.md)).
      **Decision: export in E1.5, the importer in E1.6, the Discogs converters in E1.7, the API not
      scheduled at all — and the hook list is shorter than [1.9](01-product.md) feared, because
      [section 10.4](10d-model-requirements.md) turned most of it into things we were building
      anyway.**
      **Export — E1.5, before import**, per [1.9](01-product.md). It needs no external format
      knowledge, it is synchronous with no job ([10.3.3](10c-export.md)), and it delivers
      [1.3](01-product.md)'s fourth value proposition on the day it ships. Since
      [10.2.1](10b-import.md), it is also the importer's input format, so the dependency is
      structural rather than merely preferred.
      **Import — E1.6**, immediately after, and it is the round trip's closing move:
      [11.10](11-stack.md)'s export → import → export property cannot run until it exists, and that
      property is [1.10](01-product.md)'s first success criterion turned into a test. **This slice
      reads our own format only** and needs no third-party file to be finished or tested.
      **Discogs converters — E1.7**, and they are **not optional despite being a separate slice**:
      our own format exists only after someone has used the site, so until E1.7 ships there is no
      way for a real collector to get in, and [1.10](01-product.md)'s criteria both stay unreachable.
      The separation buys blast radius, not deferral — [10.2.2](10b-import.md)'s unverified columns
      are confined to a pure module, and a second source later is another slice of the same shape.
      **The API — not scheduled** ([10.1.1](10a-public-api.md) found no audience). If the underlying
      need ever appears it is a **personal access token accepted on the export URL**, roughly a day's
      work, and that is the roadmap item to write rather than "build the API".
      **Hooks that must land in E0–E1 because retrofitting them is the expensive rework:**
      | Hook | Stage | Fixed by |
      |---|---|---|
      | Service layer, every mutation, explicit actor | E0 | [10.4.4](10d-model-requirements.md) |
      | Job queue table and runner | E0 | [10.4.5](10d-model-requirements.md) |
      | `bigint` identity ids on every entity | E0 (migration 001) | [10.4.6](10d-model-requirements.md) |
      | `external_id` with `added_by`/`added_at` | E1.2 | [10.4.1](10d-model-requirements.md) |
      | Revisions on **every** mutation, including merge and delete; monotonic ids | E1.2 / E1.4 | [10.4.7](10d-model-requirements.md) |
      | `revision.source` and `revision.import_id` | E1.2 | [10.4.2](10d-model-requirements.md) |
      | The export column contract, public from the first file | E1.5 | [10.3.2](10c-export.md) |
      | `import.file_bytes` plus the 7-day sweeper | E1.6 | [10.4.8](10d-model-requirements.md) |
      **[1.9](01-product.md) named three of these; the list is now eight and none of the five
      additions is new work** — each is a column or a property of something already being built.
      **What needs no hook at all, so nobody builds one "just in case":** the API
      ([10.1](10a-public-api.md)'s working notes list what it would need and every item already
      exists for another reason), webhooks ([10.1.10](10a-public-api.md) refuses them permanently),
      and [10.3.4](10c-export.md)'s dump (the same serialisers plus a cron entry).
- [x] 15.6 Once the key items are closed — produce `PLAN.md` with concrete tasks.
      **Decision: not `PLAN.md` — [`plan/`](../plan/README.md), split by stage exactly as this
      section's 2026-08-14 working note proposed and [12.8](12-infrastructure.md) confirmed. Written
      2026-08-15: 190 tasks, `T-1` … `T-190`, across four files.**
      **The shape, and why it is this shape rather than one file.** Cut by **stage**, not by design
      section: a coding session inside E1 never needs E3, which is the property that makes the split
      pay, while cutting by module would mean opening most of the folder every session. The file
      names come from [15.1](15-roadmap.md) — `plan/E0-skeleton.md` (T-1 … T-33),
      `plan/E1-mvp.md` (T-34 … T-160, under the nine slices in their argued order),
      `plan/E2-messaging.md` (T-161 … T-173), `plan/E3-depth.md` (T-174 … T-190) — plus
      `plan/README.md` as the index and the conventions.
      **E4 deliberately has no file.** [15.1](15-roadmap.md) named it so that "later" has a name;
      writing tasks for the public API or the catalogue dump would make them look scheduled.
      **The conventions are `design/`'s, on purpose** — one set of rules to remember. Task IDs are
      permanent and **global across the directory**, so a new task takes the next free number rather
      than a number near its neighbours. Every task **cites the design item that justifies it**
      ([12.8](12-infrastructure.md)'s first reason: `grep -rn "10.4.1" design/ plan/` finds both the
      decision and the work). Progress is derived by `grep`, never stored. `[ ]` open, `[x]` done,
      `[~]` blocked with the blocker named, `[-]` dropped. A task is **one sitting**
      ([1.11](01-product.md)); one that is not is two tasks.
      **Three things the writing surfaced that this section had not said.** (1) **Twelve tasks are
      `[~]` at birth** — nine in E0 plus T-151, T-157 and T-122 — and all but T-122 trace to
      [12.1](12-infrastructure.md)/[12.5](12-infrastructure.md)'s two unnamed vendors. (2) **Three
      tasks sit outside the slice their design section belongs to**, each for a dependency reason:
      the trigram index and artist picker land in E1.2 because every credit must resolve to an
      artist entity ([7.4](07-search-ux.md)), the faceted list is built in E1.3 and reused by E1.8,
      and [4.6](04-editing.md)'s `report` table lands in E1.2 because image takedown and vocabulary
      requests both need a row to write long before E3 gives it a queue. (3) **E1 needed an exit
      checklist that is not a slice** — [13.3](13-legal.md)'s four documents,
      [4.11](04-editing.md)'s guidelines, [7.8](07-search-ux.md)'s robots and sitemap,
      [12.5](12-infrastructure.md)'s deliverability check, the accessibility pass and
      [9.3](09-nfr.md)'s second restore drill are each easy to leave undone for ever, and none of
      them belongs inside a feature slice.
      **What is not in `plan/`:** no new decisions. Where a task looked like a choice, the choice was
      made in `design/` and the task is its implementation; where the design left something open, the
      task carries `[~]` and names what it waits on.

## Working notes

**2026-08-14 — shape of the plan (proposal, not decided).** Mirror the `design/` split, but cut by
**stage rather than by design section**: `plan/E0-skeleton.md`, `plan/E1-mvp.md`, … A coding session
inside E1 never needs E3, which is the property that makes the split pay; cutting by module instead
would mean opening most of the folder every session.

- Do not create `plan/` before [15.1](15-roadmap.md) fixes the stage list — the file names come from
  it, and E0 alone may be small enough for a single file.
- Task IDs permanent, same rule as design items, and each task cites the design item that justifies
  it (`T-14 external_ids table (per 10.4.1)`). That backlink is the main reason to keep the plan in
  the repo next to the design.
- No stored progress counters — derive with `grep`, as in `CLAUDE.md`.
- Depends on [12.8](12-infrastructure.md): if work is tracked in a real issue tracker rather than
  markdown, `plan/` shrinks to a thin index and most of this note is moot. Settle 12.8 first.

**2026-08-15 — both dependencies in that note are now resolved, and the proposal survives intact.**
[12.8](12-infrastructure.md) chose `plan/*.md` in the repository as the single source of truth, for
exactly the backlink reason above, with GitHub Issues kept only as an inbox for outside reports.
[15.1](15-roadmap.md) fixed the stage list, so the file names are known:
`plan/E0-skeleton.md`, `plan/E1-mvp.md`, `plan/E2-messaging.md`, `plan/E3-depth.md`. E1 is eight
slices and will want internal headings rather than eight files. E4 gets no file — it is not
scheduled ([15.1](15-roadmap.md)).

**2026-08-15 — Section closed except [15.6](15-roadmap.md), which is the deliverable rather than a
decision.** Five items decided. Four of them are compilations rather than new choices —
[15.2](15-roadmap.md) gathers refusals already made, [15.5](15-roadmap.md) gathers hooks already
required, and [15.1](15-roadmap.md)'s stage boundaries follow [1.9](01-product.md)'s scope line. The
genuinely new material is **the order of E1's nine slices** and its argument,
[15.3](15-roadmap.md)'s week-one slice, and [15.4](15-roadmap.md)'s ranking of the risks.

**2026-08-15 — What [15.6](15-roadmap.md) is now waiting on: nothing but the writing.** Every P0
section is closed and the only open items anywhere in `design/` are
[10.2.2](10b-import.md) (`[~]`, needs a real collection export, and after
[10.2.1](10b-import.md)'s restructuring it blocks E1.7 alone) and
[12.1](12-infrastructure.md)/[12.5](12-infrastructure.md) (`[~]`, hosting and mail vendors,
deferred by choice — they block the first deploy, not the plan). `plan/E0-skeleton.md` can be
written from [15.3](15-roadmap.md) and [12.x](12-infrastructure.md) today.

**2026-08-15 — Section closed. [`plan/`](../plan/README.md) is written and this section is done.**
190 tasks in four files; the design is no longer the place work is tracked. Two consequences worth
stating. **First, `design/` is now read-only in a specific sense:** it is still the single source of
truth for *what was decided and why*, and a task that needs a decision changed reopens the design
item rather than deciding it in `plan/`. **Second, the backlink is load-bearing in both directions**
— `grep -rn "<item>" design/ plan/` is the navigation, so a task without its citation and a decision
without a task are both defects. [12.7](12-infrastructure.md)'s commit convention closes the loop by
citing the same item from the commit.

**2026-08-15 — Where the design's remaining `[~]`s land in the plan**, so that closing them is a
scheduled act rather than a discovery. [10.2.2](10b-import.md) becomes `T-122` and is the *only*
thing E1.7 waits on — the cheapest research task in the project and the one that decides whether
`T-132` exists at all. [12.1](12-infrastructure.md) and [12.5](12-infrastructure.md) become the nine
`[~]` tasks in E0 plus `T-151` and `T-157`; **`T-22` is the one with teeth**, because the domain and
its SPF/DKIM/DMARC records must be published in the same sitting as the box is provisioned or
[6.1](06-accounts.md)'s barrier quietly breaks.

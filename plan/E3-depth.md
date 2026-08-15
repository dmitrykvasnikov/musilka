# E3 — Depth

**Size:** open-ended by nature ([15.1](../design/15-roadmap.md)) · **Tasks:** T-174 … T-190

**This is where the project lives after the round trip works.** Two kinds of work, and it is worth
keeping them apart: **turning on what already shipped switched off** (T-174 … T-179 — each is a UI
over a table that exists, with no migration), and **building the tools whose data has been
accumulating since day one** (T-180 … T-185 — each is the payoff for
[4.2](../design/04-editing.md)'s and [5.6](../design/05-messaging.md)'s asymmetry: store what cannot
be reconstructed, defer what can be built later from it).

**Nothing here is ordered.** Unlike E1, no task in this stage blocks another, and
[1.11](../design/01-product.md) says scope is cut by interest and value rather than by a calendar.
Take them in whatever order the catalogue actually demands.

## Switched on, not built

Each of these is on [15.2](../design/15-roadmap.md)'s "out of the first version but not refused"
list, and each is **already in the schema**.

- [ ] T-174 Companies UI (per 2.4.6, 2.1.5)
      `release_company` already carries `role` — `pressed_by`, `distributed_by`,
      `phonographic_copyright`, `copyright`, `made_by`, `recorded_at`, `mastered_at` — and the MVP
      showed only `label`. This is a form and a rendering; there is **no migration**.
      It also settles the `Studio`/`Publisher` half of
      [2.1.5](../design/02-catalogue-model.md): they are label rows under a different role, which is
      one of the parts of the Discogs model worth copying literally.
- [ ] T-175 Track-level credits (per 2.4.5, 2.4.4)
      `release_credit.track_id` has been nullable since T-54, written that way precisely so this
      would not be a migration on a populated table. Plus `track_artist_credit`, which is where a
      compilation's real artists live — the release billing being the `Various` singleton.
      **Absence of a track credit means "as the release is credited"**, not "unknown", so only
      compilations and guest spots carry one.
- [ ] T-176 Membership editing (per 2.2.2)
      `artist_relation` with `member_of` and `alias_of` and its partial dates has existed since
      T-52; the artist page has shown members read-only since T-67. This is the edit form.
      No constraint that a `member_of` source is a person — bands do contain bands. The UI may warn;
      the schema does not forbid.
- [ ] T-177 Tracklists by hand (per 2.4.4)
      `release_track` exists with `sequence` (the only thing ordering ever uses), free-text
      `position` (display only — any attempt to parse `A1` into structure breaks on the next
      release), `duration_s`, `kind ∈ track|heading`, and `parent_id` nesting **one level deep,
      enforced**.
      No import channel carries a tracklist, so **every row here is one a human typed** — which is
      why it was never in the MVP and why the global `Track` entity was refused.
- [ ] T-178 Artist photos (per 2.2.6, 8.4, 13.5)
      The one real exclusion in [2.2.6](../design/02-catalogue-model.md), and it is excluded for two
      compounding reasons: images are the cost that scales with the catalogue, and **an artist photo
      is the most copyright-fraught image on the site** — a press or live photograph with an
      identifiable owner, unlike a sleeve scan of an object the uploader holds.
      Do not open this without a legal answer to go with it. The pipeline (E1.9) needs no change.
- [ ] T-179 Vocabulary expansion and the approval path (per 2.3.2, 4.7, 4.11)
      T-77 gave users a lever — a `report` row and an email. This is the moderator side: approve a
      requested value, seed it, and note the decision in T-154's guidelines page if it establishes a
      rule.
      This is the mechanism that makes T-61's "start deliberately small" safe.

## The tools whose data already exists

- [ ] T-180 Report queue and moderator tooling (per 4.6, 8.4, 5.7)
      `report` has been written since T-78 and read by nobody. **A filtered list, not a workflow
      engine**: `status ∈ open|closed`, no assignment, no priority, no SLA. Resolving is a moderator
      action.
      It is also what finally gives an ordinary user a lever on a duplicate — "duplicate of X" is
      expected to be the most used reason ([4.4](../design/04-editing.md), T-98).
- [ ] T-181 Diff viewer and revision history (per 4.2)
      `revision` has carried an aggregate snapshot per mutation since T-56. **The expensive part of
      versioning was never the table** — it is the diff renderer, the history tab and the revert
      flow, and the data to build them on is already there.
      Note that the snapshot is a state, not a patch: a diff is computed from two adjacent
      snapshots. The history page is public catalogue data the day it ships, which is why
      [4.10](../design/04-editing.md) keeps IP addresses out of `revision` and in a separate
      admin-only security log.
- [ ] T-182 Revert (per 4.2, 4.4)
      Writing an older snapshot back as a new revision. Until this exists, fixing a bad edit means
      editing the entity again by hand and fixing a bad merge means clearing `merged_into` and
      repointing — a moderator with database access, not a button.
      **That asymmetry is why [4.1](../design/04-editing.md) keeps merge and delete moderator-only.
      If revert ships, reopening the question of who may merge is reasonable.**
- [ ] T-183 Mass revert (per 4.9)
      "Revert all edits by user X since T". Deliberately not built in the MVP: building the tool
      before the vandal is spending against an imaginary threat, **but storing the snapshots is
      not** — that is precisely what T-56 bought, and this is a day's work the moment it is needed.
- [ ] T-184 Change feed (per 4.2, 10.4.7)
      "Everything since revision N" is one indexed query, because `revision.id` is a monotonic
      `bigint` identity that is never reused and **every mutation writes one, including merge and
      delete** — the two that look like exceptions and whose absence is what makes a change feed
      untrustworthy.
      One honest limit, recorded so it is not rediscovered as a bug: ids come from a sequence, so a
      transaction can commit revision 100 after a reader has seen 101. At one process and a handful
      of edits a day this is theoretical; the fix, if it ever matters, is to page by `created_at`
      with an overlap window, **not to add a table**.
- [ ] T-185 `master_title` variant rows (per 7.4, 7.2, T-140)
      The additive fix for the known search gap: a Latin-typed query for a Cyrillic **title** finds
      nothing, because masters have one title and no variant rows.
      `master_title(master_id, kind, title)` mirroring `artist_name`, indexed the same way — one
      migration and one trigger change. **It was not built earlier because nothing would fill it**;
      build it when hand-typed titles exist to put in it, or when a real user actually hits the gap.
      Not transliteration, which [2.2.1](../design/02-catalogue-model.md) forbids as an invariant.

## Operational, when the trigger fires

- [ ] T-186 Error tracking (per 9.5, 15.2)
      **The most likely of all the deferrals to be reopened first**, and the trigger is specific: the
      moment a user reports a bug we cannot reproduce from T-31's digest.
      Reopening it costs a third-party account, a DSN to keep secret, an entry in T-13's CSP, a
      sub-processor line in T-152's policy, and stack frames containing user data leaving the box.
      All four are real; none is fatal. Decide it against the digest, not against the idea.
- [ ] T-187 WAL archiving and PITR (per 9.3, 15.2)
      Trigger: the day losing 24 hours of edits would actually hurt. It is a second mechanism to
      configure, monitor and **test**, which is the part that makes it expensive — an untested
      recovery path is worse than an honest 24-hour RPO.
- [ ] T-188 Staging environment (per 12.4, 15.2)
      Trigger: a migration goes wrong on production and [9.2](../design/09-nfr.md)'s hand-run rule
      plus T-27's restore-based rehearsal turn out not to have been enough.
      Costs a day to add. Note what it does *not* replace: a staging database full of invented rows
      is a worse rehearsal than a restored copy of the real one.
- [ ] T-189 Security log (per 4.10)
      Authentication events, role grants, blocks, deletions and merges, **with IP and user agent**,
      admin-only, retained about 90 days. Distinct from `revision`, which is permanent, public the
      day T-181 ships, and carries no IP.
      The split is the actual decision: putting IPs on content revisions makes personal data both
      permanent and eventually public.
- [ ] T-190 Personal access token on the export URL (per 10.1.1, 15.5)
      **The roadmap item to write instead of "build the API".** If the underlying need ever appears
      — "dump my collection with a script" — it is a token accepted on T-99's export URL, roughly a
      day's work, and it is the realistic substitute [10.1.1](../design/10a-public-api.md) names.
      It is not a first step towards an API. There is no API, nothing anywhere is reserved for one,
      and three refusals inside [10.1](../design/10a-public-api.md) are permanent rather than
      deferred: **no OAuth2 server, no catalogue writes through an API, and no webhooks, ever.**

## Deferred, with the trigger that would reopen each

Not tasks. [15.2](../design/15-roadmap.md)'s second list, minus what already has a task above, kept
here so that "not yet" does not quietly decay into either "never" or "backlog".

| Deferred | Reopening costs | Trigger |
|---|---|---|
| 2FA ([6.2](../design/06-accounts.md)) | TOTP **for everyone**, never admin-only — a second factor with one user is a code path with no coverage | Anything money-shaped, or a real account compromise |
| Money on a collection item ([3.2](../design/03-collection.md)) | Two nullable columns on `collection_item`, one on `wantlist_item`, plus a total-value tile. Owner-only, and **never converted between currencies** | A demonstrated need, not a hypothetical one |
| Wanting a *master* ([3.5](../design/03-collection.md)) | Additive | Master merge making masters trustworthy |
| Rich previews for release links in messages ([5.4](../design/05-messaging.md)) | One query at render time, no column | Whenever |
| Group chat ([5.1](../design/05-messaging.md)) | Additive only because per-user state already lives on a participant row | A population with groups in it, which we do not expect |
| Message search ([5.10](../design/05-messaging.md)) | One index, one query | Volume that the browser's find cannot handle |
| Catalogue dump ([10.3.4](../design/10c-export.md)) | JSON Lines from the same serialisers, a `LICENSE` file in the archive, a cron entry | **A real legal read first** — NOTES' EU database-rights risk was accepted for ingest and never for redistributing the accretion |
| Arbitrary CSV with column mapping ([10.2.1](../design/10b-import.md)) | A real feature: column preview, per-column target, saved mappings, coercion | A source common enough to be worth automating — at which point a **converter** is the better answer |
| A second import source | A new converter, and nothing else — no schema, job or report change | Whenever one exists |
| A global `work` entity ([2.1.4](../design/02-catalogue-model.md)) | A new table plus a linking pass; tracklist items keep their titles | Enough hand-typed tracklists to link, which T-177 would have to create first |
| `Series` ([2.1.5](../design/02-catalogue-model.md)) | An additive table | A genuine cluster of series-organised releases |

**And the refusals that are not on this list are not deferrals.** A marketplace and anything
order-shaped, monetisation, any server call to an external music database (including an admin with
`psql`), Discogs live sync and write-back, social features beyond messaging, real-time transport,
OAuth login, a second UI language, reputation and voting, CAPTCHA, third-party scripts and a cookie
banner, a JavaScript build step, catalogue writes through an API, webhooks, audio previews and
streaming links — all permanently out, each with the item that decided it recorded in
[15.2](../design/15-roadmap.md). Adding a task for any of them is reopening a design decision, and
that happens in `design/` first.

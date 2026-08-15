# Cross-cutting design notes

Only what belongs to **no single section** lives here. A note about one section goes in that
section file's `Working notes`.

Keep entries short and dated (`YYYY-MM-DD`). When a note hardens into a decision, move it into the
relevant item as a `**Decision:**` block and delete it from here.

## Agreed invariants

Rules that hold across the whole design and must not be quietly broken by a later decision.
Each one names the item it was decided in.

- **2026-08-14 — Two channels in, and no others.** Data enters the system only via (1) a file the
  user uploads or (2) a person typing into our UI. The server never calls an external music
  database, downloads a dump, or scrapes. Any design that needs "we can just fetch it from
  Discogs/MusicBrainz" is invalid. Decided in [1.5](01-product.md).
  **This constrains provenance, not the schema** — no field is forbidden because an external source
  would have been the convenient way to fill it.
- **2026-08-14 — The catalogue is thin by construction.** Entries start with only what a user's
  export carried and are enriched by hand afterwards. Nothing may assume a fully populated release
  (no required tracklists, no required images, no required credits) — but equally, nothing forbids
  those fields existing and being filled in by hand. Follows from [1.5](01-product.md).
- **2026-08-14 — One small box, and a veto list to keep it that way.** No search engine, no Redis,
  no replicas, no CDN, no orchestration — see [1.4](01-product.md) for the full list and the row
  estimates behind it. Cite that item to reject scale-driven proposals in
  [section 9](09-nfr.md) and [section 12](12-infrastructure.md). Exceptions that still get real
  engineering: image storage ([section 8](08-media.md)), import as a background job
  ([10.4.5](10d-model-requirements.md)), and tested backups.
- **2026-08-14 — Nothing order-shaped in the model.** No price, availability or seller state on any
  entity, at any stage. Decided in [1.7](01-product.md).
- **2026-08-14 — UI language and catalogue data are separate concerns.** The interface is English
  only ([1.8](01-product.md)) with no i18n machinery. Catalogue data is independent of that: names
  and titles are recorded as they appear on the object, in any script, and are never translated or
  transliterated by the application. The multi-script problem (`Кино` / `Kino`) is therefore a
  catalogue modelling question for [section 2](02-catalogue-model.md), unaffected by the UI language
  decision.
- **2026-08-14 — Physical media of any format.** Vinyl, CD, cassette and others are equal citizens;
  no format-specific field may be promoted onto the release itself. Decided in
  [1.2](01-product.md).
- **2026-08-15 — Every release belongs to a master.** `release.master_id` is `NOT NULL`, with no
  orphan case anywhere in the model, the UI, the API or the export. Decided in
  [2.1.2](02-catalogue-model.md), against the opening recommendation and with the cost accepted.
  **Consequence that reaches other sections:** the importer mints a master per row, so duplicate
  masters are the normal state and **master merge moves into early scope** alongside release merge —
  [section 4](04-editing.md) cannot treat merging as a late moderation feature.
- **2026-08-15 — Object facts on the release, work facts on the master.** Where a fact could sit on
  either, it goes on the release: the object is ground truth, the master is an interpretation. Decided
  in [2.1.1](02-catalogue-model.md); it is what puts the bootleg flag on the release
  ([2.4.7](02-catalogue-model.md)) and genres on the master ([2.3.2](02-catalogue-model.md)).
- **2026-08-15 — No uniqueness constraint on releases, ever.** Two pressings may differ only in a
  runout etching, and a constraint would make the importer silently discard a user's row. Duplicates
  are handled by advisory detection at create time and by merge afterwards — never by rejection.
  Decided in [2.4.9](02-catalogue-model.md).
- **2026-08-15 — Nothing a user's collection points at is ever hard-deleted.** Merging leaves a
  `merged_into` pointer, the old ID keeps resolving, and collection items follow. Binds
  [section 3](03-collection.md) and [section 4](04-editing.md). Decided in
  [2.4.9](02-catalogue-model.md).
- **2026-08-15 — The application never translates or transliterates a name, and one artist holds
  many names.** `Кино` and `Kino` are one entity with one primary name and variants beside it, all in
  one search index — not two artists, not a translation string. Decided in
  [2.2.1](02-catalogue-model.md), closing the multi-script question [1.8](01-product.md) parked on
  section 2.
- **2026-08-15 — Everything a catalogue entity credits is an artist entity.** No raw-name credits
  anywhere — not in the billing ([2.3.3](02-catalogue-model.md)), not in role credits
  ([2.4.5](02-catalogue-model.md)). A raw name would be a second class of artist that no page links
  to and no search finds. Creating a stub is one click and [2.2.6](02-catalogue-model.md) requires
  nothing of an artist but a name.
- **2026-08-15 — Merge never copies a field, and delete is never hard.** Merging writes `merged_into`
  on the loser and repoints every reference; no value is moved from loser to winner, so a wrong merge
  destroys nothing and unmerge is the reverse operation. Deletion is `deleted_at` plus a tombstone
  that still resolves. Both are moderator-only. Decided in [4.4](04-editing.md) and
  [4.5](04-editing.md), closing what [2.4.9](02-catalogue-model.md) left to section 4.
  **Consequence:** salvaging a better value from the loser is an ordinary edit performed *before* the
  merge, and every UI that offers merge must say so.
- **2026-08-15 — Edit history is written from day one and shown to nobody.** Every mutation writes an
  aggregate snapshot; the diff viewer, revert flow and change feed are deferred until something needs
  them. Decided in [4.2](04-editing.md). The rule this generalises: **store what cannot be
  reconstructed, defer what can be built later from it.** Applies to any "do we need X history"
  question in a later section.
- **2026-08-15 — Rights come from roles, never from history.** Three roles (`user`, `moderator`,
  `admin`), enforced in the service layer so [10.1](10a-public-api.md) obeys the same rules, and
  nothing anywhere is gated on a contribution count or a reputation score. Decided in
  [4.7](04-editing.md) and [4.8](04-editing.md).
- **2026-08-15 — A collection item holds no copy of catalogue data.** No denormalised title, artist,
  year or format on `collection_item` — it is a foreign key plus the facts about *this copy*.
  Decided in [3.1](03-collection.md); it is what makes a catalogue edit a non-event for collections
  ([3.11](03-collection.md)) and what lets statistics read through to the master
  ([3.8](03-collection.md)). Any proposal to cache catalogue fields on the item is a proposal to
  build a synchronisation problem.
- **2026-08-15 — Nothing money-shaped anywhere, and the deferral is itemised.** [1.7](01-product.md)
  already barred price from catalogue entities; [3.2](03-collection.md) now keeps it off the
  collection item too — no purchase price, no currency, no valuation, no maximum price on a wantlist
  item, no total-value statistic. **Deferred, not refused:** reopening is two nullable columns on
  `collection_item` plus one on `wantlist_item`, listed in that section's working notes.
- **2026-08-15 — An import never discards data silently.** Whatever a user's file carries that our
  model has no home for — price columns ([3.2](03-collection.md)), custom fields
  ([3.4](03-collection.md)) — is either folded into a free-text field with its source label or
  dropped, and **either way the importer reports it**. Binds [10.2](10b-import.md). The reason is
  that a silent drop is indistinguishable from a bug, and the user is the only person who still has
  the original file.
- **2026-08-15 — Curated set, with free text beside it rather than inside it.** Every field drawn
  from a vocabulary (genres and styles, format descriptors, identifier types, credit roles, company
  roles) is seeded and closed, with a free-text companion field for what the vocabulary cannot say.
  Users request new values through moderation; they never mint them inline. The reason holds
  wherever it recurs: the field exists to group or filter, free text destroys grouping, and a closed
  set with no escape hatch makes users lie. Established across
  [2.3.2](02-catalogue-model.md), [2.4.2](02-catalogue-model.md), [2.4.3](02-catalogue-model.md),
  [2.4.5](02-catalogue-model.md), [2.4.6](02-catalogue-model.md).
  **The scope is shared vocabularies.** A user's own collection tags ([3.3](03-collection.md)) are
  private labels nobody else groups by, and are freely named — not an exception to the rule but a
  case outside it.
- **2026-08-15 — Messaging is the only social feature, at any stage.** No comments on releases, no
  reviews or ratings, no following, no activity feed, no forum. Decided in
  [5.11](05-messaging.md). The reason generalises past messaging and should be cited whenever a
  discussion venue is proposed in a later section: **a comment is a fact that failed to become an
  edit**, and [section 4](04-editing.md) exists so that observations land in the catalogue rather
  than in a thread beside it. Same principle as [2.4.7](02-catalogue-model.md)'s refusal of a
  data-quality field — the signal must be data, not an assertion parked next to the data.
- **2026-08-15 — No real-time transport anywhere in the system.** No WebSocket, no SSE, no client
  polling loop, no push service, in any feature. Decided for messaging in [5.3](05-messaging.md) and
  [5.9](05-messaging.md), but it is a property of [1.4](01-product.md)'s target shape — one process
  on one small box, deploys that may drop the site for ten seconds — so it binds
  [section 9](09-nfr.md), [section 11](11-stack.md) and [section 12](12-infrastructure.md) equally.
  Timeliness, where it is needed at all, is email.
- **2026-08-15 — Private user content is plaintext in Postgres, and we say so rather than imply
  otherwise.** Message bodies are readable by whoever holds the database; end-to-end encryption is
  meaningless when the same party renders the pages. Decided in [5.8](05-messaging.md). The
  obligation this creates is on [section 13](13-legal.md) (the privacy policy states it) and
  [section 14](14-security.md) (at-rest and backup handling), not on the messaging module.
- **2026-08-15 — A verified email address is the barrier, and unverified means read-only.** Every
  anti-abuse decision in the design leans on it — [4.9](04-editing.md) declined captcha and
  honeypots, [5.7](05-messaging.md) declined everything heavier than a rate limit — so it may not be
  softened later without reopening both. An unverified account reads public pages and does nothing
  else: no edit, no collection, no message. Decided in [6.1](06-accounts.md), enforced in the service
  layer so [10.1](10a-public-api.md) inherits it.
- **2026-08-15 — A user row is never deleted, only emptied.** Deletion anonymises in place and leaves
  a tombstone: [5.1](05-messaging.md) requires exactly two participant rows per conversation and
  [4.2](04-editing.md) hangs every revision off its author, so a missing row would dangle both.
  Message bodies and catalogue edits survive attributed to the tombstone; the user's own collection,
  wantlist and tags are hard-deleted. Decided in [6.5](06-accounts.md), closing what
  [5.8](05-messaging.md) handed on. **This generalises the earlier rule** that nothing a collection
  points at is hard-deleted: in this design, deletion of anything anyone else references is
  anonymisation or a tombstone, never a `DELETE`.
- **2026-08-15 — One text-rendering policy for all user prose.** Message bodies
  ([5.5](05-messaging.md)) and profile bios ([6.3](06-accounts.md)) are stored verbatim, escaped at
  render, newlines preserved, URLs linkified with `rel="noopener nofollow ugc"`. Any later
  free-text-for-humans field takes the same rule rather than inventing a second one — two policies
  means two sanitising stories and two places to get XSS wrong ([14.3](14-security.md)).

## Rejected approaches

What we considered and turned down, and why — so we do not re-litigate it in three months.

- **2026-08-14 — Seeding the catalogue from a Discogs or MusicBrainz dump/API.** Turned down at
  [1.5](01-product.md). It would solve the thin-catalogue problem, but it buys a licensing question,
  an availability and rate-limit dependency, and a sync problem, in exchange for making the editing
  features — the interesting part of a portfolio project — largely redundant.
- **2026-08-14 — Building for future commercialisation.** Considered at [1.1](01-product.md) as
  "portfolio with commercial potential" and set aside. We do not pay the up-front cost of keeping
  that door open; if it is ever re-opened, re-seeding the catalogue under a different data licence is
  accepted as the price.
- **2026-08-14 — A marketplace, in any form or at any stage.** Turned down at [1.7](01-product.md).
- **2026-08-15 — A global `Track` / composition entity.** Turned down at
  [2.1.4](02-catalogue-model.md). It would give "which releases contain this song", but no import
  channel carries a tracklist ([1.5](01-product.md)) so the table would be near-empty
  ([1.4](01-product.md)); doing it properly needs the work/recording distinction, and doing it by
  title produces garbage. Reversible additively — tracklist items keep their titles — so a future
  work entity is a new table plus a linking pass.
- **2026-08-15 — Free-form genre and style tags.** Turned down at [2.3.2](02-catalogue-model.md) in
  favour of a curated vocabulary. `hip hop` / `hip-hop` / `HipHop` fragments within a week and
  filtering is the entire point of the field.
- **2026-08-15 — A table per medium for formats**, and equally a free-text format string. Turned down
  at [2.4.2](02-catalogue-model.md). Typed-to-SQL makes every new descriptor a migration and leaves a
  half-known imported format nowhere to go; free text kills "show me all my cassettes", which
  [1.2](01-product.md) exists to support. The sum types live in the domain layer instead.
- **2026-08-15 — A data-quality field** in the Discogs "Needs Vote / Complete and Correct" mould.
  Turned down at [2.4.7](02-catalogue-model.md): a status column with no voting workflow behind it is
  decoration that goes stale. The quality signal is the edit history
  ([section 4](04-editing.md)), which is data rather than an assertion.
- **2026-08-15 — Collection folders, and custom user-defined fields.** Turned down at
  [3.3](03-collection.md) and [3.4](03-collection.md) in favour of user-owned tags plus a free-text
  note. Folders impose single membership we cannot justify; custom fields are a per-user schema
  (definitions, types, validation, rendering, export columns, API shape) bought for an undemonstrated
  need. Both are what tags already do, and a Discogs folder imports as a tag losslessly.
- **2026-08-15 — A current-valuation field and a total-collection-value statistic.** Turned down at
  [3.2](03-collection.md) and [3.8](03-collection.md) on top of the money deferral: with
  [1.5](01-product.md) forbidding market data, valuation could only be a number typed once and never
  revisited, and a total computed from it is fiction wearing a statistic's clothes.
- **2026-08-15 — Per-item collection privacy.** Turned down at [3.7](03-collection.md). Real for some
  collectors, but it puts a clause in every listing query and a control on every row to hide what a
  private collection already hides. Sensitive fields (note, purchase date and place) are owner-only
  unconditionally instead, which covers the actual worry.
- **2026-08-15 — Voting on edits, MusicBrainz style.** Turned down at [4.3](04-editing.md). Voting
  presumes an electorate; ours has one member ([1.4](01-product.md)), and a vote queue nobody drains
  is worse than no queue at all. Reopen only if there are ever enough simultaneous editors to
  disagree.
- **2026-08-15 — Reputation, contributor ranks and a `trusted` role.** Turned down at
  [4.8](04-editing.md) and [4.7](04-editing.md). Ranks exist to allocate moderation privileges by
  earned standing, which is the mechanism [4.1](04-editing.md) already declined; at our volume the
  leaderboard has one entry. A derived "edits made" count on the profile is kept — it gates nothing.
- **2026-08-15 — Captcha, honeypot, and a prebuilt mass-revert tool.** Turned down for the MVP at
  [4.9](04-editing.md) as cost against an imaginary threat, behind a verified-email wall at
  portfolio scale. Note the asymmetry that decided it: mass revert is *not built* but its data *is
  stored*, because the tool is a day's work later and the history is unrecoverable.
- **2026-08-15 — `Series` as an entity.** Turned down at [2.1.5](02-catalogue-model.md) — real on
  real objects, but a mostly-empty table with a moderation cost at ~12,000 releases, and expressible
  as a format descriptor or a note. Additive if ever reopened.
- **2026-08-15 — Comments on releases, reviews and ratings, following, an activity feed, a forum.**
  All turned down at [5.11](05-messaging.md), and each for its own reason: comments compete with the
  edit path, ratings are noise at ten users and are RateYourMusic's product anyway, a feed needs a
  population and a rate of activity we will not have, and a forum is a second product with the
  largest moderation load available to us.
- **2026-08-15 — Group chat.** Turned down at [5.1](05-messaging.md). Participant management,
  divergent read state and "who may add whom" bought for a population with no groups in it. Additive
  later only because per-user state lives on a `conversation_participant` row for its own reasons.
- **2026-08-15 — Attachments in messages, and markdown or restricted HTML in message bodies.**
  Turned down at [5.4](05-messaging.md) and [5.5](05-messaging.md). Private attachments are the
  unmoderatable worst case of the storage problem [1.4](01-product.md) already flags; rich text
  means owning a sanitiser and a standing XSS liability for a table with dozens of rows. Plain text
  upgrades to markdown cleanly later; the reverse migration does not exist.
- **2026-08-15 — Web push, and read receipts.** Turned down at [5.9](05-messaging.md) and
  [5.6](05-messaging.md). Push is a service worker, VAPID keys, a permission prompt and a
  subscription table aimed at users who are not in the app. Receipts are social pressure and a
  privacy choice made on the user's behalf — but `last_read_at` is stored regardless, so this is
  another case of the [4.9](04-editing.md) asymmetry: store the data, do not build the tool.
- **2026-08-15 — OAuth login (Google/GitHub/VK), at any stage.** Turned down at
  [6.1](06-accounts.md). Every provider is a console registration, a secret in configuration, a
  callback route and an account-linking matrix, and it removes no work: the mail sender, the address
  column and the verification flow are all needed regardless ([6.1](06-accounts.md)'s barrier). It
  only adds a second way in. Additive later as a provider table beside the password column.
- **2026-08-15 — 2FA in the MVP.** Deferred at [6.2](06-accounts.md), not refused. With nothing
  money-shaped anywhere ([1.7](01-product.md), [3.2](03-collection.md)), the worst case of a stolen
  account is a reversible edit ([4.4](04-editing.md)) and a read mailbox — against secret storage,
  recovery codes and a lockout path with no support channel. If reopened it is TOTP for **everyone**,
  never an admin-only privilege: a second factor with one user is a code path with no coverage.
- **2026-08-15 — A key/value settings table, and every setting not already decided elsewhere.**
  Turned down at [6.6](06-accounts.md). The complete list is three columns on `user`
  (`collection_visibility`, `wantlist_visibility`, `notify_new_message`); a key/value table is a
  schema you cannot query, constrain or index. Explicitly not settings: transactional mail, profile
  visibility, per-conversation muting or digests ([5.9](05-messaging.md)), per-item collection
  privacy ([3.7](03-collection.md)), and "who may message me" — [5.7](05-messaging.md)'s blocking
  covers that reactively.
- **2026-08-15 — A profile links field, and profile fields we have no use for.** Turned down at
  [6.3](06-accounts.md). A link list is a table, a rendering, a validation and an SEO-spam target for
  a page nobody reads; URLs go in the bio and are linkified there. No real name, birthday, gender or
  phone either — data we do not hold is data [13.3](13-legal.md) need not account for and
  [14.6](14-security.md) need not protect.
- **2026-08-15 — An undo window on account deletion.** Turned down at [6.5](06-accounts.md).
  It needs a scheduled purge job and a fourth account state to explain, on top of the three
  ([6.5](06-accounts.md)) already has. Deletion is immediate and irreversible, with an export offered
  on the same page ([10.3](10c-export.md)) so that leaving does not mean losing.

## Constraints discovered

Facts found while designing that constrain later choices (external ToS, format quirks, volumes).

- **2026-08-14 — A Discogs CSV row is richer than the agenda assumed.** Verified on a real export:
  rows carry artist, title, label, catno, format and release_id — enough to build a recognisable
  stub, not an empty shell. `recommendations.md` point 8 overstates the problem and should not be
  used as an argument for needing an external metadata source. Details in
  [10.2 working notes](10b-import.md).
- **2026-08-14 — Discogs packs multiple values into single CSV fields, comma-joined and
  positionally paired** (`label` against `catno`), while label names may themselves contain commas.
  Any CSV parsing design that splits on `,` is wrong. Details in [10.2 working notes](10b-import.md).
- **2026-08-14 — EU database rights cover repeated extraction of insubstantial parts.** A single
  user's collection export is not a substantial part of anyone's database, so one upload is safe. But
  the Database Directive also bars *repeated and systematic* extraction of insubstantial parts, and
  accreting many users' exports into one public catalogue is arguably that. Accepted as a residual
  risk at portfolio scale ([1.5](01-product.md)); it would need real legal review before any public
  launch at volume or any commercial turn. Not verified against current case law.

## Open questions blocking other items

Questions that must be answered before some other item can be closed. Format: what is blocked ← what we are waiting on.

- ~~[1.9](01-product.md) MVP scope and [1.10](01-product.md) success criterion~~ — **closed
  2026-08-15**, immediately after section 2 removed what they were waiting on. **Section 1 is now
  fully closed.** The MVP is the collector's round trip (upload, browse, correct, image, search,
  export) with messaging, the public API and any moderation queue explicitly out; success is the
  round trip working on a real export plus a second person using it unaided.
  **What this now constrains:** [section 15](15-roadmap.md) inherits a scope line and two sequencing
  rules (export before import; master merge early, not late), and every remaining P0 section should
  be read against the deferred list rather than designed in full.
- ~~Multi-script artist and title names (`Кино` / `Kino`)~~ — **closed 2026-08-15** at
  [2.2.1](02-catalogue-model.md): one artist, many name rows, one search index, no transliteration
  anywhere. Kept here only as a pointer, since [1.8](01-product.md) and several notes refer to it as
  an open question. It is now an invariant above.
- ~~[11.2](11-stack.md) final stack confirmation ← section 5~~ — **unblocked 2026-08-15.**
  Sections 1–5 are now all closed and **nothing is waiting on a model section any more**;
  [11.2](11-stack.md) can be decided whenever section 11 is taken up. What the model sections handed
  it: section 2 — four entities, no global track table, vocabularies as seeded lookups, an ordered
  artist credit in three parallel tables; sections 3 and 4 — no denormalisation, aggregate-snapshot
  history, role-based rights in a service layer; section 5 — **nothing new at all** (no real-time
  transport, no file storage, no search backend, no push service; five ordinary tables and one email
  template). [1.12](01-product.md) records the preference (Haskell) and the fallback
  (Elixir/Phoenix), and the argument for Elixir that the messaging module might have supplied — a
  real-time vertical — **did not materialise**, since [5.3](05-messaging.md) refused real time
  outright. The stack may be chosen on the catalogue's merits alone.
- ~~[6.5](06-accounts.md) account deletion ← [5.8](05-messaging.md)'s tombstone constraint~~ —
  **closed 2026-08-15.** The account is anonymised in place and the row is never deleted; messages,
  conversations and catalogue edits survive attributed to the tombstone, collection and wantlist are
  hard-deleted. **Section 6 is now fully closed.** What it hands on: [13.3](13-legal.md) inherits
  three statements it may not overstate (what deletion erases and keeps, that message bodies are
  plaintext and survive the sender, that the only personal data held is an address, a nickname and a
  bio); [section 14](14-security.md) inherits password storage ([14.2](14-security.md)), the rate
  limits on registration, login, reset and resend ([14.5](14-security.md)) and log retention
  ([14.6](14-security.md)); [8.2](08-media.md) gains a second consumer in the avatar;
  [section 12](12-infrastructure.md) inherits transactional-mail deliverability for four templates.
- ~~Soft delete and merge semantics ← [section 4](04-editing.md)~~ — **closed 2026-08-15.** It is
  `merged_into` *and* `deleted_at`, as two distinct operations with different meanings, and neither
  is a consequence of edit versioning ([4.4](04-editing.md), [4.5](04-editing.md)). Now an invariant
  above. **What it hands on:** [3.11](03-collection.md) inherits three cases to answer — edited
  (nothing happens), merged (the item follows the pointer), deleted (the item survives and is shown
  as deleted).
- Seeding the vocabularies ← [section 15](15-roadmap.md). Genres, styles, credit roles, format
  descriptors and country codes (ISO plus historical `SU`, `YU`, `DD`, `CS`) all need initial
  contents, and [1.5](01-product.md) forbids fetching them, so they are hand-written seed data — our
  own list, not an extraction from anyone's database. Nobody has estimated the work; it is a real
  roadmap task, not a footnote.

## Needs verification against reality

Claims in the agenda that are assumptions until checked against a real file, API or document.

- [10.2.2](10b-import.md) — **partly done 2026-08-14.** A real Discogs *inventory* export was
  inspected (see that file's working notes). Discogs' CSV conventions are now known; the
  *collection* export's own column set is still unverified and needs a real collection export.
- [10.2.7](10b-import.md) — still open. The inventory export carries `listing_id`; whether the
  collection export carries `Collection Item Instance ID` is unconfirmed.
- [10.5.1](10e-legal-sources.md) — narrowed by [1.5](01-product.md): we never call the Discogs API,
  so its rate limits are moot. What remains to check is whether Discogs' ToS says anything about
  what a user may do with their **own** export.

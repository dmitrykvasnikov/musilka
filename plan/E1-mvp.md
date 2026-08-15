# E1 — The MVP

**Size:** 4–5 months ([15.1](../design/15-roadmap.md)) · **Tasks:** T-34 … T-160

**What it is:** [1.9](../design/01-product.md)'s collector's round trip — upload, browse, correct,
image, search, export. **Exit is [1.10](../design/01-product.md)'s criterion 1, not a feature
list:** a real Discogs export imports, nothing is orphaned, twenty releases get corrected by hand
with the history intact, the whole collection exports, and **re-importing that export lands in the
same state with no duplicates**.
~~and a second person uses it unaided~~ — **withdrawn 2026-08-15** by
[1.10](../design/01-product.md)'s amendment. The scope of E1 is unchanged; what changed is that a
public deployment is deferred and non-gating, which is why nine tasks below moved rather than
disappeared.

**The nine slices are in the order [15.1](../design/15-roadmap.md) argued for, and the order is
itself a decision.** Accounts first because everything else is user-scoped and
[6.1](../design/06-accounts.md)'s barrier is load-bearing for four other decisions. Catalogue before
collection because `collection_item.release_id` is `NOT NULL` — there is nothing to collect until
releases exist. **Merge before export and import**, which is
[1.9](../design/01-product.md)'s "early, not late" made concrete: the importer mints a master per
row, so the first real import lands in a system that must already be able to repair itself. Export
before import because since [10.2.1](../design/10b-import.md) the export file **is** the importer's
input format. The importer before the converters, so the job, the ladder, the report and rollback
are provably correct against a format we control before a third party's comma-packed,
sentinel-bearing file is pointed at them. Search after converters because tuning a search over an
empty catalogue is guesswork. **Images last, deliberately** — they are the one irreversible decision
in the design and [1.4](../design/01-product.md)'s predicted first cost overrun, so they are the
thing least suited to being half-built while everything else moves.

**The surface area is [7.5](../design/07-search-ux.md)'s 24 screens**, roughly 30 templates.
Anything not on that list is not in the MVP.

**Four things ship switched off**, already in the schema and named in
[15.2](../design/15-roadmap.md): artist photos, the companies UI, track-level credits, and
membership editing. Each is E3. Where a task creates the column or table for one of them, it says
so — writing it now is the cheapest possible instance of "retrofitting is the expensive rework".

---

## E1.1 Accounts

[Section 6](../design/06-accounts.md), plus [14.2](../design/14-security.md)'s password answer and
[14.5](../design/14-security.md)'s limits. This slice ends with a stranger able to register, verify,
log in, recover, change their address, edit a profile and delete the account — and with the
verified-email barrier enforced in the service layer, where the rest of the design assumes it is.

- [ ] T-34 Migration 002: accounts (per 6.3, 6.4, 11.6, 14.2, 14.5)
      `user(id, email, password_hash, email_verified_at, role, nickname, avatar_id, bio, location,
      created_at, collection_visibility, wantlist_visibility, notify_new_message, deleted_at,
      disabled_at)`; `user_token(user_id, kind, token_hash, expires_at, consumed_at)` — one table
      for verification, reset and pending email change rather than three near-identical ones;
      `nickname_reservation(nickname, user_id, released_at)`; `session`; `rate_limit`.
      Unique index on `lower(nickname)`. `avatar_id` is nullable and points at an `image` row that
      does not exist until E1.9 — add the FK there, not here. No settings table, no OAuth provider
      table, no 2FA secret ([6.6](../design/06-accounts.md), [6.1](../design/06-accounts.md),
      [6.2](../design/06-accounts.md)).
- [ ] T-35 Password hashing and the common-password list (per 14.2)
      Argon2id via `crypton`/libsodium, tuned so one hash takes 100–250 ms on the target box,
      starting at 64 MiB / t=3 / p=1 and dropping the *memory* rather than the algorithm if the box
      complains. The encoded string carries its parameters, so a login re-hashes when they change.
      No pepper.
      Policy is **minimum 10 characters and nothing else** — no composition rules, no rotation, no
      maximum below 128, every Unicode character accepted including spaces — plus a **vendored list
      of the ~10,000 most common passwords**, checked at registration and at password change. One
      file, one lookup, no service call.
- [ ] T-36 Session store, cookie and CSRF token (per 11.6, 14.2, 14.3, 13.4)
      32 bytes from the OS CSPRNG; **the table stores a SHA-256 of the token, not the token**, so a
      leaked backup does not hand over live sessions. Cookie `HttpOnly`, `Secure`, `SameSite=Lax`,
      30 days idle expiry — and it is the **only** cookie in the product, which is what removes the
      banner. [14.3](../design/14-security.md)'s CSRF token lives on the session row rather than in
      a second cookie; wire it to T-12's form helper.
      The session *store* is a service like any other; the cookie and middleware are `musilka-web`'s.
- [ ] T-37 Registration (per 6.1, 14.2, 14.5)
      Open registration, email and password. **Registration with an address that already exists
      returns the same "check your email" page as a new one**, and mails the existing account a
      "someone tried to register with your address" note instead of a verification link. 3/hour and
      10/day per IP.
- [ ] T-38 Outbound mail port, job and templates (per 6.1, 10.4.5, 12.5)
      The queue's second consumer and the first one to exist. A port with an SMTP/API implementation
      and a no-op that writes to the log locally; jobs enqueued by the service layer, never sent
      inline. Four templates and no more: verify, reset, email-change warning, and (in E2)
      new-message notification. **No import-completion email** — [10.2.8](../design/10b-import.md)
      refused a fifth template for a job that takes a minute.
      Unblocks E0's T-31.
- [ ] T-39 Email verification (per 6.1, 14.2, 14.5)
      Token 32 random bytes, single-use, stored hashed, **24 hours**. Resend rate-limited 3/hour per
      account and IP.
      **This is the barrier, not a courtesy.** An unverified account reads public pages and does
      nothing else — no edit, no collection, no message. T-50 is where that is actually enforced.
- [ ] T-40 Login, logout and brute-force backoff (per 14.2, 14.5)
      **Login never distinguishes an unknown address from a wrong password**, and runs a dummy
      Argon2 verification when the user does not exist so the timing does not either.
      Failed logins increment a counter on the account; the response is delayed and then refused
      with an exponential window from the 5th failure in 15 minutes, and the counter resets on
      success. A per-IP cap of 20 per 15 minutes runs alongside for the distributed case.
      **Backoff, never lockout** — a lockout is an account-denial tool handed to the attacker, and
      there is no support channel to undo it. No email alert on failed logins.
- [ ] T-41 Password reset (per 6.2, 14.2, 14.5)
      Single-use hashed token, **1 hour**, 3/hour per account and IP. The response is identical
      whether or not the address is registered. **A successful reset invalidates every session** —
      this is the requirement that excluded JWT at [11.6](../design/11-stack.md), so it must
      actually hold.
      Recovery is not optional: without it a forgotten password permanently orphans a collection,
      which contradicts the trust argument [3.9](../design/03-collection.md) makes for export.
- [ ] T-42 Email change (per 6.2)
      A `pending_email` plus a token; **the new address is confirmed before it takes effect**, and a
      warning goes to the old one. Every session is invalidated on change. The attack this closes is
      cheap and real: a borrowed session that could rewrite the address owns the account, because
      the address is what recovery trusts.
- [ ] T-43 Nicknames (per 6.4)
      `[A-Za-z0-9_-]`, 3–30 characters, case-insensitively unique, display preserving the typed
      case. Deliberately ASCII — **this is not a breach of [2.2.1](../design/02-catalogue-model.md)'s
      no-transliteration invariant**, which governs catalogue data, names belonging to an object we
      are describing; a nickname is an identifier the user invents for our URLs.
      Renaming at most once per 30 days; **the old nickname is held by that user permanently** in
      the reservation table and never returned to the pool. Links to `/u/<old>` 404 rather than
      redirecting, and that is accepted. Seed the short impersonation list: `admin`, `moderator`,
      `support`, `staff`, `system`, `musilka`.
- [ ] T-44 The linkifier (per 14.3, 5.5, 6.3)
      **One text-rendering policy for all user prose, and this is the function that is it.** Stored
      verbatim, **escaped first, then linkified over the escaped text**, newlines preserved,
      `rel="noopener nofollow ugc"`, and only `http` and `https` schemes — no `javascript:`, no
      `data:`. Never resolve a URL, ever: [14.3](../design/14-security.md) forbids link previews by
      name.
      It is the highest-risk function in the codebase and it exists exactly once, which is why
      NOTES insisted on one policy rather than two. **Property tests are owed to
      [11.10](../design/11-stack.md)'s list** and belong in this task, not later.
- [ ] T-45 Profile page (per 6.3, 4.8)
      `/u/<nickname>`, readable without logging in. Nickname, avatar (E1.9), bio (plain text, 500
      characters, rendered by T-44), location (free text, not a vocabulary), member-since, and an
      **edit count computed live** from `revision` — no counter column to drift, and it gates
      nothing. Collection and wantlist links appear only when [3.7](../design/03-collection.md) says
      `public`; **a `link` list is never linked from the profile**, since publishing the unguessable
      URL is the one thing that would make it meaningless.
      No links field, no real name, birthday, gender or phone.
- [ ] T-46 Settings (per 6.6, 6.2)
      `/settings`: the three columns — `collection_visibility` (default **private**),
      `wantlist_visibility` (default **public**), `notify_new_message` (default on) — plus email
      change and password change. That is the complete list; a fourth setting is a reopening of
      [6.6](../design/06-accounts.md), not an addition.
- [ ] T-47 Account deletion by anonymisation (per 6.5, 13.3)
      **The row is never deleted.** Cleared: email, `password_hash`, bio, location, avatar (file
      deleted per E1.9), notification setting; nickname replaced by `deleted-<id>`, which T-43's
      reservation table makes unclaimable. Hard-deleted: collection items and their tags, wantlist
      items, the user's own tag definitions, blocks they created. Kept and attributed to the
      tombstone: catalogue edits and revisions, and (from E2) conversations and message bodies.
      The page offers T-99's export first, then requires typing the nickname and the password.
      Immediate and irreversible, said plainly — [6.5](../design/06-accounts.md) refused an undo
      window. Login is refused afterwards; re-registering with the same address creates a **new**
      account with no link to the old one.
- [ ] T-48 `disabled_at` (per 6.5, 5.7)
      One column, no UI: login refused, content untouched, reversible, and the admin runs an
      `UPDATE`. Distinct from deletion in both meaning and effect. It exists so that
      [5.7](../design/05-messaging.md)'s report email has an action behind it.
- [ ] T-49 Rate limiting (per 14.5, 10.4.5, 9.4)
      Fixed-window counters in `rate_limit`, keyed by bucket, subject and window start, **checked
      and incremented in the service layer** — which is what makes [9.4](../design/09-nfr.md)'s
      promise true that a future API would inherit the numbers instead of reinventing them. There is
      no second rate-limiting design anywhere; [14.5](../design/14-security.md)'s table is the whole
      list, and later slices add their rows to it rather than inventing numbers.
      Exceeding a limit returns **429 with a page saying when to retry** — never a silent drop,
      never a shadowban. Moderators and admins are exempt from the content limits and **never** from
      the authentication limits. A daily systemd timer deletes expired windows (not a queue job —
      [10.4.5](../design/10d-model-requirements.md) is explicit that recurring work is a timer).
- [ ] T-50 The verified-email barrier in the service layer (per 6.1, 10.4.4, 4.1)
      One check, in `musilka-app`, that every mutating service function passes through. It is where
      [4.9](../design/04-editing.md)'s and [5.7](../design/05-messaging.md)'s refusal of captcha
      actually lands, so it is worth a dedicated integration test per operation
      ([11.10](../design/11-stack.md) level 2) rather than a comment.
- [ ] T-51 `/status` page (per 9.5)
      Authenticated. Database reachable, applied migration version, queue depth and the age of the
      oldest pending job, failed job count, timestamp of the last successful backup. This is what
      replaces a metrics stack, and it is on [9.3](../design/09-nfr.md)'s short list of tasks that
      are easy to leave undone forever.

---

## E1.2 Catalogue

[Section 2](../design/02-catalogue-model.md) and [section 4](../design/04-editing.md)'s editing
half, plus the hooks [10.4](../design/10d-model-requirements.md) requires. The largest slice by
some distance, and the one holding [15.4](../design/15-roadmap.md)'s unestimated risk: **the
vocabulary seed data**.

**Everything in this slice obeys the curated-set invariant:** every field drawn from a vocabulary is
seeded and closed, with a free-text companion beside it rather than inside it. The field exists to
group or filter, free text destroys grouping, and a closed set with no escape hatch makes users lie.

- [ ] T-52 Migration 003: artist (per 2.2.1, 2.2.2, 2.2.3, 2.2.5, 2.2.6)
      `artist(id, artist_type nullable, disambiguation nullable, country nullable, begin_date,
      end_date, profile, is_special)`; `artist_name(artist_id, kind ∈ primary|variant|real_name,
      name)` with exactly one `primary`; `artist_relation(from, to, type ∈ member_of|alias_of,
      begin_date, end_date)`; `artist_link(artist_id, position, url, label)`.
      **Nothing is required beyond one `primary` name row.** `artist_type` nullable is the whole
      point — an imported row gives a name and nothing else, and a required type would mean
      guessing. The relation table ships now and its **editing UI is E3**.
- [ ] T-53 Migration 004: master and artist credits (per 2.3.1, 2.3.3, 2.3.4)
      `master(id, title, first_release_year nullable, primary_type nullable, key_release_id
      nullable)` plus secondary types as a set and `master_tag` for genres and styles.
      **Three credit tables, not one polymorphic one**: `master_artist_credit`,
      `release_artist_credit`, `track_artist_credit`, identical in shape —
      `(position, artist_id, credited_as nullable, join_phrase nullable)`. Postgres gets real
      foreign keys; Haskell gets one shared type and one renderer.
      `credited_as` is normally `NULL`, meaning "use the artist's current primary name" — copying
      the name in would freeze it. `join_phrase` is **free text and a deliberate exception** to the
      curated-set invariant, because nothing groups by it.
- [ ] T-54 Migration 005: release and its satellites (per 2.4.1 – 2.4.8)
      `release(id, master_id NOT NULL, title, date + precision + date_is_approximate, country
      nullable, is_unofficial, notes)`; `release_format(position, medium, qty, descriptors)`;
      `release_identifier(position, type, value, description)` with a **normalised digits-only
      column beside `barcode`**; `release_track(sequence, position, title, duration_s, parent_id,
      kind)` with nesting **one level deep, enforced**; `release_credit` with a **nullable
      `track_id` from day one**; `release_image(position, type, is_primary)` with a cap of 8.
      Mandatory: `title`, `master_id`, artist credit. Everything else optional — a release created
      from a CSV row must be able to exist.
      **No uniqueness constraint anywhere on the release, at any stage.** Not on
      `(artist, title, catno)`, not on barcode. Two pressings really can differ only in a runout
      etching, and a constraint would make the importer silently discard a user's row.
      Track-level credits and hand-typed tracklists are **E3**; the tables ship here.
- [ ] T-55 Migration 006: label and companies (per 2.1.3, 2.4.6)
      `label` as a top-level entity with its own page, and
      `release_company(release_id, position, label_id, role, catno)` — the `role` column is what
      lets one entity serve labels, pressing plants, studios and copyright holders. **The MVP shows
      and edits only `label`**; the other roles cost nothing now and turning them on later is a UI
      change with no migration ([15.2](../design/15-roadmap.md)'s companies UI, E3).
      A release has an *ordered list of (label, catno) pairs*, which is a join table no matter what
      — the Discogs positional pairing finding makes that non-negotiable.
- [ ] T-56 `revision` table (per 4.2, 10.4.2, 10.4.7)
      `revision(id bigint identity, entity_type, entity_id, editor_id, created_at, note, snapshot
      jsonb, source ∈ ui|import|merge|moderation, import_id nullable)`.
      **The versioning unit is the aggregate, not the row** — a release snapshot carries its
      formats, identifiers, label pairs, credits and tracklist. `revision.id` must be monotonic and
      never reused ([10.4.7](../design/10d-model-requirements.md)); no `changed_fields` column, no
      outbox, no per-entity feed.
      Written from day one and **shown to nobody**: the diff viewer, revert flow and change feed are
      E3. Snapshots nobody reads cost one INSERT per mutation; snapshots never written cost history
      that cannot be reconstructed.
- [ ] T-57 The revision write inside every mutation (per 4.2, 10.4.4, 10.4.7)
      Not a hook a caller may forget — it lives inside the service function, alongside the
      authorisation clause, the rate-limit check and the verified-email barrier. **Including merge
      and delete** (T-91, T-93), the two that look like exceptions: a consumer replaying the log
      would otherwise never learn that an entity had disappeared or been absorbed.
      Owed to [11.10](../design/11-stack.md)'s list: a property test that an aggregate snapshot
      reconstructs its entity.
- [ ] T-58 `external_id` table (per 10.4.1, 2.5.4)
      `(id, entity_type, entity_id, source, external_id, added_by, added_at)`, **unique on
      `(entity_type, source, external_id)`**, index on `(entity_type, entity_id)`. No `url` — it is
      a pure function of the other three and **a stored URL is an invitation for some later code
      path to fetch it**. No `synced_at` — there is no sync and there never will be.
      No foreign key, because `(entity_type, entity_id)` is polymorphic; referential integrity is
      the service layer's job, and the concrete obligation lands on merge (T-91).
      One source in the MVP, `discogs`, written by E1.7's converter. **This is the only place a
      third party's identifier lives**, which is what lets the importer stay format-agnostic.
- [ ] T-59 Partial dates (per 2.4.1, 2.2.2)
      One nullable date plus a precision enum (year / year-month / full) plus
      `date_is_approximate` for `ca. 1974` on the sleeve — **not three columns and not a string**,
      so ordering and range queries still work. Used by releases, artist years-active and artist
      relations.
- [ ] T-60 Format sum types in the domain (per 2.4.2, 1.12)
      The storage stays uniform `(position, medium, qty, descriptors[])`; `musilka-domain` parses
      each row into a per-medium sum type (`Vinyl VinylSpec | Cd CdSpec | …`). The types live in
      Haskell, where they are cheap; rigidity in the schema is expensive. Descriptor validity is
      enforced by a validation table, not by the schema — `45 RPM` and `12"` cannot land on a CD.
      **No format-specific field is ever promoted onto the release itself** — `size` and `speed` are
      vinyl descriptors.
- [ ] T-61 Size the vocabulary seed work (per 15.4 risk 4, NOTES)
      **Do this before T-62 … T-66, not after.** Nobody has estimated it and
      [NOTES.md](../design/NOTES.md) has carried it as a real roadmap task since 2026-08-14. Count
      the entries each list actually needs, decide how small "deliberately small" is, and write the
      number down here. The mitigation is that [section 4](../design/04-editing.md)'s moderation
      path exists precisely so users can request what a seeded list is missing.
      [1.5](../design/01-product.md) forbids fetching any of it: these are **our own lists**, not an
      extraction from anyone's database.
- [ ] T-62 Seed: country codes (per 2.4.1, 2.2.6)
      ISO 3166-1 alpha-2 **plus historical entries** — `SU`, `YU`, `DD`, `CS` and similar — and
      codes are never retired. A Melodiya pressing is from `SU`, and collectors of physical media
      will not accept `RU`. One table, two uses (release country and artist country).
- [ ] T-63 Seed: genres and styles (per 2.3.2)
      Two kinds in one `tag` table: ~20 genres and a few hundred styles. **Flat, not hierarchical**
      — styles are not cleanly nested under genres in reality. They live on the **master**, not the
      release: a 2015 repress of a jazz record is still jazz. Adding a value is a moderation action
      (E3); users never mint one inline.
- [ ] T-64 Seed: format media and descriptors (per 2.4.2)
      Media as a closed enum — `vinyl`, `cd`, `cassette`, `reel`, `minidisc`, `shellac`, `file`,
      `other` — plus a descriptor vocabulary **scoped per medium** via the validity table from T-60.
      Expect it to stay open-ended: `FYE` in the observed Discogs file is a retailer's abbreviation
      and will never be in a sensible vocabulary, which is what the free-text companion field is for.
- [ ] T-65 Seed: credit roles, company roles, identifier types (per 2.4.5, 2.4.6, 2.4.3)
      Credit roles (`Producer`, `Mixed By`, `Guitar`, `Sleeve Design`, …) with a free-text detail
      field rendering in brackets — `Guitar [Slide]` without letting the role itself fragment.
      Company roles: `label`, `pressed_by`, `distributed_by`, `phonographic_copyright`, `copyright`,
      `made_by`, `recorded_at`, `mastered_at`. Identifier types: `barcode`, `matrix_runout`,
      `mastering_sid`, `mould_sid`, `label_code`, `rights_society`, `asin`, `other`.
- [ ] T-66 Seed: conditions and the `Various` singleton (per 3.2, 2.2.4)
      Goldmine grading with a **`rank` column**, so "everything VG+ or better" is a comparison
      rather than a list of strings: `M, NM, VG+, VG, G+, G, F, P`, with sleeve carrying three more
      — `Generic`, `No Cover`, `Not Graded`.
      Plus exactly one reserved artist row, `Various`, `is_special = true`: not editable, not
      mergeable, not deletable, and credited **exactly like any other artist** so that no read path
      anywhere gets a branch.
- [ ] T-67 Artist page (per 7.5, 2.2.x)
      `/artist/<id>/<slug>`: names (all kinds), disambiguation rendered beside the name and never
      concatenated into it, discography, links, and members read-only. Editing membership is E3.
- [ ] T-68 Master page (per 7.5, 2.3.4)
      `/master/<id>/<slug>`: credit, genres and styles, its releases. The tracklist shown is
      `key_release_id`'s, and where that is unset **the master page shows no tracklist at all**,
      which is honest — nothing is derived, nothing is cached, nothing goes stale.
      **Nothing in the UI may treat a one-child master as suspect**: after an import, duplicate
      masters are the normal state, not an error state.
- [ ] T-69 Release page (per 7.5, 2.4.x)
      `/release/<id>/<slug>`, growing T-16's page into the real thing: format entries, label/catno
      pairs, identifiers, credits, tracklist, notes, `is_unofficial`, and the add-to-collection
      control (E1.3). Images arrive in E1.9.
- [ ] T-70 Label page (per 7.5, 2.1.3)
      `/label/<id>/<slug>`: releases on this label, via `release_company` filtered to `role = label`.
- [ ] T-71 Entity create and edit forms (per 7.5, 4.1, 12.7)
      One template shape, four entities, built through T-12's form helper. Wiki style: **any user
      with a verified email edits immediately** — no pre-moderation, no reputation threshold, no
      trusted tier. Errors are associated with their field and repeated in a summary at the top,
      because a failed submission is a full page render ([7.7](../design/07-search-ux.md)).
      This is several sittings; split by entity when it comes to it rather than pretending it is one.
- [ ] T-72 Trigram indexes and the artist/label picker (per 7.4, 7.2)
      **Search ships here, not in E1.8, and the reason is a dependency rather than a preference:**
      every credit in the catalogue must point at an artist entity, so the picker on an edit form is
      on the critical path — without it, crediting an artist means leaving the form to find an ID.
      `gin_trgm_ops` on `artist_name.name` and `label.name`; an HTMX fragment with
      `hx-trigger="keyup changed delay:250ms"` returning an `<li>` list. **It degrades to a plain
      text field and a button that submits** — the suggestion list is an enhancement, never the
      mechanism.
      Creating an artist from inside the picker is part of this task: the friction is real and it is
      one click, and [2.2.6](../design/02-catalogue-model.md) requires nothing of an artist but a
      name.
- [ ] T-73 Advisory duplicate check at create (per 2.4.9, 4.4)
      On save, surface likely matches: normalised barcode, then trigram similarity on title plus
      catno. **It advises; it never blocks**, and normalisation is case and punctuation only — it
      never transliterates. The user picks an existing entity or confirms theirs is new.
      Note for E1.6: the importer deliberately does **not** run this per row.
- [ ] T-74 Roles and the permission check (per 4.7, 10.4.4)
      Three roles — `user`, `moderator`, `admin` — enforced in the service layer, never in the UI
      alone. In the MVP every account is a `user` and the maintainer is `admin`; the two upper tiers
      are schema and a permission check, not a feature to build. **Nothing anywhere is gated on a
      contribution count or a reputation score.**
- [ ] T-75 Catalogue edit rate limits (per 14.5, 4.9)
      Rows in T-49's table: 100 edits/hour and 50 new entities/hour per user. Generous enough that a
      genuine cataloguing session never notices them — they exist to bound damage, not to pace
      contributors.
- [ ] T-76 `revision.source` and `import_id` wired through (per 10.4.2)
      Every mutation records which of `ui | import | merge | moderation` it was, and imports record
      which one. **This is the whole of our provenance story** — there is no field-level provenance
      anywhere, and "is this an untouched import stub?" is *exactly one revision with
      `source = import`*, derived rather than stored.
      The model rule that travels with it: **nothing in the system ever replaces a non-empty field
      automatically.**
- [ ] T-77 Genre, style and vocabulary request path (per 2.3.2, 4.6)
      A user asks for a missing vocabulary value; in the MVP that is a `report` row
      ([4.6](../design/04-editing.md)'s table, created in T-78) plus an email to the admin, and a
      moderator seeds it. The queue UI is E3 — the point here is only that users have **some** lever
      other than lying in a free-text field.
- [ ] T-78 `report` table (per 4.6, 8.4)
      `report(id, target_type, target_id, reporter_id, reason, note, status ∈ open|closed,
      resolved_by, resolved_at)`. `reason` is curated with a free-text `note` beside it. Shipped
      switched off: **the queue and the triage UI are E3**, and deciding the table now costs nothing
      and stops it being designed under pressure. E1.9's image takedown reuses it as another target
      type. 20 reports/day per user ([14.5](../design/14-security.md)).

---

## E1.3 Collection and wantlist

[Section 3](../design/03-collection.md). The slice where the product becomes a product.
**The item holds no copy of catalogue data** — no denormalised title, artist, year or format. That
one rule is what makes a catalogue edit a non-event (T-96), what lets statistics read through to the
master, and what any "let's cache the title on the item" proposal is a proposal to undo.

- [ ] T-79 Migration 007: collection and wantlist (per 3.1, 3.2, 3.3, 3.5, 10.4.3)
      `collection_item(id, user_id, release_id NOT NULL, added_at, media_condition_id,
      sleeve_condition_id, purchased_on, purchased_at, note, rating)` — **no unique constraint on
      `(user, release)`**, because owning two copies is ordinary in this hobby and the fields differ
      per copy.
      `wantlist_item(id, user_id, release_id NOT NULL, priority, note, added_at)` as a **separate
      table**, not a flag — a flag would corrupt every count and every export, and each would then
      need a "but not the wanted ones" clause forever.
      `collection_tag` owned by the user and freely named, plus `collection_item_tag`.
      **Nothing money-shaped**: no purchase price, no currency, no valuation, no maximum price.
      Deferred, not refused — reopening is `price_amount`/`price_currency` on `collection_item` plus
      `max_price` on `wantlist_item`, and nothing else in the design moves.
- [ ] T-80 Add to collection, add to wantlist (per 3.1, 7.5)
      From the release page, one control each. `release_id` is `NOT NULL` everywhere, and there is
      no unresolved state to render.
- [ ] T-81 The filtered-list component (per 7.3)
      **Built once here and reused in four places**: the whole catalogue (`/releases`, E1.8), an
      artist's releases, a label's releases, and a user's collection or wantlist.
      Facets: genre and style (through `release.master_id`), **decade** rather than year (60 buckets
      of three items is not a filter), country including `SU`/`DD`/`YU` and never remapped, format
      medium and descriptor, label, and tags on collection views only.
      **Multi-select within a facet is `OR`, across facets is `AND`**, and that is the only combining
      rule — no negation, no nesting, no saved filters. Counts are computed live on every request
      with no cache and no counter columns.
      **Filters are `GET` parameters and every filtered view has a real URL.** HTMX swaps the list in
      place when available; the plain form submission is what actually happens. Echoed query
      parameters go through the same escaping as any other user text.
- [ ] T-82 Collection listing (per 7.5, 3.1)
      `/collection` on T-81's component. Watch the N+1: [9.1](../design/09-nfr.md) names this as one
      of the three heaviest queries and says the expected failure mode is a list page fetching
      per-row data, not row count. One query per list page.
- [ ] T-83 Wantlist listing (per 7.5, 3.5)
      `/wantlist`, same component, with `priority` (low / normal / high — a small ordered vocabulary,
      not a free integer) in the sort options.
- [ ] T-84 Collection item form (per 3.2, 7.5)
      `/collection/<id>/edit`: both conditions from T-66's ranked vocabulary, purchase date, purchase
      place (free text — it is a memory, not a field to group by), note, rating 1–5, tags.
      **`note`, `purchased_on` and `purchased_at` are owner-only unconditionally**, with no setting
      to change it — that is what per-item privacy was rejected in favour of.
      Condition grades must **read as text, not as a coloured dot** ([7.7](../design/07-search-ux.md)).
- [ ] T-85 Tags (per 3.3)
      Add and remove inline, HTMX-enhanced with a working non-JS path. Tags are **personal labels
      nobody else groups by**, so they are freely named — outside the curated-set invariant rather
      than an exception to it.
      They are load-bearing: having refused folders, custom fields, for-sale status and per-item
      privacy, tags are now the answer to four separate user needs. If filtering them feels awkward,
      all four refusals get worse at once.
- [ ] T-86 Collection statistics (per 3.8)
      Total items and distinct releases; distinct artists; breakdown by format descriptor, genre,
      decade and country. **Computed live on every request, no materialised counters, no cache** — a
      collection is thousands of rows and a `GROUP BY` answers this in milliseconds.
      No total value: [3.2](../design/03-collection.md) stores no prices, so the tile has nothing
      behind it, and its absence is honest rather than a gap.
      The breakdown tiles are clickable, because a breakdown and a facet are the same query.
- [ ] T-87 Visibility and the share link (per 3.7, 14.3, 7.8)
      The two settings from T-46 enforced **in the service layer as a clause in the query**, never in
      a template — a visibility rule that lives in a view is a rule an API does not have.
      `link` is an **unguessable random capability**, not an identifier and not an ACL: the list is
      served to anyone holding it and we say so rather than pretending it is security.
      `Referrer-Policy: same-origin` (T-13) is what keeps it out of other sites' logs; T-14 already
      keeps it out of ours.
- [ ] T-88 Public collection and wantlist pages (per 7.5, 3.7, 7.8)
      `/u/<nickname>/collection` and `/wantlist`. `public` is indexable; **`link` gets `noindex` and
      `X-Robots-Tag: noindex` and never appears in the sitemap** — an unguessable URL in a search
      index is a guessable one; `private` returns 404 and needs no meta tag.
      Owner-only fields never appear here, whatever the setting.
- [ ] T-89 Bulk operations (per 3.10)
      Exactly three: tag a selection, untag a selection, remove a selection. **"Add many releases" is
      not a UI feature** — it is what import does, and a second bulk-add path duplicates the importer
      badly. No bulk condition or rating editing: those are per-copy facts and a bulk grade is almost
      always wrong.
      "Remove" is a genuine hard delete of the user's own row, unrelated to
      [4.5](../design/04-editing.md)'s soft deletion — nobody points at your collection item.
- [ ] T-90 Collection integration tests (per 11.10, 3.7, 14.3)
      Level-2 tests per service operation against a real Postgres, carrying the rights model, the
      verified-email barrier and privacy — all three are service-layer rules by decision, which is
      only a meaningful claim if the tests exercise the service.
      Include the IDOR case explicitly: another user's item id, by URL, must fail in the `WHERE`
      clause and not in an `if`.

---

## E1.4 Merge and delete

[4.4](../design/04-editing.md) and [4.5](../design/04-editing.md). **Early, not late** — the
importer mints a master per row, so duplicate masters are the normal state and merge is the
mechanism by which the catalogue becomes correct at all. Building it before the first import means
the first import lands in a system that can already repair itself.

- [ ] T-91 Merge (per 4.4, 10.4.1, 2.4.9)
      The loser keeps its row and gains `merged_into`; **no field is ever copied from loser to
      winner**. Every reference is repointed: collection items, wantlist items, images, credits,
      `release.master_id` for a master merge, and `external_id` rows.
      **The one collision case, and it is the only row merge ever deletes:** when winner and loser
      carry the same `(source, external_id)` — frequently *why* they were duplicates — T-58's unique
      index makes repointing impossible, so the loser's row goes. Safe, because the fact survives
      intact on the winner and an `external_id` row is not user content.
      A revision is written on both entities (T-57). Moderator-only (T-74).
- [ ] T-92 Unmerge (per 4.4)
      Clearing `merged_into` and reversing the repointing. It exists because merge copies nothing,
      which is what makes a wrong merge non-destructive.
      **Owed to [11.10](../design/11-stack.md)'s list: a property test that merge and unmerge are
      inverse operations.**
- [ ] T-93 Soft delete and undelete (per 4.5, 3.11)
      `deleted_at`, `deleted_by` and a reason on the entity. The URL keeps resolving to a tombstone
      that says the entry was removed. Moderator-only and reversible.
      **Delete is not merge.** Merge says "this is the same object as that one" and redirects; delete
      says "this object does not exist" and does not. Using merge as a tidier delete would put a
      user's item on a release they do not own.
      Hard deletion exists only as database surgery outside the application, for content we are
      legally obliged to remove. It is not a feature and has no UI.
- [ ] T-94 Merge screen (per 7.5, 4.4)
      `/<entity>/<id>/merge`, moderator only. **It must say, on the page, that salvaging a better
      value from the loser is an ordinary edit performed *before* the merge** — that is the
      consequence of merge copying nothing, and every UI offering merge owes the user that sentence.
- [ ] T-95 Old identifiers keep resolving (per 10.4.6, 7.8, 4.4, 4.5)
      A merged entity's old URL **301s to its winner**; a soft-deleted one returns **410** with its
      tombstone page. This is what makes a two-year-old export still land somewhere, and it is
      rung 1 of E1.6's matching ladder.
- [ ] T-96 What a collection item does in each case (per 3.11)
      Edited — **nothing happens**, because the item stores no copy of catalogue data. Merged — the
      item follows the pointer, and the user is not notified and does not need to be: the object in
      their hands did not change. Deleted — the item **survives**, renders with a marker, and still
      counts in the statistics. A user's list never shrinks without their action.
- [ ] T-97 Advisory duplicate detection reused for merge candidates (per 4.4, 2.4.9)
      The same signals as T-73, in descending confidence: a matching identifier → normalised artist +
      title + catalogue number → normalised artist + title. Duplicates surface **in the merge tooling,
      where somebody is actually looking** — never per row during an import, which would flag most of
      them by design and teach the user to ignore it.
- [ ] T-98 Accept the consequence, and record it in the UI (per 4.4)
      In the MVP an ordinary user who spots a duplicate has **no lever at all** until T-78's reports
      grow a UI in E3, where "duplicate of X" is expected to be the most used reason. Make sure the
      release page says where to report one rather than leaving it silent.

---

## E1.5 Export

[10.3](../design/10c-export.md). Ships before import, per [1.9](../design/01-product.md) — it needs
no external format knowledge and delivers [1.3](../design/01-product.md)'s "never locked in" on the
day it lands. Since [10.2.1](../design/10b-import.md) it is also a hard dependency: **the export
file is the importer's input format.**

**The contract begins with the first file that leaves the box, not with the design document.**
Appending a column is compatible; renaming, removing, reordering or retyping is not. Get the column
order right in T-100, because it is free exactly once.

- [ ] T-99 `/export` page (per 10.3.1, 10.3.3, 14.5)
      **Synchronous and streamed, always** — one indexed query straight into the response body, no
      job, no generated file, no emailed link. **Owner-only**, and there is no export button on
      someone else's public collection. 10/hour per user.
      Two sentences on the page, because a user choosing a format is choosing which they get: JSON
      round-trips exactly; CSV flattens.
- [ ] T-100 CSV writer and the column sets (per 10.3.2, 14.4)
      Collection: `item_id, release_id, discogs_release_id, artist, title, label, catno, format,
      country, released, media_condition, sleeve_condition, rating, purchased_on, purchased_at,
      note, tags, added_at`.
      Wantlist: `item_id, release_id, discogs_release_id, artist, title, label, catno, format,
      country, released, priority, note, added_at`.
      `item_id` **first** — a reorder, and therefore free only because no file has been exported yet.
      Conditions carry **our own codes** (`NM`, `VG+`), not Goldmine prose. `tags` is `;`-joined.
      Empty is empty: no `n/a`, no sentinel.
      **CSV injection is this task's obligation:** a field beginning `=`, `+`, `-` or `@` is a formula
      when the file is opened in a spreadsheet, and an artist name or a note can begin with any of
      them. The serialiser prefixes such fields.
- [ ] T-101 JSON export (per 10.3.2, 10.2.7)
      Our own shape, lossless, top-level `format_version`, plus **`exported_by` and `exported_at` in
      the header** — an `item_id` from a file is only trustworthy when the file came from the account
      importing it, and without that guard user B's item 500 and user A's item 500 are
      indistinguishable.
      Everything the model holds, including tags and owner-only fields.
- [ ] T-102 Denormalised catalogue crumbs in every row (per 10.3.5, 10.2.3)
      Artist, title, label, catno, format, country, year written into each exported row — although
      [3.1](../design/03-collection.md) forbids storing any of them on the item. **A file has no
      foreign keys and is read once, detached from us; a row has both.** That is the entire
      difference, and it is also what lets E1.6 mint a recognisable stub from a converted row.
- [ ] T-103 The flattening report (per 10.3.2)
      The exporter counts and reports what CSV flattened — tags containing `;` above all — on the
      export page and in the file's own header comment where the format allows one. This is
      [10.2](../design/10b-import.md)'s never-discard-silently obligation pointed the other way, and
      it costs one counter.
- [ ] T-104 Account dump (per 10.3.1, 10.3.5, 13.3)
      The list export plus more sections, **not a second exporter with its own bugs**: profile
      (nickname, address, bio, the three settings, registration date), collection with owner-only
      fields and tags, wantlist, conversations (E2), and the revisions the user authored as
      entity/id/timestamp — **not the snapshots**, which are catalogue content.
      No sessions, no rate-limit counters, no logs. A conversation partner appears as a nickname and
      **never as an email address**; a deleted account appears as its tombstone.
      This is [13.3](../design/13-legal.md)'s portability answer, and it is a button that runs in
      under a second.
- [ ] T-105 Export privacy and rate limit tests (per 10.3.1, 10.3.5, 14.3)
      Level-2 tests: another user's export is refused in the query; owner-only fields appear in the
      owner's file and nowhere else; the account dump never leaks a partner's address.
- [ ] T-106 The round-trip property, as far as it goes without an importer (per 11.10, 1.10)
      Serialise → parse → serialise on the JSON format, plus the CSV writer's escaping properties.
      The real property closes in T-121.

---

## E1.6 Importer — our own format only

[10.2](../design/10b-import.md). **This slice reads exactly one format: the one E1.5 writes**, and
it needs no third-party file to be finished or tested. It closes
[1.10](../design/01-product.md)'s first success criterion as a test that runs on every push.

**Two invariants govern everything here.** An import never discards data silently — whatever the
file carries that we have no home for is folded or dropped, and **either way the importer reports
it**, because a silent drop is indistinguishable from a bug and the user is the only person who
still has the original. And matching is exact or it does not happen — **a duplicate is repairable by
merge, a wrong link is silent and looks correct.**

- [ ] T-107 Migration 008: `import` (per 10.2.6, 10.4.8)
      `import(id, user_id, source, filename, file_bytes bytea, sha256, status, rows_total,
      rows_applied, created_at, finished_at, report jsonb)`, plus nullable `import_id` on
      `collection_item` and `wantlist_item` for rollback, plus nullable `source` and
      `source_instance_id` on `collection_item` for idempotency.
      `source` records **which converter ran, or `native`** — one column, and it is what makes the
      report legible a year later. The file goes in the database, **not in
      [8.2](../design/08-media.md)'s bucket**: a private prefix for one temporary blob would be a
      second access-control story and a second lifecycle for a file that lives for minutes.
- [ ] T-108 Upload page and whole-file refusals (per 7.5, 10.2.8, 10.2.9, 14.4, 14.5)
      `/import`. A 20 MB body cap at the edge (T-23) plus hard limits of **10 MB and 20,000 rows**
      checked before anything is applied. They are **refusals, not truncations** — truncating a
      collection at row 20,000 is the silent data loss this whole section exists to avoid.
      Whole-file failure only before anything is applied, and only for: no recognised format, no
      recognisable header, over a limit, or an already-imported hash without the override.
      3 imports/day and 1 concurrent per user.
- [ ] T-109 Format detection by header signature (per 10.2.1, 10.2.9)
      Our own export recognised by its column names or its JSON `format_version`; a converted source
      by its own. Never ask the user which it is. An unrecognised file is refused **with a message
      naming what we support** — and that refusal is also where a mapping UI would attach if one is
      ever built ([10.2.1](../design/10b-import.md) deferred it, deliberately).
- [ ] T-110 The import job (per 10.2.8, 11.7)
      On T-10's queue. Streaming parse — never the whole file in memory — with rows applied in
      **batches and a commit per batch**, not one transaction for the file: a transaction that fails
      on row 9,999 discards the other 9,998 and holds locks for minutes.
      `rows_applied` written every few hundred rows. A **15-minute wall-clock cap**: past it the
      import is marked failed, what was applied stays and is reported, and rollback is available. A
      job that hangs must end in a state the user can see and act on.
- [ ] T-111 Restartability (per 11.7, 10.2.8)
      A deploy kills a running import mid-file. On restart the job re-reads `file_bytes` — still on
      the row, since it is cleared only at a terminal state — re-runs the converter (pure, therefore
      yielding identical rows), skips `rows_applied` rows and continues. **That is why the counter is
      persisted per batch: it is the resume point first and a progress display second.**
- [ ] T-112 The matching ladder (per 10.2.4, 10.4.6, 10.4.1)
      Three exact rungs and no inexact ones: (1) our own `release_id`, **following `merged_into`**;
      (2) `external_id`; (3) nothing — mint a stub. **Barcode, catno+label and fuzzy matching are all
      refused**, each for its own reason: no channel carries a barcode; `catno` carries the literal
      sentinel `none` and, in one observed row, a barcode; fuzzy is the same failure with a
      similarity score in front of it.
      **No match-review screen.** With exact rungs only there is nothing to adjudicate — the review
      that exists is the report afterwards.
- [ ] T-113 Stub minting (per 10.2.3, 2.1.2, 10.4.1)
      A row that resolves to nothing mints a **release and its master** from the denormalised crumbs
      T-102 puts in every row, plus an `external_id` row where the source carried one. Never an
      unresolved item, never a triage queue: `collection_item.release_id` is `NOT NULL` and this
      design never introduces a half-linked row awaiting human triage.
      Duplicate masters are the expected outcome, and merge (E1.4) is the correction path.
- [ ] T-114 Idempotency defence 1 — `item_id` with provenance (per 10.2.7, 10.3.2)
      Re-importing a file we wrote **skips items that still exist** and reports the count — but only
      when the file's `exported_by` matches the importing user. Importing somebody else's export, or
      your own into a fresh account, creates every row, which is correct.
      **Skip rather than update:** updating would let a re-upload silently revert edits the user made
      in our UI afterwards, which is a worse failure than a no-op and an invisible one.
- [ ] T-115 Idempotency defence 3 — the file hash (per 10.2.7)
      `sha256` of the uploaded bytes, per user. A file already imported is refused with "you imported
      this file on <date>; it created N items", a link to that import, and an **explicit override for
      the user who means it**. Format-independent and always runs, which is why it is the one that
      never fails. `sha256` survives the clearing of `file_bytes`.
      (Defence 2, a source instance id, is E1.7's and only if [10.2.2](../design/10b-import.md)'s
      verification says one exists.)
- [ ] T-116 The report (per 10.2.9, 10.2.6, 3.2, 3.4)
      One `jsonb` document holding **only rows that need one** — skipped, folded, dropped, unmapped,
      unparsed — with row numbers. Ten thousand `ok` entries are a log nobody reads.
      Rendered per category with counts: rows applied · rows skipped and why · stubs created ·
      columns ignored entirely · values folded into the note · unmapped conditions and format
      descriptors · ambiguous label/catno fields · **which converter ran**. Every one of those
      categories exists because some invariant says the user must be told rather than surprised
      later.
- [ ] T-117 Failed rows downloadable as CSV (per 10.2.9)
      In the original column layout — fix in a spreadsheet, re-upload just those. It costs nothing
      because T-100 already built the CSV writer, and that the export machinery serves the importer
      is a small argument for [1.9](../design/01-product.md)'s ordering being right.
      **Partial success is the normal outcome and the unit of failure is the row.** Three bad rows
      must not cost the other 9,997.
- [ ] T-118 Rollback (per 10.2.6, 3.10)
      One button: `DELETE … WHERE import_id = ?` over the user's own collection and wantlist rows.
      **The stubs it minted into the catalogue stay** — they are catalogue entities other people may
      already reference, and deletion is moderator-only and soft in any case. The button says so: an
      honest "your items are gone, the releases remain and may be merged" beats a promise we cannot
      keep.
- [ ] T-119 Import status page (per 7.5, 10.2.8)
      `/import/<id>`: progress, then the report. **The page does not refresh itself** — no polling
      loop, no SSE, no push. The user reloads. A progress bar is not worth reopening the
      no-real-time invariant for; if it ever proves genuinely bad in use, the smallest change is a
      `<meta http-equiv="refresh">` on this one page, and that argument belongs at
      [5.3](../design/05-messaging.md) where the invariant lives, not smuggled in with an
      implementation.
- [ ] T-120 The 7-day file sweeper (per 10.4.8)
      `file_bytes` is set to `NULL` in the same transaction that writes a terminal status — but a
      deploy can leave an import non-terminal indefinitely, and for exactly those rows "deleted when
      the job ends" silently means "kept for ever". A **daily systemd timer** nulls `file_bytes` on
      any import row untouched for 7 days and marks it failed with that reason.
      The retention claim is only as good as the case that goes wrong, and this is the sentence
      [13.3](../design/13-legal.md) gets to state.
- [ ] T-121 The round trip closes (per 11.10, 1.10)
      **export → import → export produces the same state**, as a property that runs on every push
      rather than a thing checked once by hand. It is [1.10](../design/01-product.md)'s first success
      criterion turned into a test, and it needs no third-party file.

---

## E1.7 Discogs converters

[10.2.1](../design/10b-import.md). **Not optional despite being a separate slice:** our own format
only exists after somebody has used the site, so until this ships there is no way for a real
collector to get in and both of [1.10](../design/01-product.md)'s criteria stay unreachable. The
separation buys blast radius, not deferral.

**Everything here is a pure function in `musilka-domain` with no database access.** A converter
returns **rows *and* findings**; the findings cross the boundary and land in T-116's single report,
because the layering is ours, not the user's. Every third-party quirk is confined to this module and
can never reach the database.

- [~] T-122 Get a real Discogs **collection** export (per 10.2.2, 15.4 risk 3) — blocks this slice
      **The cheapest research task on [15.4](../design/15-roadmap.md)'s list, and the only thing this
      slice is waiting on.** What was inspected in 2026-08-14 was an *inventory* export; the
      collection export's own column set, the literal name of the default folder
      ([10.2.5](../design/10b-import.md)) and whether `Collection Item Instance ID` exists
      ([10.2.7](../design/10b-import.md)) are still guesses.
      Ask any Discogs user. Then close [10.2.2](../design/10b-import.md) in `design/`.
- [ ] T-123 CSV reading and header resolution (per 10.2.2, 11.9)
      `cassava` with a real parser — **never split on `,`**, because label names legitimately contain
      commas. Columns resolved **by header name through a lookup table, never by position**, and
      **every column treated as optional**: empty columns are genuinely empty, not absent. An
      unrecognised column is reported, not ignored, which is what turns the first real file into the
      verification.
      A wrong guess therefore costs one row in a mapping table.
- [ ] T-124 Label / catno positional pairing (per 10.2.5, NOTES 2026-08-14 finding 3)
      Split the field's contents and pair positionally **only when the two splits yield equal
      counts**. When they do not, the pairing is genuinely ambiguous: take the whole field as a
      single label name and put the raw strings in the report. **Guessing here silently invents a
      catalogue number.**
      One observed row read `"United Guttural Records, United Guttural Records"` /
      `"UGR 018, UG018"` — the same label twice with two different numbers, so deduplication is
      wrong too.
- [ ] T-125 Format descriptor decomposition (per 10.2.5, 2.4.2, NOTES finding 4)
      Optional `2x` quantity prefix → quantity; first token → media type; remaining tokens →
      descriptors matched against T-64's vocabulary. **Unknown descriptors go into the free-text
      companion field and are reported** — `FYE` is a retailer's abbreviation and will never be in a
      sensible vocabulary, which is exactly the case the curated-set invariant provides for.
- [ ] T-126 Condition parsing (per 10.2.5, NOTES finding 6)
      **Parse the abbreviation inside the brackets, never the prose:** `Near Mint (NM or M-)` → `NM`.
      Straight into T-66's vocabulary, sleeve included with its three extra values. An unrecognised
      grade becomes `NULL` plus a report line — both columns are nullable precisely because most rows
      carry no grade at all.
- [ ] T-127 Folders → tags, and rating (per 10.2.5, 3.3, 3.2)
      One tag per folder, losslessly — a Discogs folder imports as a tag with no loss, which is what
      made refusing folders cheap. **The default folder is skipped rather than imported**: a tag every
      item carries groups nothing. Its exact literal comes from T-122.
      Rating copied as-is; Discogs' `0` means unrated and becomes `NULL`.
- [ ] T-128 Discarded and folded columns (per 10.2.9, 1.7, 3.2, 3.4, NOTES finding 8)
      Marketplace columns — `price`, `status`, `accept_offer`, `listed`, `quantity`, `weight` — are
      **discarded on sight** rather than stored "just in case", because nothing order-shaped enters
      the model. [3.2](../design/03-collection.md)'s price column goes the same way. Custom-field
      columns are **appended to the item's note, labelled with their column name**.
      All three are reported. Lossy in shape, not in content.
- [ ] T-129 `external_id` rows from `release_id` (per 10.4.1, 10.2.4)
      The converter writes `(release, discogs, <id>)` for every row that has one, which is what fills
      rung 2 of T-112's ladder and what T-100's `discogs_release_id` column reads back out. **That
      single field is what actually lets a user rebuild their collection somewhere else**; the rest
      of our column names are decoration.
- [ ] T-130 Wantlist converter (per 10.2.1, 3.5)
      Nearly free once the collection converter exists — the same field mapping writing a
      `wantlist_item` row instead. Both converters are in the MVP; neither is optional.
- [ ] T-131 Converter property tests (per 11.10, 10.2.1)
      Owed to [11.10](../design/11-stack.md)'s list: CSV field splitting, the equal-count pairing
      rule, and that a converter's output is **byte-compatible with what our exporter emits** — which
      is what makes "convert then import" provably the same path as "import a file we wrote".
- [ ] T-132 Idempotency defence 2, if T-122 says it exists (per 10.2.7)
      `collection_item.source` + `source_instance_id`, unique per user; a re-import **skips** rows it
      already created and reports the count. If the collection export carries no instance id, this
      task is `[-]` and T-115 is the only defence for converted files — the design accommodates both
      answers, which is why [10.2.7](../design/10b-import.md) could close while
      [10.2.2](../design/10b-import.md) could not.
      **Say so on the upload page either way:** without a per-item identifier there is no incremental
      re-import, and stating that beats a heuristic that silently duplicates ten thousand rows.

---

## E1.8 Search

[Section 7](../design/07-search-ux.md). After the converters, because tuning a search over an empty
catalogue is guesswork. The pickers already shipped in T-72; what is left is the public search
surface.

- [ ] T-133 Migration 009: the search columns and indexes (per 7.2)
      `artist_name.search_tsv` and `label.search_tsv` as **generated columns**
      (`to_tsvector('simple', unaccent(name))`), which is the point of putting artist search on
      `artist_name` — one row per name gives [2.2.1](../design/02-catalogue-model.md)'s "every name
      in one index" for free and cannot drift.
      `master.search_tsv` maintained by **triggers** on `master`, `master_credit` and `artist_name`,
      because it must contain the credited artists' names or the single most common query in the
      product (`kino gruppa krovi`) matches nothing.
      GIN on each `search_tsv`, plus the `gin_trgm_ops` indexes from T-72.
      **`simple`, not `english` or `russian`** — we cannot know a row's language and will not be
      given a per-row language column; stemming with the wrong dictionary is worse than not stemming.
      **This is the one cache the design permits**, and it passes the test that admits it: one
      statement rebuilds it from the truth, so a stale value is detectable.
- [ ] T-134 The search service (per 7.1, 7.2)
      One box, in order: (1) if the query normalises to a barcode or matches a catalogue number
      exactly, release matches come first and a single match navigates straight to the release page;
      (2) otherwise `websearch_to_tsquery` over artists, masters and labels, rendered as three
      labelled groups on one page — no tabs, no "search in:" dropdown.
      **Trigram only as a fallback**, when the exact query returned nothing, and **labelled as
      such** ("Nothing found for X — did you mean Y"). Blending trigram distance into the primary
      ranking makes results unpredictable for a gain nobody asked for; silent correction is how a
      search stops being predictable.
      Ranking is `ts_rank` plus one tie-break — exact prefix match on the primary name or title.
      There is no popularity signal because there is none to have.
- [ ] T-135 `/search` page (per 7.5, 7.1)
      Three groups on one screen. **Releases are not a text-search result type**, tracks and users
      are not searchable at all, and companies are not either — each would be an empty heading on
      every results page.
- [ ] T-136 `/releases`, the catalogue list (per 7.3, 7.5)
      T-81's component pointed at the whole catalogue. Unfiltered `/releases` is indexable; every
      filtered URL is not (T-155).
- [ ] T-137 Autocomplete on the main search box (per 7.4)
      The same HTMX fragment as T-72, where it is a convenience rather than a requirement.
- [ ] T-138 Home page (per 7.5)
      `/`: a search box and recently added releases. **Not a dashboard.**
- [ ] T-139 Search performance check (per 9.1, 9.5)
      [9.1](../design/09-nfr.md) budgets search with live facet counts at 300 ms p95 and names it the
      slowest path in the product. Measure it with `pg_stat_statements` once the catalogue has real
      rows in it. **When a budget is missed the answer is an index or a rewritten query** — not a
      cache, and not a bigger architecture. A bigger VPS is the sanctioned fix.
- [ ] T-140 Record the known gap where a user will hit it (per 7.4, NOTES)
      A Latin-typed query for a **Cyrillic title** finds nothing: artists have variant name rows,
      titles do not. The workaround is the artist page; the fix is an additive
      `master_title(master_id, kind, title)` table mirroring `artist_name`, which is E3 and one
      migration away.
      **This is a recorded limitation, not a surprise** — expect it to be the first search complaint
      from a real user, and make sure the empty-results page points at the artist route.

---

## E1.9 Images

[Section 8](../design/08-media.md). Last on purpose: **the one irreversible decision in the design**
([8.1](../design/08-media.md) discards originals) and [1.4](../design/01-product.md)'s predicted
first cost overrun. Nothing uploaded before a change of mind can ever be re-derived at higher
resolution.

- [ ] T-141 Storage port and the filesystem implementation (per 8.2, 12.2)
      A small interface — `put`, `get`, `delete`, `url` — with a filesystem implementation for
      development and tests, so the test suite needs no MinIO container. This is the task
      [section 8](../design/08-media.md)'s working notes wanted in E0; it is here because
      [15.1](../design/15-roadmap.md)'s ordering wins.
- [ ] T-142 Migration 010: `image` (per 8.2, 6.3)
      `image(id, sha256 unique, width, height, byte_size, uploaded_by, created_at, blocked_at)`,
      referenced by `release_image` (T-54) and `user.avatar_id` (T-34). **No variants table** — the
      two derivatives are implied by the key scheme, and adding a third size later means generating
      it, not migrating.
- [ ] T-143 Upload validation (per 8.1, 14.4)
      15 MB rejected **at the proxy** before a byte reaches the application (T-23); JPEG, PNG and
      WebP only, **determined by sniffing the leading bytes** — the client's `Content-Type` and the
      filename are ignored and **the filename is discarded entirely**, since keys are content
      addressed. 6000 px maximum and 300 px minimum on the longest side, rejected rather than
      silently downscaled.
      **SVG is refused outright**: it is a script container, and no amount of rasterising makes
      accepting it a good idea. Not HEIC, not GIF. 30 uploads/hour per user.
- [ ] T-144 `vips` derivatives (per 8.3, 14.4)
      `vipsthumbnail` as a **short-lived subprocess**, not an in-process library, with a
      pixel-dimension cap checked from the header before full decode (also the decompression-bomb
      defence), a memory limit and a timeout. It is the largest attack surface in the system: a C
      library parsing hostile input, and a separate process is what contains a crash to the child.
      Two variants and no more: `full` at 1600 px longest side WebP q82, `thumb` a 300 px
      centre-cropped square WebP q80. **EXIF stripped after auto-rotation** — a phone photo of a
      sleeve carries the owner's home coordinates, which requires *not* asking `vips` to preserve
      metadata. **The re-encode is the sanitiser**: a polyglot does not survive decode and re-encode.
      **Synchronous, in the request.** The user has just chosen a file and is waiting on their own
      action; three seconds is the budget, ten is a bug. A queued derivative would mean an `image`
      row with nothing renderable behind it, which is a placeholder state in every template plus a
      poll to resolve it.
- [ ] T-145 Content-addressed keys and the original discarded (per 8.1, 8.2)
      `img/<sha256[0:2]>/<sha256>/full.webp` and `…/thumb.webp`. Two users uploading the same scan
      store one object. Objects are immutable, so
      `Cache-Control: public, max-age=31536000, immutable` is honest and needs no CDN and no purge
      story.
      **The original is discarded once the derivatives exist**, and this is the irreversible one. If
      it is ever reopened it is "keep originals from this date onward", and older images simply do
      not have one.
- [ ] T-146 S3 implementation and bucket policy (per 8.2, 12.1, 14.3) — *provider named by T-191*
      The S3 API and nothing beyond it: no signed-URL upload flow, no bucket events, no lifecycle
      rules, no versioning. The image prefix is public-read and **served from the bucket's own
      endpoint**, so uploaded bytes are never same-origin with a session cookie. Objects written with
      the `Content-Type` **we** determined, `Content-Disposition: inline` and `nosniff`. The bucket
      is not writable by browsers — uploads go through the application, so there is no presigned-PUT
      path to abuse.
      Add the bucket origin to T-13's `img-src`, **and to nothing else**.
- [ ] T-147 Release images (per 2.4.8, 8.1)
      `release_image` with `type ∈ front|back|sleeve|media|other`, exactly one `is_primary` per
      release, **cap 8**. If a removed image was primary, the next by position becomes primary.
- [ ] T-148 Rendering (per 9.1, 7.7, 8.3)
      `srcset` over the two derivatives, `loading="lazy"`, and **`alt` text generated rather than
      requested**: `"Front cover of <title> (<label>, <year>)"` from the image type; decorative
      images get `alt=""`. Asking uploaders to write alt text would produce empty strings and lies.
      Keep a catalogue page under ~100 KB of HTML and CSS — at this scale the network, not the
      database, is the whole of perceived latency.
- [ ] T-149 Avatar (per 6.3, 8.2)
      One per user, on the same pipeline and **never its own**: a hard size cap, resized to a single
      small square, no gallery, no history. It is the second consumer of the storage port and ships
      after the catalogue's images.
- [ ] T-150 Takedown (per 8.4, 13.5, 4.10)
      **The only hard delete of something other people reference, and the exception is principled:**
      a tombstone that still serves the file answers nothing. The `release_image` row goes; when the
      last reference to a blob goes, the objects are deleted from the bucket. What survives is an
      `image` row reduced to its `sha256` and a `blocked_at` — and because storage is content
      addressed, **re-uploading the same file is refused by hash for free**.
      The catalogue entity is untouched: removing an image is not deleting a release. Who may:
      moderators and admins, plus the uploader removing their own image before anyone reports it. A
      rights-holder is not a user and does not need an account — which is why the mechanism is an
      address (T-153).
      A moderator-only `DELETE` route is enough; there is no report UI to reach it from until E3.
      Record the removal and its reason in the security log.
- [~] T-151 Weekly image bucket sync (per 9.3, 8.2) — deferred with T-194
      `rclone sync` of the image prefix to a second bucket, weekly. `pg_dump` does not cover the
      bucket, and **because the original is discarded, image loss is permanent** — there is nothing
      to re-derive from. Weekly is enough given immutable content-addressed keys: nothing is ever
      modified, only added, so a week's exposure is a week's uploads.
      **The number to watch is this sync's time and egress**, not the storage bill — that is the form
      [1.4](../design/01-product.md)'s warning actually takes. [8.2](../design/08-media.md) estimates
      ~6.5 GB at [1.4](../design/01-product.md)'s counts.

---

## Before the first real user

Not a tenth slice — the exit checklist. [13.3](../design/13-legal.md) hands over four documents that
are "easy to leave until after launch, which is why they are named here", and
[9.3](../design/09-nfr.md) and [15.4](../design/15-roadmap.md) add the rest.

**Re-sorted 2026-08-15.** [13.3](../design/13-legal.md) now reads "the first real user" as **the
first user who is not the author** — on a single-user localhost instance there is no other party's
personal data being processed, so the legal documents have nothing to govern yet. They move behind
the deployment as `[~]`, together with everything else that presupposes a public site. **Three
tasks stay in E1 unchanged, because they are about the software rather than the service:** T-154
(the guidelines are how *you* stay consistent with your own model), T-156 (error pages are pages),
and T-158 (accessibility is a property of the HTML, not of where it is served from).
T-160 stays too — [9.3](../design/09-nfr.md)'s drill is about the restore script, and E1 is still
not finished until it has been run once with a real collection in the database.

- [~] T-152 Privacy policy (per 13.3, 6.5, 5.8, 14.6) — deferred with the deployment
      Written to the GDPR. **The exhaustive table of personal data we hold**, because a policy that
      is not exhaustive is worse than none. The three statements [6.5](../design/06-accounts.md)
      forbids overstating, in the words they must appear in: deletion *empties* the account rather
      than removing the row; **private messages are private from other users, not from us** — plain
      text, readable by the operator, surviving the sender's deletion; the only profile data held is
      an address, a nickname and an optional bio.
      Lawful bases: contract for the account, legitimate interest for logs and abuse limits. **No
      consent basis anywhere**, because there is nothing to consent to. Backups hold a deleted
      account for up to ~90 days, disclosed, and are never restored to recover one. Sub-processors
      named — three lines, filled in by T-21 and T-22.
- [~] T-153 Terms of service, `/licence` and `/contact` (per 13.1, 13.5, 13.2, 9.2, 8.4) — deferred with the deployment
      The ToS says five things: **16+** (a checkbox, never verified); catalogue contributions are
      **CC0 1.0 including any sui generis database right**, and the contributor confirms they are
      entitled to grant it; we may edit, merge or remove any catalogue contribution and may suspend
      an account for abuse; the service is as-is with **no availability promise** — what we promise
      instead is durability, no data loss beyond 24 hours, and that is the sentence worth putting in
      front of users; and the governing law is the operator's country of residence.
      `/licence` states what is CC0 (the catalogue) and what is emphatically not (**images**, and all
      user data). `/contact` publishes `abuse@` and the takedown path: acknowledgement within a few
      days, content removed or the claim declined with a reason, a note kept of what and why.
      **Ten minutes with a lawyer before publishing**, per [NOTES.md](../design/NOTES.md)'s three
      unverified claims: the CC0-waiver-by-ToS assumption, the DSA/DMCA position, and
      [10.5.1](../design/10e-legal-sources.md).
- [ ] T-154 Data entry guidelines (per 4.11)
      **One short page**, not Discogs' constitution. Transcribe from the object and never translate
      or transliterate; object facts on the release, work facts on the master; a stub artist with
      nothing but a name is fine; when to merge rather than edit; how to ask for a vocabulary
      addition instead of forcing a value.
      The decisions already taken are worthless if only their author knows them. It grows a paragraph
      each time a moderator decision proves it needed one.
- [~] T-155 `robots.txt`, sitemap and per-page tags (per 7.8) — deferred with the deployment
      Indexable: artist, master, release and label pages, and `/releases` unfiltered.
      **`Disallow` every filtered and search URL** — [7.3](../design/07-search-ux.md)'s facets
      multiply into a combinatorially infinite space, and letting a crawler in is a crawl trap that
      would also be the closest thing to a load test this box ever gets. `noindex` and `Disallow`
      everything behind login, every form, `/settings`, `/import`, `/export`.
      Per page: `<title>`, meta description, a **self-referential `rel="canonical"`** so the cosmetic
      slug cannot fork a page into many, OpenGraph with the primary image, and minimal schema.org
      JSON-LD (`MusicAlbum`, `MusicRelease`, `MusicGroup`).
      `sitemap.xml` generated on request from a single query — no static file, no cron.
      **The `Disallow` half is not deferrable in the way the rest is:** it is a defence rather than
      an SEO nicety, so it ships **in the same sitting as the deployment**, never after it. The
      canonical, OpenGraph and JSON-LD half can follow at leisure.
- [ ] T-156 Static and error pages (per 7.5, 13.x)
      `/about`, plus 403, 404 and 500. Screen 24 of [7.5](../design/07-search-ux.md)'s list, and the
      last one.
- [~] T-157 Deliverability check against a real inbox (per 12.5, 15.4 risk 2) — deferred with T-22
      **A ten-minute check, and it now protects the deployment rather than a success criterion** —
      T-159 is dropped, so there is no invitation to time it against, but nothing else about it
      changed. Send all four templates to a real mailbox at a large provider and confirm they arrive
      in the inbox, not the spam folder. DMARC moves from `p=none` to `p=quarantine` once the reports
      are clean, and the domain gets reputation time before anyone else is pointed at it.
      **What the local mail catcher does and does not tell you:** it proves the templates render, the
      queue job runs and the verification flow closes. It says nothing whatsoever about inbox
      placement, and treating a green local flow as evidence here is the specific mistake this task
      exists to prevent.
      If this fails, [6.1](../design/06-accounts.md)'s barrier fails, and with it
      [4.9](../design/04-editing.md), [5.7](../design/05-messaging.md) and
      [14.5](../design/14-security.md) at once.
- [ ] T-158 Accessibility and browser pass (per 7.7, 9.6)
      Walk [7.7](../design/07-search-ux.md)'s list: one `<h1>` and landmarks per page; every control
      labelled, errors associated and summarised; visible focus never removed; **WCAG AA contrast in
      both themes — the dark theme is where this is usually lost**; no `title`-only affordances; no
      colour as the sole carrier of meaning.
      Test manually on one Chromium, one Firefox and a phone. **Check every core flow with
      JavaScript disabled** — that is the claim [11.3](../design/11-stack.md) makes and the reason
      the accessibility story is cheap.
      Confirm every `light-dark()` has its plain fallback declaration (T-11).
- [-] T-159 ~~Find a willing collector~~ (per 1.10, 15.4 risk 8) — **dropped 2026-08-15**
      [1.10](../design/01-product.md)'s criterion 2 is withdrawn, so there is no second person to
      find and [15.4](../design/15-roadmap.md)'s risk 8 is gone with it.
      **What does not go with it:** T-122 still needs a real Discogs **collection export**, and it
      still has to come from somebody else. Losing the stranger as a criterion does not remove the
      one file we need from a person — the ask is just much smaller now (send me a CSV, not use my
      site).
- [ ] T-160 Second restore drill and a `/status` read (per 9.3, 9.5)
      Monthly by hand, from the real production backup, now with real data in it. Row counts within
      expectation, newest revision from yesterday, app boots, one release page renders, one login
      works.
      E1 is not finished until this has been done once with a user's data in the database.

## Working notes

**2026-08-15 — where the slices bleed, deliberately.** Three tasks sit outside the slice their
section belongs to, and each has a dependency reason rather than a preference: T-72 puts the trigram
index and the artist picker in E1.2 because every credit must resolve to an artist entity, so the
picker is a precondition of the edit form ([7.4](../design/07-search-ux.md) says so explicitly);
T-81 builds the faceted list for the collection view in E1.3 and E1.8 reuses it for the catalogue;
and T-78's `report` table lands in E1.2 because [8.4](../design/08-media.md)'s takedown and
[2.3.2](../design/02-catalogue-model.md)'s vocabulary requests both need a row to write, long before
E3 gives it a queue.

**2026-08-15 — the hook list, and where each one is.** [15.5](../design/15-roadmap.md) named eight
things that must land early because retrofitting them is the expensive rework. Service layer with an
explicit actor — E0 T-16 and everywhere after. Job queue — E0 T-10. `bigint` identity ids — E0 T-9.
`external_id` — T-58. Revisions on every mutation including merge and delete, with monotonic ids —
T-56, T-57, T-91, T-93. `revision.source`/`import_id` — T-56, T-76. The export column contract —
T-100. `import.file_bytes` plus the sweeper — T-107, T-120. **None of the five additions to
[1.9](../design/01-product.md)'s original three is new work**; each is a column or a property of
something already being built.

**2026-08-15 — what has no hook and must not grow one "just in case".** The public API
([10.1](../design/10a-public-api.md) found no audience, and everything it would need already exists
for other reasons), webhooks (refused permanently), and
[10.3.4](../design/10c-export.md)'s catalogue dump (the same serialisers plus a cron entry).

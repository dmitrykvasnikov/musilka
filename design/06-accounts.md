# 6. Accounts and profiles

**Priority:** P1

**What this section is not free to decide.** More than any other section, this one arrives with its
answers already implied. [4.9](04-editing.md) and [5.7](05-messaging.md) both spend the
**verified-email wall** as their only anti-abuse mechanism, so email verification is load-bearing
rather than a preference. [3.7](03-collection.md) has already fixed two privacy settings and
[5.9](05-messaging.md) one notification setting, which makes [6.6](06-accounts.md) an act of
enumeration rather than design. [5.8](05-messaging.md) has handed [6.5](06-accounts.md) a
requirement it must satisfy. Password mechanics belong to [14.2](14-security.md), rate limits to
[14.5](14-security.md), image storage to [8.2](08-media.md), and the legal text to
[13.3](13-legal.md). What is genuinely open here is the product shape of the account, and it is
small.

- [x] 6.1 Registration: email+password? OAuth (Google/GitHub/VK)? Mandatory email confirmation?
      **Decision: email and password, open registration, confirmation mandatory — and no OAuth, at
      any stage.**
      **Confirmation is not a courtesy, it is the barrier.** [4.9](04-editing.md) turned down captcha
      and honeypots, and [5.7](05-messaging.md) turned down every anti-spam mechanism heavier than a
      rate limit, both on the explicit grounds that a verified address is enough at our scale. That
      makes it the one thing here that may not be softened: an unverified account may **read public
      pages and nothing else** — no catalogue edit ([section 4](04-editing.md)), no collection
      ([section 3](03-collection.md)), no message ([section 5](05-messaging.md)). One check in the
      service layer ([4.7](04-editing.md)), so [10.1](10a-public-api.md) inherits it.
      **No OAuth.** Each provider is a console registration, a client secret in configuration, a
      callback route, and an account-linking matrix ("same address, different provider", "provider
      returns no address", "provider account deleted"). It removes no work, because the mail sender,
      the address column and the verification flow are all needed regardless — it only adds a second
      way in. Additive later as a provider table beside the password column, with no change to
      anything else.
      Token is single-use, stored hashed, expires in 24 hours; resend is rate-limited
      ([14.5](14-security.md)). Password hashing and policy are [14.2](14-security.md)'s to choose;
      this section requires only that a password exists.
- [x] 6.2 Password recovery, email change, 2FA — what is in the MVP.
      **Decision: recovery in, email change in, 2FA out.**
      **Recovery is not optional.** Without it a forgotten password permanently orphans a collection,
      which contradicts the trust argument [3.9](03-collection.md) makes for export — a user who
      cannot leave and cannot get back in has been locked in the worst way. Single-use token, stored
      hashed, one hour, rate-limited; a successful reset **invalidates every session**.
      **The response is identical whether or not the address is registered.** Note this is not the
      silent failure [5.7](05-messaging.md) rejected: there we would have lied to an account holder
      about their own message; here we decline to confirm a stranger's registration to whoever typed
      their address.
      **Email change requires confirming the new address before it takes effect** (a `pending_email`
      plus a token), and sends a warning to the old one. The attack this closes is real and cheap: a
      borrowed session that could rewrite the address owns the account, because the address is what
      recovery trusts. Every session is invalidated on change.
      **No 2FA in the MVP.** There is nothing money-shaped anywhere in this design
      ([1.7](01-product.md), [3.2](03-collection.md)), so the worst case of a stolen account is a
      reversible catalogue edit ([4.4](04-editing.md)) and a read mailbox. Against that: secret
      storage, recovery codes, and a lockout path with no support channel behind it. **If it is ever
      reopened it is TOTP for everyone**, not a privilege of the admin role — a second factor with
      one user is a code path with no test coverage.
      Session and cookie mechanics are [14.3](14-security.md)'s. The only rule this section fixes is
      that password reset and email change invalidate all sessions.
- [x] 6.3 Public profile: nickname, avatar, bio, location, links, collection, wantlist, contribution stats.
      **Decision: nickname, avatar, bio, location, member-since, edit count, and the two lists when
      they are public. No links field, and nothing we have no use for.**

      ```
      user(id, email, password_hash, email_verified_at, role,
           nickname, avatar_id, bio, location, created_at,
           collection_visibility, wantlist_visibility, notify_new_message,
           deleted_at, disabled_at)
      ```

      - **Avatar: yes, one per user, on [8.2](08-media.md)'s pipeline and never its own.** Bounded by
        construction — one image per account at tens of accounts — so it does not threaten
        [1.4](01-product.md)'s size estimate the way [5.4](05-messaging.md)'s attachments would. Hard
        size cap, resized to a single small square, no gallery, no history. It is the **second**
        consumer of whatever [8.2](08-media.md) chooses and ships after the catalogue's images; it
        may not invent a parallel upload path, and [8.4](08-media.md)'s moderation question covers it
        unchanged.
      - **Bio: plain text, capped at 500 characters, rendered by [5.5](05-messaging.md)'s rule** —
        escaped, newlines preserved, URLs linkified with `rel="noopener nofollow ugc"`. Deliberately
        the same rule rather than a second one: two text policies means two sanitising stories and
        two places to get XSS wrong.
      - **Location: free text, optional, not a vocabulary.** It is a line on a profile, not a field
        anything groups by, so the curated-set invariant does not reach it (same reasoning as
        [3.3](03-collection.md)'s tags).
      - **No separate links field.** A link list is a table, a rendering, a per-row validation and an
        SEO-spam target, bought for a page that at our scale nobody reads. URLs go in the bio and are
        linkified there, already `nofollow ugc`.
      - **Collection and wantlist appear only when [3.7](03-collection.md) says `public`.** A `link`
        list is **not** linked from the profile — publishing the unguessable URL on a public page is
        the one thing that would make it meaningless.
      - **Contribution stats: member-since and an edit count, computed live.** [4.8](04-editing.md)
        kept exactly this and made it gate nothing; no ranks, no badges, no leaderboard. Live
        `GROUP BY` like [3.8](03-collection.md), no counter column to drift.
      - **Nothing else is collected.** No real name, no birthday, no gender, no phone. Data we do not
        hold is data [13.3](13-legal.md) does not have to account for and [14.6](14-security.md) does
        not have to protect.
      Profile pages are readable without logging in, since everything on them is already public.
- [x] 6.4 Nickname uniqueness, nickname changes, reserved names.
      **Decision: case-insensitive unique, renameable once per 30 days, old names held for ever.**
      - **Uniqueness is case-insensitive** (unique index on `lower(nickname)`), display preserves the
        case the user typed. `Kvasnikov` and `kvasnikov` must not be two people.
      - **Charset `[A-Za-z0-9_-]`, 3–30 characters.** Deliberately ASCII, and this is not a breach of
        [2.2.1](02-catalogue-model.md)'s no-transliteration rule: that invariant governs *catalogue
        data*, names that belong to an object we are describing. A nickname is an identifier the user
        invents for our URLs, and confusable-script impersonation is a real problem we decline to own
        homograph detection for.
      - **Profiles live at `/u/<nickname>`.** The prefix means no nickname can ever collide with a
        route, which removes the usual reason for a long reserved list. What remains is impersonation
        only: `admin`, `moderator`, `support`, `staff`, `system`, `musilka`, seeded and short.
      - **Renaming is allowed, at most once per 30 days**, and the old nickname is **held by that
        user permanently** in a reservation table — never returned to the pool. Nothing internal
        breaks, because every reference in the system is to `user.id`; the accepted cost is that
        external links to `/u/<old>` return 404 rather than redirecting. The reservation exists to
        stop someone claiming a name a known contributor just left, not to preserve links.
- [x] 6.5 Account deletion: what happens to their catalogue edits, collection, messages (GDPR-compatible — see [13.3](13-legal.md)).
      **Decision: the account is anonymised in place and the row is never deleted.**
      A hard `DELETE` is not available to us and this is structural, not a preference:
      [5.1](05-messaging.md) requires exactly two `conversation_participant` rows per conversation,
      and [4.2](04-editing.md) hangs every revision off its author. A missing user row would leave
      both dangling. Anonymisation is also the stronger answer to erasure — the personal data is
      gone, not merely hidden.

      | | |
      |---|---|
      | **Cleared** | email, `password_hash`, bio, location, avatar (file deleted per [section 8](08-media.md)), notification setting; nickname replaced by `deleted-<id>`, which the reservation table ([6.4](06-accounts.md)) makes unclaimable |
      | **Hard-deleted** | collection items and their tags, wantlist items, the user's own tag definitions, blocks they created — nobody else points at any of it ([3.10](03-collection.md)) |
      | **Kept, attributed to the tombstone** | catalogue edits and the revision history behind them; conversations, participant rows and message bodies |

      **Message bodies survive, and the sender renders as `[deleted]`.** This is the tension
      [5.8](05-messaging.md) handed here, and it resolves the same way an email you received does:
      the sender leaving does not unsend it, and the other party's mailbox is *their* record of a
      conversation they took part in. Erasure is satisfied by anonymising the account, which is
      what leaves the message pseudonymous rather than personal. [13.3](13-legal.md) owns saying this
      in the privacy policy; it must not promise more than this.
      **Catalogue edits stay** because the history *is* the integrity mechanism
      ([4.2](04-editing.md)) — unpicking one author's contributions would break revert and leave
      entries nobody can account for.
      **Mechanics:** the deletion page offers an export first ([10.3](10c-export.md),
      [3.9](03-collection.md)'s trust argument), then requires typing the nickname and the password.
      It is immediate and irreversible, said plainly on the page — an undo window would need a
      scheduled purge job and a fourth account state to explain. Login is refused afterwards;
      re-registering with the same address creates a **new** account with no link to the old one.
      **`disabled_at` is a separate lever, and this section owns it because nothing else does.**
      [5.7](05-messaging.md) sends the admin a report email with no action attached to it; the action
      is disabling an account — login refused, content untouched, reversible. One column and no UI in
      the MVP: the admin runs an `UPDATE`. Distinct from deletion in both meaning and effect.
- [x] 6.6 Privacy and notification settings — the full list.
      **Decision: three columns on `user`, and this is the complete list.**

      ```
      collection_visibility   public | link | private     default private   [3.7]
      wantlist_visibility     public | link | private     default public    [3.7]
      notify_new_message      bool                        default on        [5.9]
      ```

      **Defaults carry the argument [3.7](03-collection.md) made:** a collection is an inventory of
      valuable objects in someone's home and starts private; a wantlist exists to be seen and starts
      public. Notification starts on, because a mailbox whose arrivals are silent is
      indistinguishable from a broken one, and [5.9](05-messaging.md)'s throttle already caps it at
      one email per conversation per read.
      **Columns, not a settings table.** A key/value settings table is a schema you cannot query,
      cannot constrain and cannot index — for three values it is pure loss.
      **What is deliberately not a setting**, each already decided elsewhere: transactional mail
      (verification, reset, email change) cannot be opted out of, because it is how the account
      works; per-conversation muting, digests and frequency pickers ([5.9](05-messaging.md));
      per-item collection privacy ([3.7](03-collection.md)); profile visibility, since the profile
      holds only what the user chose to type and the lists carry their own settings; and "who may
      message me", which [5.7](05-messaging.md)'s blocking already covers reactively.
      Enforcement is in the service layer ([10.4.4](10d-model-requirements.md)) so
      [10.1.5](10a-public-api.md) inherits it.

## Working notes

- **2026-08-15 — Schema delta implied by the decisions above.** One table plus two small ones:
  `user` (as sketched in [6.3](06-accounts.md)), `nickname_reservation(nickname, user_id,
  released_at)` for [6.4](06-accounts.md), and one token table for verification, password reset and
  pending email change — `user_token(user_id, kind, token_hash, expires_at, consumed_at)` rather
  than three near-identical tables. No settings table ([6.6](06-accounts.md)), no OAuth provider
  table ([6.1](06-accounts.md)), no 2FA secret ([6.2](06-accounts.md)), no session table until
  [14.3](14-security.md) says whether one is needed.
- **2026-08-15 — What this section hands on.** [13.3](13-legal.md) inherits three statements to make
  and may not overstate any of them: what deletion actually erases and what it keeps
  ([6.5](06-accounts.md)), that message bodies are plaintext and survive the sender
  ([5.8](05-messaging.md)), and that the only personal data held is an address, a nickname and
  whatever the user typed into a bio ([6.3](06-accounts.md)). [14.2](14-security.md) inherits
  password hashing and policy, [14.5](14-security.md) the rate limits on registration, login, reset
  and resend, [14.6](14-security.md) the retention question for logs.
- **2026-08-15 — Where this lands in the roadmap.** Registration, verification, login and recovery
  are a precondition of everything in [1.9](01-product.md)'s MVP, so they are the first vertical
  after the catalogue can be read. Email change, the profile page and the avatar follow; deletion and
  `disabled_at` come with [section 5](05-messaging.md), since the tombstone only has to exist once
  there are messages to survive it.
- **2026-08-15 — The mail sender is the one piece of infrastructure this section adds**, and it now
  carries four templates (verify, reset, email-change warning, new-message notification from
  [5.9](05-messaging.md)). Deliverability of transactional mail from one small box
  ([1.4](01-product.md)) is a real question and it belongs to [section 12](12-infrastructure.md),
  not here — but if it is answered with an external provider, that is the first outbound dependency
  in the system and [1.5](01-product.md)'s two-channels invariant should be re-read to confirm it
  does not touch it (it does not: that rule governs catalogue *data* entering, not mail leaving).

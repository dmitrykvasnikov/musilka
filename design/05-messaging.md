# 5. Messaging between users

**Priority:** P0 for the module

**Prior question, answered before 5.1 because everything below depends on it: why does this module
exist?** On Discogs, messaging serves the marketplace — buyer talks to seller. We removed the
marketplace permanently ([1.7](01-product.md)), so that reason went with it. What survives is
narrower: asking another collector about a copy they own, and contacting whoever made a catalogue
edit you disagree with — the second being the only use case native to our product, since
[section 4](04-editing.md) ships attribution with no venue for discussion. At the scale
[1.4](01-product.md) predicts, the message table holds dozens of rows.

**Decision: the module is built, after everything in [1.9](01-product.md), as a plain mailbox — and
the honest reason is the portfolio, not user demand.** In the same spirit as [1.3](01-product.md)'s
caveat: a mailbox exercises a technical shape the catalogue vertical does not (threading, per-user
read state, notification, abuse controls), and that is worth building for
[1.1](01-product.md)'s reader. It is not worth pretending users are queueing for it. Everything
below is sized by that: the smallest thing that is genuinely a mailbox, and a refusal of everything
messenger-shaped.

- [x] 5.1 1:1 only, or group chats too?
      **Decision: 1:1 only, permanently.** Group chat brings participant management (add, remove,
      leave, rejoin, and who may do each), per-participant read state that diverges, and a name where
      a subject was. At tens of users ([1.4](01-product.md)) there is no group to have.
      **Modelled with a `conversation_participant` row per user regardless** — not as a hedge against
      groups, but because that row is the only sensible home for per-user state (`last_read_at` in
      [5.6](05-messaging.md), `hidden_at` in [5.8](05-messaging.md)). Exactly two rows per
      conversation, enforced in the service layer ([4.7](04-editing.md)). Group is therefore additive
      if the premise ever changes, but it is not designed for and no code may assume more than two.
- [x] 5.2 Model: a mailbox (subject + thread, as on Discogs) or a messenger (real-time message feed)?
      **Decision: a mailbox.** A conversation is a subject plus an ordered list of messages; the
      subject is set by whoever opens it and is immutable afterwards. **Per subject, not per pair** —
      the same two people may hold two separate conversations about two different releases, and the
      use case above is always "a question about a thing" rather than an open-ended channel.
      Confirms recommendation 5. The secondary argument matters more than it looks: an empty mailbox
      reads as normal, an empty messenger reads as broken, and [1.4](01-product.md) guarantees this
      thing is mostly empty.
- [x] 5.3 Real time: WebSocket/SSE, or is polling + email notification enough?
      **Decision: no real-time transport at all** — no WebSocket, no SSE, and no client polling
      loop either. State is whatever the page render shows; timeliness is the email's job
      ([5.9](05-messaging.md)). This follows directly from [1.4](01-product.md)'s target shape —
      one process on one small box, deploys that may take the site down for ten seconds — which is
      precisely the shape that cannot hold long-lived connections. SSE over the same process is
      additive if it is ever wanted; nothing in the model presumes its absence.
- [x] 5.4 Attachments: images? files? links to catalogue releases (rich previews)?
      **Decision: no attachments of any kind, ever, in this module.** [1.4](01-product.md) already
      names images as the item that breaks the size estimate first, and private user-to-user
      attachments are its worst case: unbounded, unmoderatable, invisible to everyone but two people,
      and outside the eight-image cap that [2.4.8](02-catalogue-model.md) puts on the part of the
      system we do host images for.
      Links are text: URLs in the body are linkified at render ([5.5](05-messaging.md)), and a link
      to one of our own pages is an ordinary link. **Rich previews are deferred, not refused** —
      rendering a release link as title, artist and thumbnail is one query at render time and adds no
      column.
- [x] 5.5 Formatting: plain text / markdown / restricted HTML. Sanitisation.
      **Decision: plain text.** Stored verbatim, escaped at render, newlines preserved, URLs
      linkified (external ones with `rel="noopener nofollow ugc"`). No markdown, no HTML, no BBCode.
      Restricted HTML means owning a sanitiser and a standing XSS liability
      ([section 14](14-security.md)) for a feature with dozens of rows; markdown means a parser
      *plus* that sanitiser. The asymmetry decides it: plain text upgrades cleanly to markdown later,
      because every existing message is still valid input — the reverse migration does not exist.
- [x] 5.6 Read receipts, "typing…", unread counter.
      **Decision: unread counter yes; the other two no.** The counter is `last_read_at` on the
      participant row plus a count query — a mailbox without it does not work at all, and it is the
      only piece of state the inbox listing needs.
      "Typing…" requires the real-time channel [5.3](05-messaging.md) refused, so it is not a
      separate decision. **Read receipts** — telling the sender their message was read — are declined
      as social pressure dressed as a feature, and as a privacy choice we would rather not make on a
      user's behalf.
      Note the asymmetry [4.9](04-editing.md) established and this reuses: **the data is stored, the
      feature is not built.** `last_read_at` exists from day one, so receipts are a later UI change
      rather than a migration.
- [x] 5.7 Blocking users, reporting messages, anti-spam (limit on new conversations for newcomers).
      **Decision: three mechanisms, in descending order of how much they matter.**
      1. **Blocking, built with the module.** `user_block(blocker, blocked, created_at)`. A blocked
         user may neither open a conversation with the blocker nor post into an existing one,
         enforced in the service layer ([4.7](04-editing.md)) so that the future public API
         ([10.1](10a-public-api.md)) obeys the same rule rather than re-implementing it. It is the
         only abuse control that works with no moderator on duty, which is our situation by
         construction.
         **The refusal is explicit** ("You can't message this user"), not a silent success. Silent
         failure is the standard defence against harassment at scale; we do not have that scale
         ([1.4](01-product.md)), and at ours, lying to a user about whether their message was
         delivered is the larger harm.
      2. **Reporting writes a row and emails the admin.** No queue, no triage UI — consistent with
         [1.9](01-product.md) keeping any moderation queue out of scope.
      3. **Anti-spam is a rate limit on top of the verified-email wall, and nothing else.**
         [4.9](04-editing.md) already chose verified email as the barrier and turned down captcha and
         honeypots as cost against an imaginary threat; that reasoning is unchanged here. On top of
         it: a cap on new conversations per user per day and on messages per hour, in the service
         layer. Feeds [14.5](14-security.md), which should not invent a second answer.
- [x] 5.8 Message deletion: for me / for everyone / never. Storage and privacy (we are not planning encryption?).
      **Decision: delete for me only, and no encryption — stated plainly rather than implied.**
      Hiding a conversation sets `hidden_at` on the hider's participant row. Nothing is removed from
      the other person's mailbox: their copy is their data, and a sender who could unsend has a
      retroactive veto over someone else's record. No delete-for-everyone, no edit-after-send.
      Otherwise messages are retained indefinitely.
      **Message bodies are plaintext in Postgres**, readable by whoever holds the database — which is
      the author of this project. End-to-end encryption is meaningless when the same party renders
      the pages, and claiming a privacy property we do not provide is worse than saying this out
      loud. [Section 13](13-legal.md)'s privacy policy states it; [section 14](14-security.md) covers
      it as an at-rest and backup concern like any other table.
      **This hands [6.5](06-accounts.md) a question it must answer, not one we settle here:** when an
      account is deleted, its messages have to survive in the other party's mailbox, attributed to a
      tombstone user. Erasure of one party's data cannot erase the other party's record of a
      conversation they took part in, and that tension is [6.5](06-accounts.md) and
      [13.3](13-legal.md)'s to resolve.
- [x] 5.9 Notifications: in-app only / email / push (web push). User notification settings.
      **Decision: in-app unread counter plus email, with one throttle rule and one setting.**
      - Email on a new message, **at most one per conversation until the recipient next reads it**.
        Without that rule one lively thread sends twenty emails and trains the user to filter us.
      - **One global on/off** in profile settings. No per-conversation muting, no digest, no
        frequency picker — each is a preference row and a code path serving a population of tens.
      - **No web push.** A service worker, VAPID keys, a permission prompt and a per-browser
        subscription table is a vertical of its own, aimed at users who are not sitting in the app —
        and at our volume nobody is sitting in the app.
      Costs no new infrastructure: [section 6](06-accounts.md) already needs a mail sender for
      verification, and this is a second template on it.
- [x] 5.10 Search across messages — needed?
      **Decision: not built.** At the volume [1.4](01-product.md) predicts, the mailbox page and the
      browser's own find covers it. Postgres FTS is already in the stack for
      [section 7](07-search-ux.md), so if it is ever wanted it is one index and one query over a
      table that needs no schema change to support it. Deferred, not refused — and cheaply so.
- [x] 5.11 Are there other social mechanics: comments on releases, reviews, following users, activity feed, forum? (or are messages the only social feature)
      **Decision: messaging is the only social feature, at any stage.** No comments on releases, no
      reviews or ratings, no following, no activity feed, no forum.
      - **Comments on releases** are the only real contender, and they lose to the model: a comment
        saying "this pressing has a different runout" is *a fact that failed to become an edit*.
        [Section 4](04-editing.md) makes editing attributable and reversible precisely so that
        observations land in the catalogue instead of in a thread beside it, and a comment box is the
        path of least resistance away from the thing [1.3](01-product.md) names as our value.
        [2.4.7](02-catalogue-model.md) turned down a data-quality field on the same principle: the
        signal should be data, not an assertion parked next to it.
      - **Reviews and ratings** are RateYourMusic's product. An aggregate score over ten users is
        noise, and [1.3](01-product.md) does not claim opinion as a differentiator.
      - **Following and an activity feed** need a population and a rate of activity we will not have;
        [4.2](04-editing.md) already deferred the change feed for exactly that reason, and a feed of
        one person's edits is a page nobody opens twice.
      - **A forum** is a second product, and the largest generator of moderation load available to
        us — the load [1.9](01-product.md) declined to take on even for the catalogue.
      All of these are additive if the premise ever changes. None are designed for, and no schema
      accommodates them in advance.

## Working notes

- **2026-08-15 — Schema sketch implied by the decisions above.** Five tables, no new infrastructure:
  `conversation` (subject, created_at, created_by), `conversation_participant` (conversation, user,
  `last_read_at`, `hidden_at` — exactly two rows), `message` (conversation, sender, body, sent_at),
  `user_block` (blocker, blocked, created_at), `message_report` (message, reporter, reason,
  created_at). No attachment table ([5.4](05-messaging.md)), no delivery/receipt table
  ([5.6](05-messaging.md)), no subscription table ([5.9](05-messaging.md)).
- **2026-08-15 — What this section contributes to [11.2](11-stack.md).** It was the last section the
  stack choice was waiting on, and its answer is: nothing new is required. No real-time transport
  ([5.3](05-messaging.md)), no file storage ([5.4](05-messaging.md)), no search backend
  ([5.10](05-messaging.md)), no push service ([5.9](05-messaging.md)) — five ordinary tables and one
  outbound email template on a mail sender [section 6](06-accounts.md) already needs. The stack may
  be chosen on the catalogue's merits alone.
- **2026-08-15 — Where this lands in the roadmap.** After the whole of [1.9](01-product.md)'s MVP.
  Within the module the order is: conversation and message → unread counter → blocking →
  email notification → reporting. Blocking before notification is deliberate — an email that a
  blocked user's message has arrived is the one failure worth avoiding.

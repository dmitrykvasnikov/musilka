# E2 — Messaging

**Size:** 3–4 weeks ([15.1](../design/15-roadmap.md)) · **Tasks:** T-161 … T-173

**Why this is the next vertical:** [section 5](../design/05-messaging.md) needs **nothing new from
the platform** — five ordinary tables and one email template on a mail sender E1.1 already built. No
real-time transport, no storage, no search backend, no push service. It is the cheapest vertical
available after the round trip works.

**And the honest reason it exists at all is the portfolio, not user demand.** At the scale
[1.4](../design/01-product.md) predicts, the message table holds dozens of rows;
[1.9](../design/01-product.md) put messaging outside the MVP precisely because at tens of users
there is nobody to message. What it buys is a technical shape the catalogue vertical does not
exercise — threading, per-user read state, notification, abuse controls. Everything below is sized
by that: the smallest thing that is genuinely a mailbox, and a refusal of everything
messenger-shaped.

**The order within the module is [5](../design/05-messaging.md)'s working note's, and one step of it
is deliberate:** conversation and message → unread counter → **blocking** → email notification →
reporting. Blocking comes before notification because an email announcing that a blocked user's
message has arrived is the one failure worth avoiding.

- [ ] T-161 Migration 011: the five tables (per 5.1, 5.2, 5.6, 5.7, 5.8)
      `conversation(id, subject, created_at, created_by)`;
      `conversation_participant(conversation_id, user_id, last_read_at, hidden_at)` — **exactly two
      rows, enforced in the service layer**; `message(id, conversation_id, sender_id, body,
      sent_at)`; `user_block(blocker, blocked, created_at)`;
      `message_report(message_id, reporter_id, reason, created_at)`.
      No attachment table, no delivery or receipt table, no subscription table.
      The participant row is not a hedge against group chat — it is the only sensible home for
      per-user state, and `last_read_at` and `hidden_at` are why it exists.
- [ ] T-162 Conversation and message (per 5.1, 5.2)
      **A mailbox, not a messenger.** A conversation is a subject plus an ordered list of messages;
      the subject is set by whoever opens it and is **immutable afterwards**. **Per subject, not per
      pair** — the same two people may hold two conversations about two different releases, and the
      use case is always "a question about a thing" rather than an open-ended channel.
      An empty mailbox reads as normal; an empty messenger reads as broken, and
      [1.4](../design/01-product.md) guarantees this thing is mostly empty.
      1:1 only, permanently. No code may assume more than two participants.
- [ ] T-163 Inbox and conversation screens (per 5.2, 7.5)
      Two templates. Not on [7.5](../design/07-search-ux.md)'s MVP list, by design — they arrive
      here.
- [ ] T-164 Message bodies (per 5.5, 14.3)
      **Plain text**, stored verbatim, rendered through T-44's linkifier — the same function as
      profile bios, not a second policy. No markdown, no HTML, no BBCode: restricted HTML means
      owning a sanitiser and a standing XSS liability for a table with dozens of rows.
      Plain text upgrades cleanly to markdown later, because every existing message is still valid
      input; the reverse migration does not exist.
      **No attachments of any kind, ever** — private user-to-user files are the unmoderatable worst
      case of the storage problem [1.4](../design/01-product.md) already flags. Links are text; rich
      previews are deferred (one query at render time, no column), and a URL is **never resolved**.
- [ ] T-165 Unread counter (per 5.6)
      `last_read_at` on the participant row plus a count query. A mailbox without it does not work at
      all, and it is the only piece of state the inbox listing needs.
      **Read receipts are not built, and `last_read_at` is stored anyway** — the same asymmetry as
      [4.9](../design/04-editing.md)'s mass revert: store the data, skip the tool. Receipts are
      social pressure and a privacy choice we decline to make on the user's behalf; "typing…"
      requires the real-time channel [5.3](../design/05-messaging.md) refused, so it is not a
      separate decision.
- [ ] T-166 Hide a conversation (per 5.8)
      `hidden_at` on the hider's participant row. **Nothing is removed from the other person's
      mailbox** — their copy is their data, and a sender who could unsend would have a retroactive
      veto over someone else's record. No delete-for-everyone, no edit-after-send. Messages are
      otherwise retained indefinitely.
- [ ] T-167 Blocking (per 5.7)
      A blocked user may neither open a conversation with the blocker nor post into an existing one,
      enforced in the service layer. **It is the only abuse control that works with no moderator on
      duty**, which is our situation by construction.
      **The refusal is explicit** — "You can't message this user" — not a silent success. Silent
      failure is the standard defence at scale; we do not have that scale, and at ours, lying to a
      user about whether their message was delivered is the larger harm.
- [ ] T-168 Email notification (per 5.9, 6.6)
      A fifth use of T-38's sender and its fourth template. **At most one email per conversation
      until the recipient next reads it** — without that rule one lively thread sends twenty emails
      and trains the user to filter us. One global on/off, already a column from T-46
      (`notify_new_message`, default on).
      No per-conversation muting, no digest, no frequency picker, no web push — each is a preference
      row and a code path serving a population of tens.
- [ ] T-169 Message rate limits (per 5.7, 14.5)
      Rows in T-49's table: 20 messages/hour and 100/day per sender, and **5 first-messages to a new
      recipient per day**. That last one is the number that matters —
      [5.7](../design/05-messaging.md) identified unsolicited bulk messaging as the only real abuse
      vector here, and five new conversations a day is generous for a population of tens.
      Anti-spam is a rate limit on top of the verified-email wall and **nothing else**: no captcha,
      no honeypot.
- [ ] T-170 Reporting a message (per 5.7, 4.6)
      Writes a `message_report` row and **emails the admin**. No queue, no triage UI — the action
      behind the email is T-48's `disabled_at`, which is an `UPDATE` a human runs.
      The operator can read a reported body, because [5.8](../design/05-messaging.md) makes that
      possible and [13.3](../design/13-legal.md) discloses it.
- [ ] T-171 Deletion tombstone, exercised for the first time (per 6.5, 5.8)
      T-47 already anonymises the account; this is where it is actually tested. **Conversations,
      participant rows and message bodies survive**, and the sender renders as `[deleted]`. It
      resolves the same way an email you received does: the sender leaving does not unsend it, and
      the other party's mailbox is *their* record of a conversation they took part in.
      This is why [6.5](../design/06-accounts.md) could not hard-delete the user row: two participant
      rows are required per conversation, and a missing row would dangle them.
- [ ] T-172 Account dump gains its conversations section (per 10.3.1, 10.3.5)
      Extend T-104. Every conversation the user takes part in, **both sides' bodies verbatim**. A
      partner appears as a nickname and **never as an email address**; a deleted partner appears as
      the tombstone, not as the nickname they used to hold.
- [ ] T-173 Messaging integration tests (per 11.10, 5.1, 5.7, 14.3)
      Level-2, against a real Postgres: exactly two participants enforced; a blocked sender refused
      explicitly; another pair's conversation refused **in the query, not after the load** — this is
      the IDOR case [14.1](../design/14-security.md) says an actual user will try.

## Working notes

**2026-08-15 — what E2 does not add, and it is the whole argument for building it here.** No
WebSocket, no SSE, no polling loop, no service worker, no file storage, no search index, no new
infrastructure of any kind. If a task in this stage starts to need one, it has drifted out of
[section 5](../design/05-messaging.md) and into a messenger.

**2026-08-15 — message search is deferred, cheaply** ([5.10](../design/05-messaging.md)). At this
volume the mailbox page and the browser's own find covers it, and Postgres FTS is already in the
stack — if it is ever wanted it is one index and one query over a table needing no schema change.

**2026-08-15 — messaging is the only social feature, at any stage.** No comments on releases, no
reviews or ratings, no following, no activity feed, no forum. The reason generalises past this stage
and should be cited whenever a discussion venue is proposed: **a comment is a fact that failed to
become an edit**, and [section 4](../design/04-editing.md) exists so that observations land in the
catalogue rather than in a thread beside it.

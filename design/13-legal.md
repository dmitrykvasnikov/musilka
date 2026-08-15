# 13. Legal and organisational

**Priority:** P1

**What this section is not free to decide.** It inherits more than it chooses.
[6.5](06-accounts.md) fixed what deletion erases and what it keeps, and named three statements this
section may not overstate. [5.8](05-messaging.md) made message bodies plaintext and put the
obligation to *say so* here. [9.2](09-nfr.md) refused to promise availability and offered durability
instead. [9.3](09-nfr.md) put backups in a bucket whose jurisdiction becomes a sentence on the
privacy page. [1.5](01-product.md) means we ingest nothing from anyone's database except through a
user's own file, which is why the licensing question is small. What was genuinely open here was the
data licence, the code licence, the jurisdiction, and the brand.

- [x] 13.1 Licence for the catalogue data: do we make the data open (CC0) like MusicBrainz/Discogs? What do we put in the ToS about user contributions. (see also [10.1.11](10a-public-api.md), [10.3.4](10c-export.md))
      **Decision: CC0 1.0 for the catalogue data, images explicitly excluded, and a one-line grant in
      the ToS rather than a CLA.**
      **What is CC0.** The catalogue and only the catalogue: masters, releases, artists, labels,
      credits, identifiers, the seeded vocabularies, and the relations between them — the entities
      [section 2](02-catalogue-model.md) defines. This is the database plus the facts in it, and it
      matches where the rows came from: Discogs' dumps are CC0 and MusicBrainz' core data is CC0, so
      a user's export ([10.2](10b-import.md)) carries public-domain material into a public-domain
      catalogue. Enclosing what we received freely would be both ungracious and hard to justify,
      and [1.1](01-product.md) already set commercialisation aside, so the licence costs us nothing
      we were going to use.
      **What is not CC0, and this is the important half:**
      - **Images.** Uploaded sleeve scans are third-party artwork ([8.4](08-media.md) exists
        precisely because we hold no rights to them). We cannot dedicate to the public domain what
        is not ours. The bucket is served under no licence claim at all, and any dump
        ([10.3.4](10c-export.md)) carries image *keys*, never image bytes.
      - **User data.** Collections, wantlists, notes, tags, private messages and profiles are not
        catalogue data and are never published under any licence. A user's export
        ([10.3](10c-export.md)) is their own file and carries no licence claim from us.
      - **The code**, which is [13.2](13-legal.md)'s separate answer.
      **The ToS grant.** One clause, accepted at registration: *contributions to the catalogue are
      dedicated to the public domain under CC0 1.0, including any sui generis database right, and
      the contributor confirms they are entitled to do so.* No CLA, no signature flow, no separate
      contributor agreement — those exist to manage a population of contributors we do not have
      ([1.4](01-product.md)), and the whole point of choosing CC0 over a share-alike licence is that
      downstream nobody has to think about compliance either. Most of what a contributor types is a
      fact about a physical object and not copyrightable in the first place; the clause exists for
      the residue and for the database right.
      **Where it is stated:** a `/licence` page linked from the footer, the same text in the ToS,
      and a `LICENSE` file inside any published dump. Not in a user's own export — see above.
      **What this unblocks.** [10.3.4](10c-export.md)'s public catalogue dump was blocked on this
      item in one direction: publishing *is* the licensing act. The licence now exists, so the dump
      is a scheduling question rather than a legal one. Two cautions carried forward: we can only
      dedicate what we hold, and NOTES' EU database-rights constraint was accepted for *ingesting*
      one user's export, never for redistributing the accretion — a dump at any real volume is the
      point where that risk stops being theoretical and wants an actual lawyer.
- [x] 13.2 Licence for the project code (open source or not).
      **Decision: MIT, one `LICENSE` file covering the whole repository, public from the first
      commit.**
      **MIT** because the job of the licence here is to remove thinking from a stranger reading the
      code — which is what [1.1](01-product.md) says the repository is for. Apache-2.0's patent
      grant protects against a risk we do not carry; AGPL protects against a commercial fork that
      [1.1](01-product.md) says we do not expect and would arguably welcome as flattery.
      **The whole repository, including `design/`.** These documents are the most interesting
      artifact in the project and they travel under the same licence as the code, because a second
      licence means a per-file question and a header nobody maintains.
      **Public from the start, not "public once it is good".** The history is part of what the
      repository demonstrates, and a repository made public later is a repository whose early
      history nobody reads.
      **The cost of a public repository is exactly one discipline:** no secret ever enters it. Not
      the database URL, not the mail credentials, not the object-storage keys, not [9.3](09-nfr.md)'s
      backup *private* key (its public key is fine and may be committed). Configuration comes from
      the environment, and [section 12](12-infrastructure.md) owns how. A leaked secret in git
      history is not fixed by a later commit.
      **No contributor process.** No CLA (nothing to sign for MIT), no issue templates, no
      contribution guide, no promise to review a pull request. One developer
      ([1.4](01-product.md)); pretending otherwise creates an obligation we would not meet.
- [x] 13.3 Terms of service and privacy policy; GDPR/152-FZ: where personal data is stored, export and deletion of a user's data. (see also [6.5](06-accounts.md), [10.3.1](10c-export.md), [10.4.8](10d-model-requirements.md))
      **Decision: one privacy policy and one ToS written to the GDPR, with the box and both buckets
      in the EU/EEA. 152-FZ is not claimed and its localisation requirement is therefore not
      inherited.**
      **The jurisdiction choice is an infrastructure constraint, and that is why it is here.**
      152-FZ Art. 18.5 would require the primary database recording Russian users' personal data to
      sit in Russia, which would decide [12.1](12-infrastructure.md) for it and drag Roskomnadzor
      notification along behind. Writing to the GDPR instead costs one policy page and constrains
      [12.1](12-infrastructure.md) to an EU/EEA provider — which also removes the international
      transfer question entirely, since [9.3](09-nfr.md)'s backup bucket and
      [8.2](08-media.md)'s image bucket are then in the same jurisdiction as the database. **This
      is now binding on [section 12](12-infrastructure.md): VPS, image bucket, backup bucket and
      mail sender are all EU/EEA.**
      **The exhaustive list of personal data we hold**, because a policy that is not exhaustive is
      worse than none:

      | Data | Where | Kept for |
      |---|---|---|
      | Email address | `user` | Life of the account ([6.5](06-accounts.md) clears it on deletion) |
      | Nickname, bio | `user` | Same |
      | Password hash | `user` ([14.2](14-security.md)) | Same |
      | Sessions | `session` ([11.6](11-stack.md)) | Until expiry or logout |
      | Collection, wantlist, tags, notes | Own tables | Hard-deleted with the account ([6.5](06-accounts.md)) |
      | Private message bodies | `message` ([5.8](05-messaging.md)) | Forever, and they **survive the sender's deletion** |
      | Catalogue edits | `revision` ([4.2](04-editing.md)) | Forever, re-attributed to a tombstone |
      | Uploaded import file | `import.file_bytes` ([10.2.6](10b-import.md)) | Until the job ends |
      | IP address, request log | journald ([9.5](09-nfr.md), [14.6](14-security.md)) | 30 days |
      | Everything above, in a backup | [9.3](09-nfr.md)'s bucket | Up to ~90 days after deletion |

      **The three statements [6.5](06-accounts.md) forbids us to overstate, written the way they
      must appear:**
      1. *Deleting your account empties it rather than removing the row.* Email, nickname and bio
         are cleared; collection, wantlist, tags and notes are deleted outright; messages you sent
         and catalogue edits you made remain, attributed to a deleted user. Immediate and
         irreversible — [6.5](06-accounts.md) refused an undo window, and the deletion page offers
         [10.3](10c-export.md)'s export beside the button.
      2. *Private messages are private from other users, not from us.* They are stored as plain
         text in our database, readable by whoever operates the service, and they survive the
         sender's account deletion because the recipient's copy of a conversation is theirs.
         [5.8](05-messaging.md) chose this deliberately; the policy says it in those words and
         never implies encryption.
      3. *The only profile data we hold is an address, a nickname and an optional bio.* No real
         name, no birthday, no gender, no phone, no links table — [6.3](06-accounts.md) declined
         all of them, and this row of the table is short because of that decision.
      **Lawful bases**, which are unusually simple here: performance of a contract for the account
      and its features; legitimate interest for [14.6](14-security.md)'s logs and
      [14.5](14-security.md)'s abuse limits. **No consent basis anywhere**, because
      [13.4](13-legal.md) has nothing to consent to — no analytics, no marketing, no profiling, no
      automated decision-making. No special-category data, so no DPIA and no DPO.
      **Rights, and the honest thing is that most are already self-service:** access and portability
      are [10.3](10c-export.md), a button that runs in under a second; erasure is
      [6.5](06-accounts.md), with the caveats above stated on the same page; rectification is
      editing your profile. Objection and complaint go to a published address, plus the standard
      pointer to a supervisory authority.
      **Backups are the one place a deletion does not reach**, and the policy says so with the
      number: a dump taken before you left holds your data until it ages out of
      [9.3](09-nfr.md)'s 7/4/3 retention, up to about 90 days. We do not restore a backup to
      recover a deleted account.
      **Sub-processors are named, not gestured at:** the VPS provider, the object-storage provider
      and the mail sender, each by name and country. [12.1](12-infrastructure.md) fills in the
      names; the policy has a slot for them. There is nothing else — [9.5](09-nfr.md) refused
      Sentry and [13.4](13-legal.md) refuses analytics, so the list is three lines and stays that
      way.
      **The ToS is short and says five things:** you must be 16 ([13.5](13-legal.md)); your
      catalogue contributions are CC0 ([13.1](13-legal.md)); we may edit, merge or remove any
      catalogue contribution ([section 4](04-editing.md)) and may suspend an account for abuse; the
      service is provided as-is with **no availability promise** ([9.2](09-nfr.md) — what we do
      promise instead is durability, no data loss beyond 24 hours, and that is the sentence worth
      putting in front of users); and the governing law is the operator's country of residence.
      **Two blanks remain, and both are filled by facts we do not have yet rather than decisions we
      have not made:** the controller's name and address, and the sub-processor names from
      [12.1](12-infrastructure.md). **Both documents are written before the first real user, not
      now** — [section 15](15-roadmap.md) carries the task, and this item is the specification for
      what they must contain.
- [x] 13.4 Cookie banner and analytics (which one, and whether we need it at all).
      **Decision: no analytics of any kind, one cookie, and therefore no banner — and the absence of
      a banner is a property to defend rather than an omission.**
      **One cookie, and it is strictly necessary.** The session cookie for
      [11.6](11-stack.md)'s session table: `HttpOnly`, `Secure`, `SameSite=Lax`, no other cookie
      anywhere. [14.3](14-security.md)'s CSRF token lives in the session row, not in a second
      cookie. A cookie that exists solely to keep a logged-in user logged in is exempt from consent
      under ePrivacy, so there is nothing to ask and **no banner is shown**. The banner is not
      being skipped as a shortcut: it would be a lie, because it would consent to tracking that
      does not exist.
      **No third-party analytics.** Google Analytics and its relatives would each bring a consent
      banner, a transfer question for [13.3](13-legal.md), a sub-processor line, an entry in
      [14.3](14-security.md)'s CSP and a script tag in a product that has none — to count visits to
      a site with ten users.
      **No self-hosted analytics either**, and this is the more tempting refusal. Plausible, Matomo
      and GoatCounter each need a second daemon and usually a second database on a 2 GB box, which
      is [1.4](01-product.md)'s veto list by name.
      **What we have instead costs nothing because it already exists:** [9.5](09-nfr.md)'s
      one-line-per-request log, 30 days of it. If a question ever genuinely needs answering — which
      pages are hit, whether search returns nothing — it is `goaccess` or `awk` run offline over
      journald, on data kept for [14.6](14-security.md)'s reasons anyway. No pixel, no page weight,
      no third party, no consent.
      **The consequence is a general property**, recorded because it is easy to spend by accident:
      **the browser talks to our origin and the image bucket, and to nothing else, ever.** No fonts,
      no CDN, no CAPTCHA ([14.5](14-security.md)), no error tracker ([9.5](09-nfr.md)), no
      analytics. That is what makes [14.3](14-security.md)'s strict CSP free, and it is what keeps
      [13.3](13-legal.md)'s sub-processor list at three lines.
- [x] 13.5 Age restrictions, UGC moderation, procedure for responding to rights-holder complaints.
      **Decision: 16+ stated and unverified, moderation reactive through the two levers we already
      built, and a published abuse address with a documented takedown path — no DMCA agent, no DSA
      machinery.**
      **Age: 16, by a checkbox.** Stated in the ToS, never verified — the alternative is an
      identity check, which would collect more personal data than the entire rest of the service
      ([13.3](13-legal.md)'s table) to protect a catalogue of records. 16 is the GDPR's default
      digital-consent age, which avoids tracking member-state variations, and we have no reason to
      want younger accounts: [6.1](06-accounts.md)'s barrier already requires an email address the
      user controls.
      **Not an adult service, and no 18+ gate.** Sleeve art is routinely nude and that is normal
      catalogue content — [8.4](08-media.md) refused automated screening partly because our content
      is exactly where a classifier's false-positive rate is worst. Pornographic or shock imagery
      that is *not* the object being catalogued is abuse and is removed as such. That distinction
      is a human judgement and there is a human ([1.4](01-product.md)).
      **Moderation is reactive, and there are exactly two levers, both already designed:**
      [4.6](04-editing.md)'s report row for catalogue content, and [8.4](08-media.md)'s image
      takedown with its `blocked_at` on the `sha256` (which turns content-addressed storage into
      free refusal of a re-upload). Moderators are [4.7](04-editing.md)'s role. For messages,
      [5.7](05-messaging.md)'s block and report — and the operator can read a reported body,
      because [5.8](05-messaging.md) makes that possible and [13.3](13-legal.md) discloses it. No
      proactive review queue, no automated filter, no trusted-flagger programme: each presumes a
      volume and a staff we do not have.
      **Rights-holder complaints, and the asymmetry that runs through them:**
      - **Images get removed.** A complaint about cover art is answered by [8.4](08-media.md)'s
        hard delete — the one hard delete in the design — because a tombstone that still serves the
        file answers nothing. Removal is not an admission; we never claimed rights in the artwork
        ([13.1](13-legal.md) excludes it from CC0).
      - **Catalogue facts do not.** "This release has twelve tracks" is not anyone's property, and
        a complaint about a tracklist gets a reply rather than a deletion. If a claim is ever made
        against the *database* rather than its contents, that is the EU database-rights question
        NOTES records as accepted-with-risk, and it needs a lawyer rather than a policy.
      **The procedure, in full:** a published address (`abuse@`, once [13.6](13-legal.md)'s domain
      exists) on a `/contact` page and in the ToS; acknowledgement within a few days; content
      removed or the claim declined with a reason; a note kept of what was removed and why, which
      for images is the `blocked_at` row we already store. **No registered DMCA agent** — that is a
      US safe-harbour formality and [13.3](13-legal.md) put us in the EU — and **no DSA compliance
      machinery**: statements of reasons, transparency reports and appeal mechanisms are scoped to
      services far larger than this one. A working contact address and a takedown that actually
      happens is the substance; the rest is procedure for a company. *(Not verified against current
      law — recorded in NOTES.)*
- [x] 13.6 The name/brand "Musilka", the domain, the logo — do we lock these in?
      **Decision: the name is locked, the domain is deliberately deferred, and the logo is a
      wordmark — with the deferral's cost written down so it is not discovered on deploy day.**
      **"Musilka" is settled.** It is in the repository name, in every design document and in the
      UI. Renaming is cheap today and expensive in a month; there is nothing left to gain by
      leaving it open.
      **The domain is deferred to [12.1](12-infrastructure.md)**, and this is the item's one real
      trade. What hangs on it: the TLS certificate, and — the part that bites — the transactional
      mail that [section 6](06-accounts.md) requires for four templates. A fresh domain sending
      verification mail with no SPF, DKIM or DMARC records is a domain whose mail lands in spam,
      which breaks [6.1](06-accounts.md)'s verified-email barrier, on which every anti-abuse
      decision in the design leans. **So the rule attached to the deferral:** the domain is
      registered and its mail records published in the *same sitting* as the first deploy setup, not
      after it, and reputation is given time before a stranger is invited ([1.10](01-product.md)).
      [section 15](15-roadmap.md) carries it as a task on the critical path, and NOTES carries it as
      an open item.
      **The logo is a text wordmark** in the site header, set in a system font
      ([9.1](09-nfr.md)'s payload budget has no room for a webfont and does not want one), plus a
      single-letter favicon. No logo file to commission, no brand guide.
      **No trademark search and no registration**, and no defensive domain purchases. Portfolio
      scale ([1.1](01-product.md)): if the name ever collides with a real business, renaming a
      hobby project is cheaper than the search would have been.

## Working notes

- **2026-08-15 — Section closed. All 6 items decided.** Four were genuinely open and were decided by
  the operator: CC0 for the data ([13.1](13-legal.md)), MIT for the code ([13.2](13-legal.md)),
  GDPR with EU hosting ([13.3](13-legal.md)), and the name locked with the domain deferred
  ([13.6](13-legal.md)). The other two are readings of decisions already made elsewhere —
  [13.4](13-legal.md) follows from there being no third party anywhere in the product, and
  [13.5](13-legal.md) is [4.6](04-editing.md) and [8.4](08-media.md) with a contact address.
- **2026-08-15 — What this section hands on.**
  [Section 12](12-infrastructure.md) inherits a hard constraint — **VPS, image bucket, backup
  bucket and mail sender all in the EU/EEA** ([13.3](13-legal.md)) — plus the domain registration
  and its SPF/DKIM/DMARC records as a first-deploy task ([13.6](13-legal.md)), and the requirement
  that every secret stays out of a public repository ([13.2](13-legal.md)).
  [Section 15](15-roadmap.md) inherits four documents to write before the first real user: privacy
  policy, ToS, `/licence` and `/contact`. None is long; all four are easy to leave until after
  launch, which is why they are named here.
  [10.3.4](10c-export.md) is **unblocked** — the data licence exists, so a public dump is a
  scheduling decision now.
- **2026-08-15 — The two blanks, and why they are not open questions.** The controller's name and
  address, and the sub-processor names, are facts we will have once [12.1](12-infrastructure.md)
  picks providers. Nothing in the design waits on them.
- **2026-08-15 — What was declined.** 152-FZ and Russian hosting (and with it Roskomnadzor
  notification); CC BY-SA and any share-alike for the data; Apache-2.0, AGPL and a private
  repository for the code; a CLA; analytics of every kind including self-hosted; a cookie banner; a
  registered DMCA agent and DSA compliance machinery; age verification; a trademark search and
  registration; a designed logo.

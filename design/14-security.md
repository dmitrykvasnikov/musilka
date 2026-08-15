# 14. Security

**Priority:** P1

**What this section is not free to decide.** Most of the surface was closed by earlier sections and
this one writes down the consequences. [1.5](01-product.md) forbids the server calling anything
outward, which deletes SSRF as a category. [11.3](11-stack.md) has no build step and no JavaScript
of our own, which deletes a dependency tree in the browser — and creates one obligation, since plain
HTML forms mean nothing adds a CSRF token for you. [11.5](11-stack.md)'s service layer is where
authorisation lives, so [4.7](04-editing.md)'s roles, [6.1](06-accounts.md)'s verified-email barrier
and [3.7](03-collection.md)'s privacy hold identically in the UI and in any future API. NOTES' *one
text-rendering policy for all user prose* names [14.3](14-security.md) as its reason. [9.4](09-nfr.md)
took the crude per-IP layer and handed the per-identity table to [14.5](14-security.md). What was
genuinely open here is the password answer, the rate-limit numbers, and the exact log-line
prohibitions.

- [x] 14.1 Threat model in a nutshell: what we protect (accounts, private collections, DMs, catalogue integrity).
      **Decision: the assets are ranked, the adversary we actually have is a bot, and the things we
      do not defend against are named rather than implied.**
      **What we protect, in order of how much damage its loss does:**
      1. **Private message bodies and email addresses.** This is the whole of the sensitive data in
         the system, it is plaintext in Postgres by decision ([5.8](05-messaging.md)), and it
         survives account deletion ([6.5](06-accounts.md)). Everything below matters mostly because
         it is a route to this.
      2. **Account control** — the password and the session. A stolen account reads a mailbox and
         writes edits.
      3. **Private collections and wantlists** ([3.7](03-collection.md)), including the owner-only
         fields (note, purchase date and place) that [3.7](03-collection.md) made unconditional
         rather than per-item.
      4. **Catalogue integrity**, ranked last on purpose: [4.2](04-editing.md) stores every
         revision, [4.4](04-editing.md)'s merge copies nothing and reverses cleanly, and
         [4.5](04-editing.md)'s delete is a tombstone. Vandalism here is a chore, not a loss —
         which is exactly what [4.9](04-editing.md) leaned on when it declined captcha.
      5. **Availability**, which [9.2](09-nfr.md) already declined to promise.
      **Who is actually attacking, honestly:**
      - **A scanner.** Constant, automated, uninterested in us specifically, probing for
        WordPress paths, `.env` files and known CVEs. Handled by [9.4](09-nfr.md)'s proxy caps, by
        not running the software it is looking for, and by patching.
      - **A spam registration script**, after a page to put links on. Handled by
        [6.1](06-accounts.md)'s verified-email barrier plus [14.5](14-security.md), and by
        [6.3](06-accounts.md) having refused a links field and linkifying with
        `rel="noopener nofollow ugc"`.
      - **A curious logged-in user** changing an id in a URL to see someone's private collection or
        another pair's conversation. This is the one that requires real care, and
        [14.3](14-security.md)'s IDOR rule is the answer.
      - **A targeted attacker**, who is out of scope. We hold nothing money-shaped
        ([1.7](01-product.md), [3.2](03-collection.md)) and no credentials to anything else
        ([10.2.1](10b-import.md) refused Discogs OAuth precisely so that we would not).
      **What we explicitly do not defend against, stated so no document implies otherwise:** the
      operator, who can read everything ([13.3](13-legal.md) says so in those words); the hosting
      and storage providers, mitigated only for backups ([9.3](09-nfr.md) encrypts the dump); a
      volumetric DDoS ([9.4](09-nfr.md)); a compromised developer machine, which is game over and
      whose only mitigation is that deploys come from it; and supply-chain compromise of a Hackage
      dependency, which we pin ([11.11](11-stack.md)) and do not audit.
      **The worst realistic case, written down:** disclosure of the whole database — every private
      message and every email address, in plaintext. That single sentence is why
      [9.3](09-nfr.md)'s dumps are encrypted, why [13.3](13-legal.md) may not promise
      confidentiality it cannot deliver, and why [6.2](06-accounts.md) *deferred* 2FA rather than
      refusing it. It is also the argument for keeping [13.3](13-legal.md)'s data table short: the
      cheapest defence available to us is not holding things ([6.3](06-accounts.md)).
- [x] 14.2 Password storage, password policy, brute-force protection.
      **Decision: Argon2id, a length minimum and a common-password check instead of composition
      rules, and per-account exponential backoff rather than a lockout.**
      **Storage.** Argon2id via `crypton`/libsodium — the default recommendation, memory-hard, and
      already in the dependency set for other hashing. Parameters tuned so one hash takes roughly
      100–250 ms on the target box, starting at 64 MiB / t=3 / p=1 and dropping the memory rather
      than changing algorithm if a 2 GB box ([1.4](01-product.md)) complains; [14.5](14-security.md)
      bounds how many can run at once, which is what makes a memory-hard hash affordable here. The
      encoded string carries its parameters, so a login re-hashes when the parameters change. No
      pepper — it is a second secret to store on the same box the database backup is taken from.
      **Policy: minimum 10 characters, and that is the entire rule.** No composition requirements,
      no rotation, no security questions, no maximum below 128, every Unicode character accepted
      including spaces. Composition rules produce `Password1!` and teach users to write passwords
      down; length plus a blocklist is what current guidance actually says.
      **A vendored list of the ~10,000 most common passwords** is checked at registration and at
      password change. One file, one lookup, no service call ([1.5](01-product.md)), and it rejects
      more real-world-guessable passwords than any rule we could write.
      **Sessions** ([11.6](11-stack.md)): 32 bytes from the OS CSPRNG, and **the session table
      stores a SHA-256 of the token, not the token** — so a database read (or a leaked backup) does
      not hand over live sessions. Cookie as [13.4](13-legal.md) describes: `HttpOnly`, `Secure`,
      `SameSite=Lax`. 30 days idle expiry. Password reset and email change invalidate **every**
      session, which [6.2](06-accounts.md) required and which is why [11.6](11-stack.md) refused
      JWT.
      **Brute force: backoff, not lockout, because a lockout is an account-denial tool handed to the
      attacker.** Failed logins increment a counter on the account; the response is delayed and then
      refused with an exponential window ([14.5](14-security.md)'s table), and the counter resets on
      success. A per-IP cap runs alongside it for the distributed case. No CAPTCHA
      ([14.5](14-security.md)), no email alert on failed logins.
      **Enumeration, which is the part that is easy to get wrong in three places at once:**
      - **Login** never distinguishes an unknown address from a wrong password, and runs a dummy
        Argon2 verification when the user does not exist so the timing does not either.
      - **Registration** with an address that already exists returns the same "check your email"
        page as a new one, and mails the existing account a "someone tried to register with your
        address" note instead of a verification link.
      - **Password reset** always reports that a mail has been sent.
      Tokens for verification and reset are 32 random bytes, single-use, stored hashed like
      sessions, and expire — 24 hours for verification, 1 hour for reset.
- [x] 14.3 XSS/CSRF/SSRF/IDOR — which framework mechanisms we use, what we check in review.
      **Decision: escaping by construction, one form helper that makes a missing CSRF token
      impossible, no SSRF surface to defend, and authorisation in the `WHERE` clause.**
      **XSS.** Lucid escapes everything by default, so the risk is entirely in the escape hatches:
      **`toHtmlRaw` is forbidden**, with a single named exception module for static inline SVG
      icons, and its absence elsewhere is a `grep` in review. The one place we build markup from
      user text is NOTES' text-rendering policy — the linkifier shared by message bodies
      ([5.5](05-messaging.md)) and profile bios ([6.3](06-accounts.md)). It **escapes first and
      linkifies the escaped text**, emits `rel="noopener nofollow ugc"`, and accepts only `http`
      and `https` schemes (no `javascript:`, no `data:`). It is the highest-risk function in the
      codebase, it exists exactly once — which is why NOTES insisted on one policy rather than
      two — and it gets property tests ([11.10](11-stack.md)).
      **A strict CSP, which [13.4](13-legal.md) has made nearly free:**
      `default-src 'self'; img-src 'self' <bucket-host>; script-src 'self'; style-src 'self';
      object-src 'none'; frame-ancestors 'none'; base-uri 'none'; form-action 'self'`.
      **No `unsafe-inline`**, and the two consequences are real constraints on templates: no inline
      `style` attributes or `<style>` blocks (everything in [11.3](11-stack.md)'s one stylesheet),
      and **no `hx-on:` attributes or inline event handlers** — HTMX's ordinary `hx-get`/`hx-post`
      attributes are unaffected, so this costs nothing we were using.
      **Other headers**, set once in middleware: HSTS, `X-Content-Type-Options: nosniff`,
      `Referrer-Policy: same-origin` — that last one is not boilerplate, it keeps
      [3.7](03-collection.md)'s share token out of other sites' referer logs, the same leak
      [9.5](09-nfr.md) closed by omitting query strings from our own.
      **CSRF: a synchroniser token bound to the session**, stored in the session row rather than a
      second cookie ([13.4](13-legal.md) keeps the cookie count at one). NOTES flagged that plain
      HTML means nothing adds it for you; the answer is that **the form helper is the only way to
      open a `<form>` in our templates** and it emits the hidden input — forgetting it is not a
      review item but a thing you cannot express. For HTMX requests outside a form, the token is
      supplied as `hx-headers` on `<body>` and the middleware accepts either the field or the
      header. `SameSite=Lax` is defence in depth and **not** the mechanism. All state change is
      `POST`; no `GET` mutates anything, which [7.3](07-search-ux.md) already implied by making
      `GET` the language of filters and sorting.
      **SSRF has no surface, and the rule that keeps it that way.** [1.5](01-product.md) bars the
      server from fetching anything, so the only outbound connections in the system are SMTP and
      object storage, both from configuration and never from user input. **No code path may take a
      URL from a user and fetch it** — which also means **no link previews, ever**, for messages or
      bios. Linkified URLs are rendered; they are never resolved.
      **IDOR, the one an actual user will try.** Authorisation is not a check performed after a
      load; it is a condition in the query. Every service function that reads or writes
      user-scoped data takes the acting user and constrains on it — `collection_item`, `wantlist`,
      `conversation` and `message` ([5.1](05-messaging.md)'s two participant rows), `import`
      ([10.2](10b-import.md)), export ([10.3.1](10c-export.md), owner-only), and profile settings.
      There is no `getById` followed by an `if`. Because that holds, **sequential integer ids are
      not a vulnerability** and are not required to be unguessable — a point that matters for
      [10.4.6](10d-model-requirements.md), which is still open. [3.7](03-collection.md)'s share
      link is the exception and is a capability rather than an identifier: unguessable random,
      unenumerable, and protected from leaking by the `Referrer-Policy` above.
      **SQL injection.** [11.4](11-stack.md) writes SQL by hand through `hasql`, so every value is a
      parameter and **string-concatenated SQL is forbidden**. The one genuinely dangerous spot is
      [7.3](07-search-ux.md)'s dynamic sorting and filtering, where a column name cannot be a
      parameter: it maps from a closed sum type to a fixed set of fragments, and user text never
      reaches the query string.
      **The review checklist, which is five greps:** `toHtmlRaw` outside its module; a `<form>` not
      built by the helper; a user-scoped query without the acting-user clause; any new outbound
      request; and any log line carrying something from [14.6](14-security.md)'s prohibited list.
- [x] 14.4 File upload security (type, size, processing, serving from a separate domain).
      **Decision: two upload paths, both capped at the edge; type from the bytes and never from the
      client; re-encoding is the sanitiser; images served from the bucket's own domain.**
      **Images** ([8.1](08-media.md), [8.3](08-media.md)):
      - **15 MB cap enforced at the reverse proxy** before a byte reaches the application
        ([9.4](09-nfr.md)), not in application code after buffering.
      - **Accepted types are JPEG, PNG and WebP**, determined by sniffing the leading bytes. The
        client's `Content-Type` and the filename are ignored, and **the filename is discarded
        entirely** — [8.2](08-media.md) keys by `sha256`, so there is no user-controlled string
        anywhere near the storage path. **SVG is refused outright**: it is a script container, and
        no amount of rasterising makes accepting it a good idea.
      - **The re-encode is the sanitiser.** [8.3](08-media.md) turns every upload into two WebP
        derivatives, so a polyglot or a file with a payload appended does not survive
        decode-and-re-encode. It also drops EXIF — including the GPS coordinates in a phone photo
        of a sleeve — which is a privacy property worth stating, and which requires *not* asking
        `vips` to preserve metadata.
      - **`vips` is the largest attack surface in the system**: a C library parsing hostile input.
        [8.3](08-media.md) already made it a subprocess, which is what makes it containable — a
        crash kills the child, not the server. It runs with a pixel-dimension cap (reject
        implausibly large images from the header before full decode, which is also the
        decompression-bomb defence), a memory limit and a timeout.
      **Serving from a separate domain**, which the item asks about and the answer is yes, for
      free: [8.2](08-media.md) already serves the public prefix from the bucket's own host, so
      uploaded bytes are never same-origin with a session cookie. Objects are written with the
      `Content-Type` **we** determined, `Content-Disposition: inline` and `nosniff`. The bucket is
      not writable by browsers — uploads go through the application, so there is no presigned-PUT
      path to abuse.
      **The import CSV** ([10.2](10b-import.md)): a 20 MB body cap at the edge (a 10,000-row
      Discogs export is a couple of megabytes), a row cap and a field-length cap applied while
      streaming, so neither a single enormous line nor a hundred thousand rows can exhaust memory
      on a box with 2 GB of it. The file lands in `import.file_bytes` ([10.2.6](10b-import.md)) and
      is deleted when the job ends, which is [13.3](13-legal.md)'s retention line for it.
      **CSV injection belongs to the export side and is named here because that is where it gets
      missed:** a field beginning `=`, `+`, `-` or `@` is a formula when [10.3](10c-export.md)'s
      file is opened in a spreadsheet, and an artist name or a collection note can begin with any
      of them. The export serialiser prefixes such fields. This is an implementation obligation on
      a closed section, not a reopening of it.
- [x] 14.5 Rate limiting and CAPTCHA (where exactly: registration, login, edits, messages). (see also [9.4](09-nfr.md), [10.1.6](10a-public-api.md))
      **Decision: no CAPTCHA at any stage, and one table of per-identity limits enforced in the
      service layer — this item owns the numbers, and there is no second list.**
      **CAPTCHA is refused, and this item only confirms it.** [4.9](04-editing.md) declined it for
      edits and [5.7](05-messaging.md) declined everything heavier than a rate limit for messages;
      [13.4](13-legal.md) adds the reason that closes it for good — every CAPTCHA is a third-party
      script in a product that has none, with a CSP entry, a privacy disclosure and a
      JavaScript dependency [11.3](11-stack.md) refuses. The barrier is
      [6.1](06-accounts.md)'s verified email, and it is load-bearing for this decision.
      **Mechanism:** fixed-window counters in a `rate_limit` table ([11.4](11-stack.md) already
      puts rate limits in Postgres), keyed by bucket, subject and window start, checked and
      incremented in the **service layer** — which is what makes [9.4](09-nfr.md)'s promise true
      that a future API ([10.1.6](10a-public-api.md)) inherits these instead of reimplementing them
      ([11.5](11-stack.md)). A daily job deletes expired windows.

      | Action | Limit | Keyed on |
      |---|---|---|
      | Registration | 3 / hour, 10 / day | IP |
      | Verification resend | 3 / hour | account + IP |
      | Failed login | exponential backoff from the 5th failure in 15 min | account (plus 20 / 15 min per IP) |
      | Password reset request | 3 / hour | account + IP |
      | Message send | 20 / hour, 100 / day | sender |
      | First message to a new recipient | 5 / day | sender |
      | Catalogue edit | 100 / hour | user |
      | New entity (release, master, artist, label) | 50 / hour | user |
      | Image upload | 30 / hour | user |
      | Import start | 3 / day, 1 concurrent | user |
      | Export | 10 / hour | user |
      | Report ([4.6](04-editing.md)) | 20 / day | user |
      | Search | 60 / minute | IP |

      **The numbers are chosen to be invisible to a human and fatal to a script**, and the one that
      matters most is *first message to a new recipient*: [5.7](05-messaging.md) identified
      unsolicited bulk messaging as the only real abuse vector in messaging, and five new
      conversations a day is generous for a population of tens.
      **Moderators and admins are exempt from the content limits** — a merge session would trip the
      edit cap — and **never from the authentication limits**.
      **Exceeding a limit returns 429 with a page saying when to retry.** Never a silent drop,
      never a shadowban: the same reasoning as [10.2](10b-import.md)'s rule that an import never
      discards silently — invisible refusal is indistinguishable from a bug.
      **What is deliberately not rate-limited:** reading public catalogue pages, which is
      [9.4](09-nfr.md)'s per-IP proxy cap's job, and where a limit would hit the crawler that
      [7.8](07-search-ux.md) wants.
- [x] 14.6 Logging of personal data and retention period.
      **Decision: [9.5](09-nfr.md) fixed the shape and 30 days; this item fixes the contents, as a
      prohibition list that is short enough to remember.**
      **Never logged, at any level, in any environment:** passwords; session tokens and their
      hashes; CSRF tokens; verification and reset tokens; [3.7](03-collection.md)'s share token;
      message bodies; email addresses; bio text; collection notes; request bodies; query strings
      ([9.5](09-nfr.md) omits them for exactly this reason); `Cookie` and `Authorization` headers;
      the contents of an uploaded file. A failure logs *that* a mail send failed and to which user
      id — not to which address.
      **Logged deliberately:** request id, timestamp, method, path without query string, status,
      duration, acting user id, client IP.
      **The user id rather than the address, and that is a design choice not a shorthand.** It
      resolves through the database while the account exists and **stops resolving when
      [6.5](06-accounts.md) anonymises the row** — so account deletion propagates into the logs for
      free, which is not true of any string we could have written instead.
      **The IP is the one genuinely personal field we keep.** One purpose (abuse investigation and
      [14.5](14-security.md)'s forensics), legitimate interest as the basis, 30 days, and
      [13.3](13-legal.md) discloses it. Retention is enforced by journald
      (`MaxRetentionSec=30day`, [12.1](12-infrastructure.md)) and journald is the only sink — no
      log shipping, no third party, because [9.5](09-nfr.md) refused Sentry and
      [13.4](13-legal.md) refused everything else.
      **[9.5](09-nfr.md)'s nightly error digest leaves the box by email**, so it is bound by the
      same list: counts, messages and stack frames, never payloads. If an error message would
      naturally interpolate user data, it does not.
      **Two stores that are emphatically not logs and have their own answer:**
      [4.2](04-editing.md)'s revisions are kept forever and re-attributed to a tombstone on
      deletion ([13.3](13-legal.md) states this), and [9.3](09-nfr.md)'s backups hold a deleted
      account until they age out — up to ~90 days, disclosed, and never restored to recover a
      deleted account.

## Working notes

- **2026-08-15 — Section closed. All 6 items decided**, and five of them are consequences rather
  than choices: [1.5](01-product.md) deleted SSRF, [11.5](11-stack.md) decided where authorisation
  lives, [9.4](09-nfr.md) and [9.5](09-nfr.md) decided the crude layer and the log line,
  [8.3](08-media.md) decided the upload pipeline. The genuinely new material is
  [14.2](14-security.md)'s password answer, [14.5](14-security.md)'s numbers, and the enumeration
  and CSP rules in [14.2](14-security.md)/[14.3](14-security.md).
- **2026-08-15 — What this section hands on.**
  [Section 11](11-stack.md)'s session table gains two expectations from
  [14.2](14-security.md): the token is stored hashed, and the CSRF token lives on the session row
  ([13.4](13-legal.md) keeps the cookie count at one). Neither changes a decision; both change a
  column list.
  [10.3](10c-export.md)'s serialiser inherits one implementation obligation — prefix fields
  beginning `=`, `+`, `-`, `@` ([14.4](14-security.md)).
  [Section 12](12-infrastructure.md) inherits the edge caps (15 MB images, 20 MB CSV, small
  everywhere else), the security headers and CSP as middleware, `MaxRetentionSec=30day`, and the
  fact that `vips` runs as a constrained subprocess.
  [Section 15](15-roadmap.md) inherits three small tasks that are invisible until missing: vendoring
  the common-password list, writing the form helper before the first form, and the linkifier's
  property tests.
- **2026-08-15 — The two rules most likely to be broken by accident**, worth repeating wherever
  code review is discussed: a `<form>` written by hand instead of through the helper (no CSRF
  token), and a service function that loads by id and then checks ownership instead of constraining
  in the query. Everything else in [14.3](14-security.md) is a `grep`.
- **2026-08-15 — What was declined.** CAPTCHA at every stage and in every form; a pepper alongside
  the password hash; password composition rules and rotation; hard account lockout; email alerts on
  failed logins; 2FA (deferred at [6.2](06-accounts.md), not re-opened here); SVG uploads; link
  previews for URLs in user prose; log shipping to any third party; and any second rate-limiting
  design beside [14.5](14-security.md)'s table.

# 9. Non-functional requirements

**Priority:** P1

**What this section is not free to decide.** [1.4](01-product.md) sized the system, named the
target shape (one small VPS, one process, deploys that drop the site for ten seconds) and wrote an
explicit veto list to be cited *from here*. It also named the three things the small numbers do not
excuse, and two of them are this section's: images as the figure that breaks first, and backups as
an operational risk that must be taken seriously and **tested**. NOTES' *no real-time transport
anywhere* and *one cache, and the test that admits it* both bind here. What is genuinely open is the
backup regime, the observability answer, and the browser baseline that [8.3](08-media.md) is waiting
on — everything else is arithmetic against [1.4](01-product.md).

- [x] 9.1 Target performance: response time, catalogue size, the heaviest queries (a 10,000-release collection, search).
      **Decision: budgets on server time, three queries named as the heavy ones, and the failure mode
      we actually expect is N+1 rather than row count.**

      | Path | Budget (p95, server time) |
      |---|---|
      | Catalogue page (release, master, artist, label) | 150 ms |
      | Search results with facet counts ([7.3](07-search-ux.md)) | 300 ms |
      | Collection listing, one page of a 10,000-item collection | 300 ms |
      | Collection statistics ([3.8](03-collection.md)) | 500 ms |
      | Image upload, end to end ([8.1](08-media.md), [8.3](08-media.md)) | 3 s |
      | Import | no latency budget at all |

      **Catalogue size is [1.4](01-product.md)'s table and is not restated here.** ~12,000 releases,
      ~15,000 collection items, ~50,000 revisions — the entire database fits in RAM several times
      over, so every one of these budgets is met by a correct index and missed only by a mistake.
      **The three heaviest queries, named so they get an index and a test:**
      1. **Search with live facet counts.** NOTES forbids caching those counts, so each result page
         runs the FTS query plus its aggregates. This is the one that will be slowest and the one to
         re-measure after the catalogue grows.
      2. **A large collection listing with a tag filter and a sort**, joined through to master and
         artist for display — the query that touches the most tables.
      3. **Statistics** ([3.8](03-collection.md)): a live `GROUP BY` over one user's items read
         through to the master.
      **The upload is the only multi-second path and that is accepted** ([8.3](08-media.md) chose
      synchronous `vips`): the user has just chosen a file and is waiting on their own action. Three
      seconds is the budget, ten is a bug.
      **Import is measured as throughput, not latency** — it is a background job with progress
      ([10.2.8](10b-import.md)). A 5,000-row file should finish in minutes, not hours; the number
      that matters is that it is restartable ([11.7](11-stack.md)), not that it is fast.
      **The expected failure mode is N+1, not volume.** [11.4](11-stack.md) writes SQL by hand, so a
      list page that fetches its cover images or artist credits per row will be slow at 50 rows while
      the database is bored. The discipline is one query per list page plus one for its associated
      collections, and it is a review item, not a monitoring item.
      **When a budget is missed the answer is an index or a rewritten query.** Not a cache — NOTES
      allows exactly one and states the test — and not a bigger architecture, which
      [1.4](01-product.md) vetoes by name. A bigger VPS is allowed and is the cheapest fix available.
      **Payload budget, because it is the part a user actually feels:** one stylesheet and one
      vendored HTMX file ([11.3](11-stack.md)), no fonts, no third-party anything; a catalogue page
      under ~100 KB of HTML and CSS, with `loading="lazy"` and `srcset` over
      [8.3](08-media.md)'s two derivatives doing the rest. At [1.4](01-product.md)'s scale the
      network, not the database, is the whole of perceived latency.
- [x] 9.2 Availability/SLA, maintenance window.
      **Decision: no SLA, none offered, and no maintenance window — because deploys are the
      maintenance window and they are ten seconds.**
      [1.4](01-product.md) fixed one process on one box with no replica, no load balancer and no
      failover. Anything shaped like an availability target would be a promise the architecture
      cannot keep, so the honest figure is *best effort*: hours per month may be lost to a host
      incident, a bad deploy or a full disk, and nobody is paged. No status page — at tens of users
      the status page has the same readership as the site.
      **What we do promise instead is durability, and it is a much stronger promise:** no data loss
      beyond [9.3](09-nfr.md)'s window. That is the sentence worth putting in front of users
      ([13.3](13-legal.md)); an uptime claim is not.
      **No maintenance window, deliberately.** A window exists to concentrate disruption where users
      are not; we have neither the users nor the traffic pattern to find such an hour, and announcing
      one would be theatre. Deploy whenever.
      **The one case that can exceed ten seconds is a migration**, since [11.8](11-stack.md) applies
      them at startup inside a transaction. Two consequences to design against: a migration that
      rewrites a large table holds the site down for its duration, and `CREATE INDEX CONCURRENTLY`
      cannot run in a transaction block at all. **Rule:** any migration expected to take more than a
      few seconds is run by hand against the live database *before* the deploy that needs it, and the
      startup migration is then a no-op. Rare at our size, and much cheaper to state now than to
      discover during an outage.
- [x] 9.3 Backups: frequency, retention, restore verification.
      **Decision: a nightly encrypted `pg_dump` pushed off the box, a weekly bucket sync, 7/4/3
      retention, and a restore drill that is scripted and actually run.**
      [1.4](01-product.md) called this an operational risk rather than a capacity one and it is the
      only item in this section where the small numbers make things *worse*: there is no replica, no
      second box and no colleague, so the database is one machine away from total loss.

      | | |
      |---|---|
      | **Database** | `pg_dump -Fc` nightly, encrypted (`age`, public key on the box, private key **not** on the box), uploaded to object storage |
      | **Where** | A **private** bucket, separate from [8.2](08-media.md)'s public image prefix, ideally at a second provider — a backup on the same VPS is not a backup, and a backup in the bucket we are also backing up is not two copies |
      | **Retention** | 7 daily, 4 weekly, 3 monthly |
      | **RPO / RTO** | 24 hours / about an hour, both accepted explicitly |
      | **Images** | Weekly `rclone sync` of the image prefix to a second bucket |

      **Encryption is not decoration here.** The dump contains every private message in plaintext
      ([5.8](05-messaging.md)) and every email address; unencrypted, it hands the storage provider
      the most sensitive thing in the system. Encrypt before upload, and keep the private key
      somewhere the box cannot read — a lost key means lost backups, which is the trade being made
      knowingly.
      **The bucket needs its own backup and `pg_dump` does not cover it** ([8.2](08-media.md)'s
      working notes say so). A restored database pointing at a lost bucket is a catalogue of broken
      images, and because [8.1](08-media.md) discards the original, **image loss is permanent** —
      there is nothing to re-derive from. Weekly is enough given content-addressed immutable keys:
      nothing is ever modified, only added, so a week's exposure is a week's uploads.
      **WAL archiving and point-in-time recovery are declined**, at this size: it is a second
      mechanism to configure, monitor and test, to shrink an RPO of 24 hours for a database taking a
      few edits a day. Reopen when losing a day of edits would actually hurt.
      **Restore verification is the item, and the rest is preamble.** An untested backup is a belief.
      1. **The restore is a script in the repository**, not a remembered sequence of commands:
         download, decrypt, `pg_restore` into a scratch database, run the application against it.
      2. **It runs monthly, by hand, from the real production backup**, and the check is not "the
         restore exited 0" but: row counts within expectation, the newest revision is from yesterday,
         the app boots, one release page renders, one login works.
      3. **CI runs the same script every week against a dump of the development fixtures**
         ([11.8](11-stack.md)), which catches the ordinary breakage — a changed Postgres version, a
         renamed flag, a script that rotted — without touching production data.
      4. **The first drill happens before the first real user**, not after, and
         [section 15](15-roadmap.md) carries it as a task.
      **Two notes for whoever reads a dump's size and worries:** [10.2.6](10b-import.md) keeps the
      uploaded CSV in `file_bytes` on the `import` row, so a dump taken while an import is running
      carries those megabytes and the next one does not; and [4.2](04-editing.md)'s revision
      snapshots are the table that grows fastest, by design.
      Where backups live becomes a jurisdiction statement the moment [12.1](12-infrastructure.md)
      picks a provider — [13.3](13-legal.md) names the country on the privacy page, and
      [14.6](14-security.md) notes that a backup taken before an account deletion still contains that
      account until it ages out.
- [x] 9.4 Limits and anti-abuse at the API level. (see also [10.1.6](10a-public-api.md), [14.5](14-security.md))
      **Decision: two layers, and this item owns only the crude one. There is no second rate-limiting
      design.**
      **Per-identity limits live in the service layer and are [14.5](14-security.md)'s table** — one
      list, enforced below HTTP, which is what makes a future public API
      ([10.1.6](10a-public-api.md)) inherit them instead of reimplementing them
      ([11.5](11-stack.md)'s reuse boundary). This item must not invent a parallel set of numbers;
      [5.7](05-messaging.md) already said the same thing when it fed [14.5](14-security.md).
      **What belongs here is protection of the box rather than of a feature**, because those are
      genuinely different concerns and only one of them survives an unauthenticated flood:
      - A per-IP request cap at the reverse proxy ([12.1](12-infrastructure.md)) — generous, in the
        order of a few requests a second with a burst, aimed at a scanner rather than a user.
      - A request body cap at the edge, before anything is read: [8.1](08-media.md)'s 15 MB for
        images, [14.4](14-security.md)'s cap for the import CSV, and something small for every other
        route. A form post is kilobytes.
      - A request timeout, so a slow-body attack cannot hold connections open.
      **There is no API in the MVP** ([1.9](01-product.md), [11.5](11-stack.md)), so nothing here is
      waiting on [section 10.1](10a-public-api.md); when an API appears it needs a key or a token to
      count against, and that is [10.1.6](10a-public-api.md)'s decision, not a gap in this one.
      **Actual DDoS is out of scope and saying so is the honest answer.** One small VPS cannot be
      defended against a volumetric attack by any means we would build; the mitigations available are
      the host's network-level protection and waiting. [9.2](09-nfr.md) already declined to promise
      availability, and this is one of the reasons.
- [x] 9.5 Observability: logs, metrics, tracing, alerts, error tracking (Sentry).
      **Decision: structured logs to stdout, a health endpoint, two alerts and a nightly error
      digest. No metrics stack, no tracing, and no Sentry.**
      **Logs.** JSON lines to stdout, captured by journald ([12.1](12-infrastructure.md)). One line
      per request with a request id, method, path *without the query string*, status, duration and
      the acting user's id; the same request id on every line the request emits, which is the whole
      of our correlation story. **What may never appear in a log line is [14.6](14-security.md)'s
      list**, and the query string is omitted precisely because it can carry
      [3.7](03-collection.md)'s share token. Retention 30 days ([14.6](14-security.md)).
      **No Sentry, and this is a real trade rather than a reflex.** Error tracking is the highest
      value observability money can buy for a solo developer — it tells you about failures no user
      reports. Against it: a third-party account on the critical path of nothing, a DSN to keep
      secret, a client library to maintain in a language where it is thin, an entry in
      [14.3](14-security.md)'s CSP, and stack frames containing user data leaving the box, which
      [13.3](13-legal.md) would then have to disclose.
      **What replaces it costs nothing and is 90% as useful:** a nightly cron greps error-level lines
      from the last 24 hours and mails them to the operator, reusing the sender
      [section 6](06-accounts.md) already requires. Counts and messages, not payloads. If the day
      comes when that digest is unreadably long, the problem is the errors, not the tooling.
      **No metrics stack.** Prometheus plus Grafana is two more daemons on a 2 GB box —
      [1.4](01-product.md)'s veto list in spirit if not by name — to graph a system with one process
      and ten users.
      What replaces it is **one authenticated `/status` page** showing what an operator would
      actually look at: database reachable, applied migration version, queue depth and the age of the
      oldest pending job ([11.7](11-stack.md)), failed job count, and the timestamp of the last
      successful backup. Plus an unauthenticated `/healthz` that checks the database and returns 200,
      for the pinger below.
      **`pg_stat_statements` is enabled**, and it is the exception worth making: it is stock contrib,
      needs no daemon, and is the only way to diagnose a [9.1](09-nfr.md) budget miss without
      guessing. It requires `shared_preload_libraries` and therefore a Postgres restart, which
      [12.1](12-infrastructure.md) should do once at setup rather than discover later.
      **No tracing.** One process, no network hops between components, no queue-crossing request
      path. A request id in a log line *is* the trace at this architecture, and OpenTelemetry would
      be instrumentation, a collector and a backend to answer a question we can answer with `grep`.
      **Exactly two alerts, both by email, both about things that are silently broken:**
      1. **The site is down** — an external pinger against `/healthz` (a free uptime service or a
         cron on any other machine; it must not run on the box it is watching).
      2. **The nightly backup did not complete** — the job mails on failure *and* the absence of a
         success line for 36 hours is itself the alarm, because a backup that stopped running
         silently is [9.3](09-nfr.md)'s worst case.
      Everything else is looked at, not alerted on. With one recipient and no rota, a third alert
      is the one that teaches the operator to ignore the first two.
- [x] 9.6 Browser support.
      **Decision: current evergreen browsers, roughly the last two years — and [8.3](08-media.md)'s
      WebP bet is confirmed and paid for, closing NOTES' open dependency.**
      **Supported:** current and previous versions of Chrome, Firefox, Safari and Edge, plus Safari
      on iOS and Chrome on Android. Nothing older, no Internet Explorer, no polyfills, no
      transpilation — and there is nothing to transpile, since [11.3](11-stack.md) has no build step
      and no JavaScript of our own.
      **WebP with no fallback is safe.** Safari 14 shipped it in September 2020 and was the last
      holdout; every browser in the baseline above has supported it for years. [8.3](08-media.md)
      made the bet explicitly rather than assuming it, and this is where it is confirmed: **no second
      derivative, no JPEG fallback, and the bucket stays at [8.2](08-media.md)'s size.** Had it gone
      the other way the price was a doubled bucket — the number [1.4](01-product.md) was watching.
      **The riskiest thing we use is CSS, not images.** [11.3](11-stack.md)'s stylesheet leans on
      custom properties and grid, which are ancient by now, and on `light-dark()`, which is not:
      Chrome 123, Safari 17.5, Firefox 120, all 2024. An unsupported `light-dark()` makes the whole
      declaration invalid and the element loses its colour, so **every `light-dark()` is preceded by
      a plain fallback declaration**. That is the entire progressive-enhancement policy for CSS, and
      it is one convention rather than a build tool.
      **Below the baseline the site still works**, which is a property of the decisions rather than a
      commitment we maintain: [11.3](11-stack.md) requires every core flow to work with JavaScript
      disabled, so an old or unusual browser gets working HTML forms and unstyled-but-readable
      content. The same property is what makes the accessibility story cheap — semantic HTML,
      keyboard-usable by construction, no ARIA machinery to get wrong. [7.7](07-search-ux.md)'s
      `prefers-color-scheme` needs no support beyond the baseline.
      **Testing is manual, on one Chromium, one Firefox and a phone.** No BrowserStack, no matrix, no
      automated cross-browser suite: [11.10](11-stack.md)'s e2e smoke runs on one engine and that is
      proportionate to a page with no scripted behaviour to diverge.

## Working notes

- **2026-08-15 — Section closed. All 6 items decided.** Four of them are readings of
  [1.4](01-product.md) rather than new choices ([9.1](09-nfr.md), [9.2](09-nfr.md),
  [9.4](09-nfr.md), and half of [9.5](09-nfr.md)); the two that were genuinely open are the backup
  regime ([9.3](09-nfr.md)) and whether to buy error tracking ([9.5](09-nfr.md)), and both are
  stated with their cost.
- **2026-08-15 — What this section hands on.**
  [Section 12](12-infrastructure.md) inherits nearly all the mechanism: a nightly backup job and its
  encryption key, a second private bucket, a weekly image sync, journald retention set to 30 days,
  `pg_stat_statements` in `shared_preload_libraries`, a reverse proxy with request and body caps
  ([9.4](09-nfr.md)), an external uptime pinger, and the rule that a slow migration is run by hand
  before the deploy ([9.2](09-nfr.md)).
  [Section 15](15-roadmap.md) inherits three tasks that are easy to leave undone forever: the
  restore script, the first restore drill *before* the first real user, and the `/status` page.
  [Section 13](13-legal.md) inherits the durability-not-availability line ([9.2](09-nfr.md)) and the
  fact that backups hold deleted accounts until they age out.
  [8.3](08-media.md) gets its answer: WebP, no fallback, confirmed.
- **2026-08-15 — What was declined, in one place.** WAL archiving and PITR, Sentry and every hosted
  error tracker, Prometheus/Grafana, OpenTelemetry, a status page, an availability target, a
  maintenance window, and cross-browser testing infrastructure. Each is a real tool declined against
  [1.4](01-product.md)'s scale rather than a thing we disapprove of; the two most likely to be
  reopened first are error tracking (the moment a user reports a bug we cannot reproduce) and PITR
  (the moment a day of lost edits would actually matter).
- **2026-08-15 — The number to watch is still images.** [8.2](08-media.md)'s revised estimate is
  ~6.5 GB at [1.4](01-product.md)'s counts, and [9.3](09-nfr.md) now attaches a weekly sync to it.
  The figure that would force a rethink is not the storage cost — object storage is cents per
  gigabyte — but the time and egress of that sync, which is the form [1.4](01-product.md)'s warning
  actually takes.

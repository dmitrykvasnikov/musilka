# E0 — Skeleton

**Size:** 3–4 weeks ([15.1](../design/15-roadmap.md)) · **Tasks:** T-1 … T-33

**Exit:** a deployed application, a green pipeline, and a restore that has actually been performed.
Not a feature — a rehearsal of the whole architecture with one column of business logic, which is
the cheapest possible way to find out that the shape is wrong.

**This is where the risk is.** [15.4](../design/15-roadmap.md) ranks Haskell pace and
[1.12](../design/01-product.md)'s four known gaps as the largest risk in the project, and E0
concentrates three of the four: `ReaderT AppEnv IO` in the large (T-5), the database layer (T-7,
T-8), and build and deploy (T-19 … T-25). If this stage runs well past four weeks that is the
signal [15.4](../design/15-roadmap.md) names — and the response is
[11.2](../design/11-stack.md)'s rule, not a language change on a bad week.

**Week one** ([15.3](../design/15-roadmap.md)) is **T-1, T-4, T-6, T-7, T-8, T-9, T-16, T-18, T-19**
and, if [12.1](../design/12-infrastructure.md) has a provider by then, **T-21, T-23, T-24**. If it
does not, week one ends at green CI and the deploy slips — the work does not. Note that production
will show an empty catalogue whatever happens: [10.2.10](../design/10b-import.md) forbids loading
catalogue rows by any channel including `psql`, so the release page is exercised by T-17's fixtures
and by T-18's integration test, and the first real row arrives through a form in E1.2.

**Nine of these tasks are `[~]`** and every one of them waits, directly or through another, on the
same thing: nobody has picked a hosting provider or a mail sender
([12.1](../design/12-infrastructure.md), [12.5](../design/12-infrastructure.md)). Nothing analytical
is blocked, and the other 24 tasks can be finished without either name.

## The repository

- [ ] T-1 Repository and Cabal project (per 11.11)
      `cabal.project` with one pinned GHC and a pinned `index-state`; three libraries —
      `musilka-domain` (pure), `musilka-app` (services), `musilka-web` (Servant, Lucid, sessions,
      the job runner) — plus a thin executable that wires them.
      **`musilka-web` must not list `hasql` in `build-depends`** ([10.4.4](../design/10d-model-requirements.md)):
      a query written in a handler is then a build error rather than a review comment.
      No `package.json`, no bundler, no CSS pipeline, ever ([11.3](../design/11-stack.md)).
- [ ] T-2 `LICENSE`, `README`, and a public repository from the first commit (per 13.2)
      MIT, one file covering everything including `design/` and `plan/`. The one discipline a public
      repository costs: **no secret ever enters it** — not the storage keys, not the mail
      credentials, not [9.3](../design/09-nfr.md)'s backup *private* key. Its public key may be
      committed.
- [ ] T-3 Formatter, linter and warning flags (per 12.7)
      `fourmolu` with a checked-in config, `hlint` with a small `.hlint.yaml`, `-Wall` in the cabal
      files. `-Werror` is added in CI only (T-19) — warnings should block a deploy without making a
      local experiment miserable.
- [ ] T-4 `docker compose` for local development (per 12.2)
      Exactly two containers: Postgres on the pinned major ([11.4](../design/11-stack.md)) and MinIO
      standing in for the bucket. The application itself runs from `cabal run` and is never
      containerised locally.
- [x] T-33 `plan/` in the repository (per 12.8, 15.6)
      Done 2026-08-15. Four files cut by stage, task IDs permanent, progress derived by `grep`.

## The application skeleton

- [ ] T-5 `AppEnv` and the application monad (per 11.1, 1.12)
      `ReaderT AppEnv IO`, settled immediately and not revisited; effect systems are explicitly out
      of scope. This is the first of [1.12](../design/01-product.md)'s four gaps and the one that
      shapes every module after it — spend a sitting getting the shape right rather than discovering
      it in E1.
- [ ] T-6 Typed configuration decoded at startup (per 12.6, 11.9)
      Every value — secret and not — decoded into a typed `Config` from the environment, and the
      application **refuses to boot on anything missing or malformed** rather than defaulting. That
      is parse-don't-validate at the configuration edge, and it turns a typo into a failed deploy
      that rolls back (T-25) instead of a 3am surprise. Nothing from the config is ever logged
      ([14.6](../design/14-security.md)).
- [ ] T-7 `hasql` pool and the database edge (per 11.4, 1.12)
      Hand-written SQL, explicit codecs, statements as ordinary values. Peer authentication over the
      unix socket in production (T-21); TCP against compose locally. The second of
      [1.12](../design/01-product.md)'s gaps — [11.4](../design/11-stack.md) notes this is the
      most reversible decision in the stack, so build it behind the service layer and do not
      optimise it.
- [ ] T-8 Migration runner (per 11.8)
      Numbered plain-SQL files embedded with `file-embed`, applied at startup inside a transaction
      against a `schema_migration` table. **Forward-only, no down-migrations.** The runner is part
      of the binary because [11.11](../design/11-stack.md) keeps the deploy at one artifact.
      Remember what this implies for T-25: **the restart is the migration**, so
      [9.2](../design/09-nfr.md)'s rule binds — anything expected to take more than a few seconds is
      run by hand against production *before* the deploy that needs it.
- [ ] T-9 Migration 001 (per 10.4.6, 15.3)
      `artist`, `master`, `release`, minimal columns only, with
      `bigint GENERATED ALWAYS AS IDENTITY` on each — no `serial`, no shared sequence, `bigint` even
      where `int` would obviously do. `release.master_id` is `NOT NULL`
      ([2.1.2](../design/02-catalogue-model.md)).
      **This is the irreversible one in E0:** from the first exported file onward
      ([10.3.2](../design/10c-export.md)), changing the identifier means a migration plus every file
      already in a user's hands being wrong.
- [ ] T-10 Job queue table and in-process runner (per 10.4.5, 11.7)
      One table, `SELECT … FOR UPDATE SKIP LOCKED`, an attempt count and a last error; failed jobs
      stay in the table, visible rather than dropped. It lands in E0 rather than with its first
      consumer because that consumer is [6.1](../design/06-accounts.md)'s verification email in
      E1.1 — the barrier every anti-abuse decision leans on.
      **Every job must be restartable**, because a deploy kills whatever is running. The primitive
      version is the final version; there is no broker upgrade path
      ([11.7](../design/11-stack.md) refused it).
- [ ] T-11 Base layout, stylesheet and vendored HTMX (per 11.3, 9.6, 7.7)
      One Lucid layout with landmarks and one `<h1>` per page; one hand-written stylesheet — custom
      properties, grid, mobile-first with two breakpoints; one version-pinned HTMX file served from
      our own static directory. **Every `light-dark()` is preceded by a plain fallback declaration**
      ([9.6](../design/09-nfr.md)) — an unsupported one invalidates the whole declaration and the
      element loses its colour. No inline `style` attributes and no `<style>` blocks, because T-13's
      CSP forbids `unsafe-inline`.
- [ ] T-12 The form helper (per 14.3)
      **Before the first form exists, or it is retrofitted forever.** The helper is the only way to
      open a `<form>` in our templates and it emits the hidden CSRF input — forgetting the token
      must not be a review item but a thing you cannot express. Middleware accepts either the field
      or an `hx-headers` header on `<body>`. All state change is `POST`; no `GET` mutates anything.
      The token's storage lands with the session row in E1.1 (T-36); the helper and the middleware
      contract come first.
- [ ] T-13 Security headers and CSP middleware (per 14.3, 13.4)
      `default-src 'self'; img-src 'self' <bucket-host>; script-src 'self'; style-src 'self';
      object-src 'none'; frame-ancestors 'none'; base-uri 'none'; form-action 'self'`, plus HSTS,
      `X-Content-Type-Options: nosniff` and `Referrer-Policy: same-origin`. **No `unsafe-inline`**,
      which is free only because [13.4](../design/13-legal.md) put no third party in the browser.
      `Referrer-Policy` is not boilerplate — it keeps [3.7](../design/03-collection.md)'s share
      token out of other sites' referer logs.
      Headers live in the application, not in nginx: one place, versioned with the templates they
      constrain.
- [ ] T-14 Request logging (per 9.5, 14.6)
      One JSON line per request to stdout: request id, timestamp, method, **path without the query
      string**, status, duration, acting user id, client IP. The same request id on every line the
      request emits — that is the whole correlation story, and there is no tracing.
      The query string is omitted precisely because it can carry a share token. Implement
      [14.6](../design/14-security.md)'s prohibition list as a habit now, while there is almost
      nothing to leak.
- [ ] T-15 `/healthz` (per 9.5)
      Unauthenticated, checks the database, returns 200. Consumed by T-25's deploy poll and T-30's
      external pinger.
- [ ] T-16 First service function and the release page (per 10.4.4, 7.5, 15.3)
      One service function in `musilka-app` taking an explicit `Actor`; one Servant route
      `/release/<id>/<slug>`; one Lucid page. The slug is cosmetic and ignored on read
      ([10.4.6](../design/10d-model-requirements.md)). Handlers decode, call exactly one service
      function, and render; templates receive domain values and query nothing.
      This task is the point of week one — it touches migration, codec, service, route and template
      exactly once.
- [ ] T-17 Development fixture generator (per 11.8, 12.4)
      One command, never run in production, **generated rather than hand-written** so it can grow: a
      few users, a few hundred releases, an import in each state. Development never runs against
      production data — the database holds message bodies and email addresses in plaintext, so
      copying it to a laptop would be the worst data-handling decision available in this design.

## CI and the box

- [ ] T-18 Test harness (per 11.10)
      `hspec` as the runner with `hspec-hedgehog` for properties, and a **disposable real Postgres
      per run** — never a mock, since [11.4](../design/11-stack.md) writes SQL by hand and a mocked
      database would test nothing that can break. No coverage percentage, deliberately; what
      replaces it is [11.10](../design/11-stack.md)'s named list, and the tasks that owe entries to
      it say so.
- [ ] T-19 CI pipeline (per 12.3)
      GitHub Actions, on every push and pull request: `build` with `-Werror`, `format`
      (`fourmolu --mode check`), `lint` (`hlint`), and `test` at all three levels with a Postgres
      service container and MinIO. Cache `~/.cabal/store` and `dist-newstyle` keyed on the GHC
      version and the freeze file — Haskell CI is unusable without it and a cold build is many
      minutes. Branch protection requiring green CI stays on even for one person.
- [ ] T-20 Builder container (per 12.2)
      The binary is built inside a container on the **same Debian release as the server**, which is
      what makes a dynamically linked binary safe to copy across. That is the whole of
      containerisation's role here: a reproducible builder, not a runtime. No Docker on the box, no
      Nix.
- [~] T-21 Provision the box (per 12.1, 11.4, 7.2, 9.5) — waiting on 12.1's provider
      One VPS, EU/EEA region ([13.3](../design/13-legal.md)), Debian stable, 2 vCPU / 4 GB. An
      unprivileged `musilka` user; Postgres on the pinned major reachable **only over the unix
      socket**, never a TCP port, with peer authentication.
      **Do the one-time Postgres setup at install, before there are users rather than during an
      incident:** `shared_preload_libraries=pg_stat_statements` ([9.5](../design/09-nfr.md)) plus
      `pg_trgm` and `unaccent` ([11.4](../design/11-stack.md), [7.2](../design/07-search-ux.md)) —
      the first needs a restart. Install `libvips`, `rclone`, `age` and `certbot` from the
      distribution's packages.
- [~] T-22 Domain, DNS and mail records (per 13.6, 12.5) — waiting on 12.5's sender
      **Register the domain and publish SPF, DKIM and DMARC in the same sitting as T-21**, not
      after. [NOTES.md](../design/NOTES.md) has flagged this since 2026-08-14 as the thing that
      quietly breaks a load-bearing decision: a fresh domain with no records lands
      [section 6](../design/06-accounts.md)'s four templates in spam, which breaks
      [6.1](../design/06-accounts.md)'s verified-email barrier, which
      [4.9](../design/04-editing.md), [5.7](../design/05-messaging.md) and
      [14.5](../design/14-security.md) all lean on instead of a captcha.
      DMARC at `p=none` moving to `p=quarantine` once the reports are clean. `abuse@` and `privacy@`
      forward to the operator's mailbox; **nothing on the box listens on port 25**.
- [~] T-23 nginx and TLS (per 12.5, 9.4, 14.4) — waiting on 12.1
      certbot for Let's Encrypt; HTTP → HTTPS; a generous per-IP request cap aimed at a scanner
      rather than a user; **body caps of 15 MB on the image upload route, 20 MB on the import route
      and something small everywhere else**, applied at the edge before a byte reaches the
      application; request and slow-body timeouts; pass-through to the socket the application
      listens on. Security headers stay in the application (T-13).
- [~] T-24 systemd unit and journald (per 12.2, 9.5, 14.6) — waiting on 12.1
      `Restart=on-failure`, `EnvironmentFile=/etc/musilka/env` (root-owned, `0600`),
      `ProtectSystem=strict`, `PrivateTmp`, `NoNewPrivileges` — which matter most for the `vips`
      subprocess [14.4](../design/14-security.md) calls the largest attack surface in the system.
      journald only, `MaxRetentionSec=30day`, no shipping anywhere.
- [~] T-25 Deploy workflow and rollback (per 12.3) — waiting on 12.1
      On green `main` only: build in T-20's container, `scp` to
      `/opt/musilka/releases/<git-sha>/`, flip the `current` symlink, `systemctl restart`, poll
      `/healthz`; on failure flip back and restart. Keep the last five releases.
      **Migrations are not a deploy step** — T-8 applies them at startup, so the restart is the
      migration, and a rollback does not undo one. No tags, no version numbers, no changelog:
      every green commit on `main` is the release. `workflow_dispatch` for a manual redeploy.
      The deploy credential is an SSH key in Actions secrets for a `deploy` user that may write only
      under `/opt/musilka/releases` and restart one unit through a narrow sudoers rule.

## Backups, and the drill that makes them real

- [~] T-26 Nightly encrypted backup (per 9.3) — waiting on 12.1's second provider
      `pg_dump -Fc` nightly, encrypted with `age` (public key on the box, **private key not on the
      box** — password manager plus paper), uploaded to a **private** bucket at a *different*
      provider from the image bucket. Retention 7 daily / 4 weekly / 3 monthly. RPO 24 hours, RTO
      about an hour, both accepted explicitly.
      Encryption is not decoration: the dump contains every private message in plaintext
      ([5.8](../design/05-messaging.md)) and every email address. A lost key means lost backups, and
      that is the trade being made knowingly.
- [ ] T-27 Restore script (per 9.3)
      **A script in the repository, not a remembered sequence of commands**: download, decrypt,
      `pg_restore` into a scratch database, run the application against it. This is also what
      replaces staging ([12.4](../design/12-infrastructure.md)) — it produces a scratch copy of real
      production data on demand, which is where [9.2](../design/09-nfr.md)'s risky migrations get
      rehearsed.
- [ ] T-28 Weekly CI restore against a fixtures dump (per 9.3, 12.3)
      CI runs T-27's script every week against a dump of T-17's fixtures. It catches the ordinary
      breakage — a changed Postgres version, a renamed flag, a script that rotted — without touching
      production data.
- [~] T-29 The first restore drill, by hand, from a real backup (per 9.3) — waiting on T-26
      **Before the first real user, not after.** The check is not "the restore exited 0" but: row
      counts within expectation, the newest revision is from yesterday, the app boots, one release
      page renders, one login works. Monthly thereafter.
      An untested backup is a belief, and this is the one item in E0 that is easy to leave undone
      for ever.

## Observability and the runbook

- [~] T-30 External uptime pinger and backup alarm (per 9.5) — waiting on T-21
      Exactly two alerts, both by email, both about things that are silently broken. The pinger hits
      `/healthz` and **must not run on the box it is watching**. The backup job mails on failure,
      *and* the absence of a success line for 36 hours is itself the alarm — a backup that stopped
      running silently is [9.3](../design/09-nfr.md)'s worst case.
      A third alert is the one that teaches the operator to ignore the first two.
- [~] T-31 Nightly error digest (per 9.5) — waiting on the mail sender (T-38, 12.5)
      A cron greps error-level lines from the last 24 hours and mails counts and messages — never
      payloads, per [14.6](../design/14-security.md) — to the operator. This is what replaces
      Sentry, at 90% of the value and none of the cost. If the digest is ever unreadably long, the
      problem is the errors.
- [ ] T-32 Runbook (per 12.6, 9.3, 9.2)
      In the repository, beside the restore script: deploy, rollback, restore, **secret rotation**
      (four secrets, by hand — undocumented rotation is rotation that never happens), and
      [9.2](../design/09-nfr.md)'s rule that a slow migration is run by hand against production
      before the deploy that needs it.

## Working notes

**2026-08-15 — what E0 deliberately does not contain.** No storage port and no `image` table: E1.9
owns them, because [15.1](../design/15-roadmap.md) puts images last on purpose and
[8.1](../design/08-media.md)'s discarding of originals is the one irreversible decision in the
design. [Section 8](../design/08-media.md)'s working notes would rather the interface existed early;
the roadmap's ordering wins, and the cost is that E1.9 starts with a port instead of a feature. Also
absent: any vocabulary seed data (E1.2), because nothing reads it yet, and the `/status` page (E1.1),
because it is authenticated and there is no auth.

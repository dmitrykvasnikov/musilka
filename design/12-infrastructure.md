# 12. Infrastructure and process

**Priority:** P2

**What this section is not free to decide.** It inherits more mechanism than it invents.
[1.4](01-product.md) fixed the target shape — one small VPS, Postgres on the same box, one process,
deploys that may drop the site for ten seconds — and wrote the veto list to be cited from here.
[9.3](09-nfr.md) specified the backup regime down to the retention numbers and the drill,
[9.4](09-nfr.md) the proxy caps, [9.5](09-nfr.md) journald and `pg_stat_statements`,
[14.4](14-security.md) the body limits, [11.8](11-stack.md) migrations at startup and
[11.11](11-stack.md) one binary as the artifact. [13.3](13-legal.md) requires everything holding
user data to sit in the EU/EEA — a constraint, not a preference. What is genuinely open here is
container-or-not, the CI shape, whether staging exists, how secrets are held, and where work is
tracked.

**Two items are deferred by choice, not by dependency** ([12.1](12-infrastructure.md),
[12.5](12-infrastructure.md)): naming a hosting provider and a mail sender was postponed on
2026-08-15. Everything in this section that does not depend on a vendor is decided, and nothing
elsewhere in the design waits on those two names.

- [~] 12.1 Where we host: VPS (Hetzner/Selectel), PaaS (Railway/Render/Fly), cloud, self-hosted at home.
      **Decision: the shape is settled — a single plain VPS in the EU/EEA with everything on it —
      and the provider is deliberately left open.**
      **PaaS is excluded, and not as a matter of taste.** Railway, Render and Fly are built around an
      ephemeral filesystem, a managed database as a separate paid add-on, and a scaling model
      [1.4](01-product.md) vetoes by name. We need the opposite of all three: Postgres on the same
      box reachable over a unix socket, a persistent disk for `dist`-less deploys and journald, a
      `vips` subprocess ([8.3](08-media.md), [14.4](14-security.md)) and `rclone`/`age` on cron
      ([9.3](09-nfr.md)). A PaaS makes each of those an exception; a VPS makes them the default.
      **Self-hosted at home is excluded** by [13.3](13-legal.md)'s EU/EEA requirement being a
      statement we publish, by residential uptime and IP reputation
      ([12.5](12-infrastructure.md)'s mail problem), and by [9.3](09-nfr.md)'s premise that the
      database is one machine away from total loss.
      **The box, as specified regardless of whose it is:**
      | | |
      |---|---|
      | **Shape** | One VPS, EU/EEA region ([13.3](13-legal.md)), Debian stable |
      | **Size** | 2 vCPU / 4 GB to start — [1.4](01-product.md) says 2 GB suffices for the data, but [14.2](14-security.md)'s Argon2id at 64 MiB per concurrent hash plus `vips` plus Postgres makes 4 GB the cheap way to never revisit it. A bigger VPS is [9.1](09-nfr.md)'s sanctioned fix for everything. |
      | **On it** | The binary as a systemd unit ([11.11](11-stack.md)); Postgres (one pinned major, [11.4](11-stack.md)) reachable **only over the unix socket**, never a TCP port; nginx in front ([12.5](12-infrastructure.md)); `libvips`, `rclone`, `age`, `certbot` from the distribution's packages |
      | **Postgres setup, once** | `shared_preload_libraries=pg_stat_statements` ([9.5](09-nfr.md)) and `pg_trgm` ([11.4](11-stack.md)) at install time, so the restart happens before there are users rather than during an incident |
      | **Logs** | journald only, `MaxRetentionSec=30day` ([9.5](09-nfr.md), [14.6](14-security.md)). No shipping anywhere |
      | **Object storage** | Two buckets at **different providers**: the public image prefix ([8.2](08-media.md)) and a private backup bucket ([9.3](09-nfr.md)). [9.3](09-nfr.md)'s reasoning is the whole argument — a backup inside the thing being backed up is one copy |
      | **Not on it** | Docker ([12.2](12-infrastructure.md)), any second daemon, any database but Postgres ([11.4](11-stack.md)) |
      **What is open: which provider, which region, and whether paying it is practical.** Deferred
      2026-08-15 at the author's request. It blocks nothing in the design — no decision above names a
      vendor — but it blocks the first deploy, and through [12.5](12-infrastructure.md) it blocks
      DNS and transactional mail, which [NOTES.md](NOTES.md) already flags as the thing that quietly
      breaks [6.1](06-accounts.md)'s verified-email barrier.
      **When it is taken up, the questions are:** an EU/EEA region for the box and both buckets
      ([13.3](13-legal.md) turns this into a published jurisdiction statement); two providers rather
      than one, for the backup separation; and a payment method that actually works, which is the
      part no architecture argument can settle.
- [x] 12.2 Containerisation and local development (docker compose?).
      **Decision: `docker compose` locally for Postgres and object storage only; the application runs
      from `cabal` in development and as a plain systemd unit in production. No Docker on the server,
      and no Nix.**
      **Local development.** GHC and Cabal from `ghcup`, pinned by `cabal.project` and `index-state`
      ([11.11](11-stack.md)). `docker compose up` starts exactly two containers: **Postgres** on the
      pinned major ([11.4](11-stack.md)) and **MinIO** standing in for the bucket. The edit loop is
      `cabal run` — putting the application in a container would cost seconds per iteration and hide
      `dist-newstyle`'s incremental build, which is the one thing a Haskell workflow cannot afford to
      lose.
      **MinIO rather than a filesystem adapter**, deliberately: a local-disk implementation of the
      storage port would be a second code path that can be right in development and wrong in
      production, and [8.2](08-media.md)'s content-addressed keys and cache headers are exactly the
      details that would diverge. One adapter, exercised everywhere. Fixtures come from
      [11.8](11-stack.md)'s generator, never from production data
      ([12.4](12-infrastructure.md) says why).
      **Production: a single binary plus static assets, installed by systemd, and no container.**
      [11.11](11-stack.md) works to keep the deploy at one artifact and a container would be a
      second: an image build, a registry, a pull on a small box, and a place for `vips`, `rclone`,
      `certbot` and journald to be either inside or outside. The isolation a container buys is not
      the property we need — there is one application on the box and nothing to isolate it from.
      **The binary is nonetheless built inside a container in CI**, on the same Debian release as the
      server, which is what makes a dynamically linked binary safe to copy across. That is the whole
      of containerisation's role here: a reproducible builder, not a runtime.
      **Nix is declined**, closing what [11.11](11-stack.md) left open. It would solve a
      reproducibility problem that a pinned GHC, a pinned `index-state` and a pinned builder image
      already solve for one machine and one developer — and [1.12](01-product.md) names build and
      deploy tooling as one of four known gaps, so adopting Nix would make the largest gap larger
      still. Reopenable at any time; nothing depends on the answer.
      **systemd carries the constraints rather than the application:** `Restart=on-failure`,
      `EnvironmentFile` ([12.6](12-infrastructure.md)), a dedicated unprivileged `musilka` user with
      peer authentication to Postgres, and the ordinary hardening directives
      (`ProtectSystem=strict`, `PrivateTmp`, `NoNewPrivileges`) — which matter most for the `vips`
      subprocess [14.4](14-security.md) calls the largest attack surface in the system.
- [x] 12.3 CI/CD: where, which steps (lint, tests, migrations, deploy), release strategy.
      **Decision: GitHub Actions; five checks on every push; automatic deploy of every green commit
      on `main` by symlink swap, with the previous release kept for rollback.**
      **GitHub Actions**, because [13.2](13-legal.md) puts the repository on GitHub publicly and a
      second CI account would be a second thing to configure and explain. **GitHub is not a
      sub-processor** for [13.3](13-legal.md)'s purposes and the distinction is worth stating: it
      holds source code and build artifacts, never user data. [13.3](13-legal.md)'s list stays at
      three infrastructure providers.
      **On every push and pull request:** `build` (`-Werror`, [12.7](12-infrastructure.md)),
      `format` (`fourmolu --mode check`), `lint` (`hlint`), and `test` — all three of
      [11.10](11-stack.md)'s levels, with **a real Postgres service container** (level 2 is
      meaningless without one) and MinIO for the storage adapter. Plus **weekly**, the restore script
      against a dump of the development fixtures, which is [9.3](09-nfr.md)'s point 3 and the thing
      that catches a rotted backup script without touching production data.
      **Caching, because Haskell CI is otherwise unusable:** `~/.cabal/store` and `dist-newstyle`
      keyed on the GHC version and the freeze file. Stated honestly — a cold build is many minutes,
      and a dependency bump pays that. The mitigation is the cache and the fact that
      [1.11](01-product.md) has no deadline.
      **Deploy, on green `main` only:**
      1. Build the artifact in the builder container ([12.2](12-infrastructure.md)).
      2. `scp` it to `/opt/musilka/releases/<git-sha>/` — binary, static assets, migration set
         (embedded in the binary anyway, [11.8](11-stack.md)).
      3. Flip the `current` symlink and `systemctl restart musilka`.
      4. Poll `/healthz` ([9.5](09-nfr.md)); on failure, flip the symlink back and restart.
      **Migrations are not a deploy step**, and this is the one place that surprises people:
      [11.8](11-stack.md) applies them at startup inside a transaction, so **the restart is the
      migration**. [9.2](09-nfr.md)'s rule therefore binds CI as much as the operator — any migration
      expected to take more than a few seconds is run **by hand against production before** the
      deploy that needs it, leaving the startup migration a no-op. A deploy pipeline that does not
      know this will one day hold the site down for the length of a table rewrite.
      **Rollback is the symlink**, plus the last five releases kept on disk. A rollback does **not**
      undo a migration ([11.8](11-stack.md) is forward-only): it recovers from a bad binary, and a
      bad migration is corrected forward or restored from [9.3](09-nfr.md).
      **Release strategy: none, deliberately.** No tags, no version numbers, no changelog, no release
      notes — every green commit on `main` is the release, and the audience for a changelog is one
      person who wrote the commits. `workflow_dispatch` exists for a manual redeploy or a forced
      rollback.
      **The deploy credential** is an SSH key held in Actions secrets, for a `deploy` user that may
      write only under `/opt/musilka/releases` and restart the one unit through a narrow sudoers
      rule. It is the only credential in the system that lets a remote service change the box, and it
      is worth keeping that small.
- [x] 12.4 Environments: local / staging / production. Do we need staging.
      **Decision: two environments, local and production. No staging.**
      **What staging would cost**: a second VPS, a second Postgres, a second set of secrets, a second
      DNS name and TLS certificate, a second bucket, and a second deploy path — all to rehearse
      against data that is not the data. **What it would catch** that CI does not is genuinely small
      here, because [11.10](11-stack.md)'s level-2 tests already run against a real Postgres and the
      e2e smoke runs the whole round trip; the residue is "does it work on the box", which a
      permanently-drifting second box answers less reliably than the box itself.
      **What replaces it is stronger than staging, and it already exists:
      [9.3](09-nfr.md)'s restore script produces a scratch copy of *real production data* on
      demand.** That is where a risky migration is rehearsed before [9.2](09-nfr.md)'s
      run-it-by-hand rule is applied for real, and it is a better rehearsal than a staging database
      full of invented rows. The restore drill is required monthly anyway; this makes it earn its
      keep twice.
      **Development never runs against production data**, with exactly one bounded exception.
      Fixtures are generated ([11.8](11-stack.md)); the database contains message bodies and email
      addresses in plaintext ([5.8](05-messaging.md), [14.1](14-security.md)), so copying it to a
      laptop for convenience would be the worst data-handling decision available in this design. The
      exception is the restore drill itself, which necessarily decrypts real data on the machine
      holding the `age` private key ([9.3](09-nfr.md) keeps it off the box): it is a drill, not an
      environment, the scratch database is dropped when it ends, and nobody develops against it.
      **Local is not a small production and does not pretend to be:** no TLS, no nginx, `cabal run`
      against compose ([12.2](12-infrastructure.md)). The differences that matter — the proxy caps
      ([9.4](09-nfr.md)) and TLS-only cookies ([14.2](14-security.md)) — are the two things to
      remember are untested locally, which is a shorter list than a staging environment's drift.
- [~] 12.5 Domain, TLS, email (transactional emails — which provider).
      **Decision: TLS and the reverse proxy are settled; the domain name, the registrar and the mail
      provider are deferred with [12.1](12-infrastructure.md).**
      **nginx, with certbot for Let's Encrypt**, both from the distribution's packages. nginx rather
      than Caddy for one concrete reason: [9.4](09-nfr.md) requires a per-IP request cap and body
      caps at the edge, and nginx has `limit_req` and `client_max_body_size` built in, while Caddy
      needs a plugin and therefore a custom build — which is the same objection
      [11.4](11-stack.md) made to extensions needing a custom Postgres image. Caddy's automatic TLS
      is the nicer half of that trade and it loses to a stock package.
      **The proxy's job, in full**, since it is where several closed sections land: HTTP → HTTPS
      redirect; a generous per-IP request cap aimed at a scanner ([9.4](09-nfr.md)); body caps of
      15 MB on the image upload route, 20 MB on the import route and something small everywhere else
      ([14.4](14-security.md)); request and slow-body timeouts; and pass-through to the unix socket
      or loopback port the application listens on. **Security headers and CSP stay in the
      application** ([14.3](14-security.md)) — one place, versioned with the templates they
      constrain, rather than split between a config file and middleware. HSTS is the exception worth
      naming: it is set by the application but only meaningful because the proxy terminates TLS.
      **Deferred: the domain name, the registrar and the transactional mail provider** — 2026-08-15,
      at the author's request, together with [12.1](12-infrastructure.md).
      **What holds regardless of who the sender turns out to be**, and what
      [NOTES.md](NOTES.md) has flagged since 2026-08-14 as the item that quietly breaks a load-bearing
      decision:
      - A fresh domain sending from a new IP with no **SPF, DKIM and DMARC** lands
        [section 6](06-accounts.md)'s four templates in spam, and that breaks
        [6.1](06-accounts.md)'s verified-email barrier — which [4.9](04-editing.md),
        [5.7](05-messaging.md) and [14.5](14-security.md) all lean on instead of a captcha. **Publish
        all three records in the same sitting as the first deploy setup**, DMARC at `p=none` moving
        to `p=quarantine` once the reports are clean, and give the domain reputation time before
        [1.10](01-product.md)'s stranger is invited.
      - Verify deliverability against a real inbox at a large provider before that invitation, not
        after. It is a ten-minute check that protects the one criterion outside our control.
      **Amended 2026-08-15 by [1.10](01-product.md).** Criterion 2 is withdrawn and the public
      deployment is deferred, so **the invitation these two bullets are timed against no longer has
      a date** — but neither bullet is cancelled and neither gets cheaper by waiting. They move to
      the deployment, as a unit, with the rule intact: register the domain and publish all three
      records *in the same sitting* as the first deploy setup.
      **What replaces the sender locally:** a mail catcher (Mailpit or equivalent) behind the same
      port [6.1](06-accounts.md)'s templates already send through. That exercises the four
      templates, the queue job and the verification flow honestly; it exercises deliverability not
      at all, and the difference must not be blurred — a green local flow says nothing about inbox
      placement.
      - The sender must be in the EU/EEA like everything else ([13.3](13-legal.md)) and becomes the
        third entry on [13.3](13-legal.md)'s sub-processor list.
      - **We never run a mail server.** Outbound goes through the provider's SMTP or API; inbound is
        forwarding only — [13.5](13-legal.md) requires a published abuse address, so `abuse@` and
        `privacy@` forward to the operator's own mailbox and nothing on the box listens on port 25.
      **TLS for the image bucket** is the provider's ([8.2](08-media.md) serves from the bucket's own
      host, which [14.4](14-security.md) wants for origin separation); no certificate of ours is
      involved.
- [x] 12.6 Secrets and configuration management.
      **Decision: environment variables from a root-owned `EnvironmentFile`, decoded once at startup
      into a typed config; six secrets, one of which we design away entirely; no secret manager.**
      **Mechanism.** systemd `EnvironmentFile=/etc/musilka/env`, owned by root, mode `0600`, never in
      git. The application decodes **every** value — secret and not — into a typed `Config` at
      startup and **refuses to boot on anything missing or malformed** rather than defaulting; that
      is [11.9](11-stack.md)'s parse-don't-validate applied to the configuration edge, and it turns a
      typo into a failed deploy that rolls back ([12.3](12-infrastructure.md)) instead of a runtime
      surprise at 3am. Nothing from the config is ever logged ([14.6](14-security.md)).
      **The complete secret list, which is short on purpose:**
      | Secret | Note |
      |---|---|
      | Image bucket credentials | Read/write on the public prefix ([8.2](08-media.md)) |
      | Backup bucket credentials | Separate provider, separate credential, write-only if the provider supports it ([9.3](09-nfr.md)) |
      | Mail sender credentials | [12.5](12-infrastructure.md) |
      | Deploy SSH key | Held by GitHub Actions, not by the box ([12.3](12-infrastructure.md)) |
      **The database password does not appear, and that is deliberate.** Postgres runs on the same
      box and listens only on the unix socket ([12.1](12-infrastructure.md)); the application
      connects as the `musilka` system user under **peer authentication**. The most commonly leaked
      credential in any deployment simply does not exist here.
      **Nor does a cookie-signing key**, which is worth noticing because most stacks have one:
      [11.6](11-stack.md)'s session is an opaque random token stored hashed
      ([14.2](14-security.md)) and [14.3](14-security.md)'s CSRF token lives on the session row —
      there is nothing to sign, so there is no signing secret to rotate or leak.
      **The `age` public key lives on the box and is not a secret; the private key never touches
      it** — password manager plus paper, per [9.3](09-nfr.md), with the trade stated there: a lost
      key means lost backups.
      **No Vault, no SOPS, no sealed secrets, no cloud secret manager.** Each is a service to run or
      an account to hold, bootstrapped by a secret of its own, to manage four values on one box —
      [1.4](01-product.md)'s veto list in spirit. **Rotation is by hand and documented in a runbook**
      in the repository, alongside the restore procedure ([9.3](09-nfr.md)); undocumented rotation is
      rotation that never happens.
- [x] 12.7 Working style with the code: git-flow/trunk, reviews, commit conventions, linters/formatters.
      **Decision: trunk-based on `main`; the repository's existing commit convention kept; no code
      review, with CI and [14.3](14-security.md)'s greps standing in for it; formatting and linting
      enforced in CI so neither is ever a judgement call.**
      **Trunk-based.** One author ([1.11](01-product.md)) and one deployable
      ([11.1](11-stack.md)) make git-flow's release and develop branches pure ceremony. Small changes
      go straight to `main`; anything that would be an unreadable single commit gets a short-lived
      branch and a self-merged pull request — mostly so the diff is reviewable *later*, by the reader
      [1.1](01-product.md) is written for. **Branch protection requiring green CI stays on even for
      one person**: it is what stops a rushed push from deploying ([12.3](12-infrastructure.md)).
      **Commit convention: the one this repository already uses** — `Area: imperative summary`, as in
      `Design: close section 9 (non-functional requirements)` — with the body explaining *why* and
      **citing the design item** (`per 10.4.1`), the same backlink rule
      [15](15-roadmap.md)'s notes impose on plan tasks. **Conventional Commits declined**: `feat:` /
      `fix:` exist to drive changelog generation and semantic versioning, and
      [12.3](12-infrastructure.md) has neither a version nor a changelog.
      **No review, and what replaces it.** There is nobody to review. The substitutes are CI on every
      push and a self-check on any diff touching the five things [14.3](14-security.md) enumerates —
      `toHtmlRaw`, a hand-written `<form>`, a user-scoped query without its actor clause, a new
      outbound request, a log line carrying something from [14.6](14-security.md)'s list. Those five
      are `grep`s precisely so that a solo author can run a review on themselves.
      **Tooling, all enforced in CI ([12.3](12-infrastructure.md)):** `fourmolu` with a checked-in
      config (formatting is settled once, by a tool, and never discussed), `hlint` with a small
      `.hlint.yaml`, `-Wall` in the cabal files with `-Werror` added in CI only — warnings should
      block a deploy without making a local experiment miserable — and `cabal check`.
- [x] 12.8 Issue tracking: where we track work (issues, Linear, plain markdown in the repo).
      **Decision: `plan/*.md` in the repository is the backlog and the single source of truth.
      GitHub Issues stays enabled as an inbox for reports from outside, and is not the backlog.**
      This is the item [15](15-roadmap.md)'s working notes said to settle first, and it settles that
      way.
      **Why markdown in the repository**, in the order the reasons actually matter:
      1. **The backlinks work.** [15](15-roadmap.md) requires every task to cite the design item
         justifying it (`T-14 external_ids table (per 10.4.1)`), and
         `grep -rn "10.4.1" design/ plan/` finding both the decision and the work is the property
         that makes this whole repository navigable. An external tracker cuts that grep in half.
      2. **Progress is derived, never stored** — the same `grep -c` discipline `CLAUDE.md` already
         uses for `design/`, with no status field to keep in sync.
      3. It versions with the code, so a task and the commit implementing it share a history, and a
         reader of the portfolio piece ([1.1](01-product.md)) sees design, plan and code side by
         side rather than behind a login.
      **Why Issues stays on anyway.** [1.10](01-product.md)'s second success criterion is a stranger
      using the site, and a stranger needs somewhere to report a bug that is not the author's inbox.
      Issues is that place.
      **2026-08-15:** that criterion is withdrawn, so this reason is now weaker but not gone —
      [13.2](13-legal.md) keeps the repository public from the first commit, and a reader of the code
      is still someone who may want to report something. Issues costs nothing to leave enabled and
      the rule that matters is unchanged: **it is an inbox, never the backlog.** **Anything acted on becomes a task in `plan/` citing the issue** — the
      inbox is not a second backlog, and the moment two lists both claim to be the truth, neither is.
      **Declined:** Linear, Jira and every hosted tracker (an account and a second surface for a
      solo project, and [1.6](01-product.md) means nothing pays for it); Issues as the backlog
      (loses the grep and the version history); a project board (a status field by another name, and
      `CLAUDE.md`'s rule is that status is derived).
      **This unblocks [15.6](15-roadmap.md)**, whose shape was waiting on the answer: `plan/` is
      real, cut by stage rather than by design section, and its file names come from
      [15.1](15-roadmap.md).

## Working notes

- **2026-08-15 — Section closed: 6 items decided, 2 deferred by choice.**
  [12.1](12-infrastructure.md) and [12.5](12-infrastructure.md) carry `[~]` because the hosting
  provider and the mail sender were postponed, not because anything is waiting on them
  analytically — the shape of both is fully specified and only a vendor name is missing. Everything
  else in the section is vendor-independent by construction, which is why it could be closed around
  the hole.
- **2026-08-15 — What the two open names actually block.** Not a design decision anywhere: the first
  deploy, and through it DNS, TLS issuance and transactional mail. The one with teeth is mail —
  [NOTES.md](NOTES.md) has flagged since 2026-08-14 that a fresh domain without SPF/DKIM/DMARC puts
  [6.1](06-accounts.md)'s verification mail in spam, and [6.1](06-accounts.md)'s barrier is what
  [4.9](04-editing.md), [5.7](05-messaging.md) and [14.5](14-security.md) all lean on in place of a
  captcha. [15](15-roadmap.md) carries it as an E0 task with that dependency stated.
- **2026-08-15 — What this section hands on.**
  [Section 15](15-roadmap.md) inherits a concrete E0: repository layout, compose file, CI with its
  five checks, the deploy script and symlink rollback, the systemd unit, nginx with
  [9.4](09-nfr.md)'s caps, certbot, the backup job and [9.3](09-nfr.md)'s first restore drill, the
  runbook, and `plan/` itself ([12.8](12-infrastructure.md)).
  [13.3](13-legal.md) inherits a clarification rather than a fact: GitHub is not a sub-processor
  ([12.3](12-infrastructure.md)), so the list stays at three infrastructure providers once
  [12.1](12-infrastructure.md) and [12.5](12-infrastructure.md) name them.
  [11.11](11-stack.md)'s open Nix question is answered: no ([12.2](12-infrastructure.md)).
- **2026-08-15 — The two decisions most likely to be regretted, and the honest reading of each.**
  **No staging** ([12.4](12-infrastructure.md)) — the day a migration goes wrong on production it
  will feel like the cause; the mitigation is [9.2](09-nfr.md)'s hand-run rule plus a restore-based
  rehearsal, and adding a staging box later costs a day. **Automatic deploy on green `main`**
  ([12.3](12-infrastructure.md)) — fine while the tests are honest, and the moment they are not, the
  fix is the tests. Neither is expensive to reverse, which is why both were decided the fast way.
- **2026-08-15 — What was declined, in one place.** PaaS of every kind and self-hosting at home;
  Docker on the server; Nix; a staging environment; tags, versions, changelogs and release notes;
  Vault/SOPS/any secret manager; a database password (designed away via peer auth); a cookie-signing
  key (designed away by [11.6](11-stack.md)); Caddy (for `limit_req` alone); Conventional Commits;
  code review by anyone; Linear/Jira and every hosted tracker; and a project board.

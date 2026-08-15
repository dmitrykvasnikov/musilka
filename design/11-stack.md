# 11. Technology stack

**Priority:** P0 — but decided after sections 1–5

- [x] 11.1 Architecture: monolith with server-side rendering / monolith API + SPA / an SSR framework (Next, Nuxt, Remix) / splitting into services (not recommended at the start). (see also [7.8](07-search-ux.md), [10.1.12](10a-public-api.md))
      **Decision:** A **monolith with server-side rendering** — one process, one binary, HTML rendered
      on the server. Not an API + SPA, not an SSR framework, not services.
      This is less a choice than a reading of decisions already taken. [1.4](01-product.md) fixes the
      target shape as one small VPS, one process, deploys that may drop the site for ten seconds.
      [1.9](01-product.md) puts the public API out of the MVP, so an API + SPA split would mean
      building and versioning a JSON layer whose only consumer is our own frontend. NOTES' *no
      real-time transport anywhere* rules out the live-update story that usually justifies a client
      application. And [1.8](01-product.md)'s English-only interface removes the last routine reason
      to own a client-side rendering pipeline.
      **Internal shape, already fixed by [1.12](01-product.md):** `ReaderT AppEnv IO`, settled on
      immediately, effect systems explicitly out of scope. The layering that goes with it is
      [11.11](11-stack.md)'s.
      **What this gives [7.8](07-search-ux.md) for free:** every public catalogue page is complete
      HTML on first response, so whichever way the SEO question is decided, the stack does not have to
      change to accommodate it. 7.8 remains a question about *robots and canonical URLs*, not about
      rendering.
- [x] 11.2 Backend: language and framework (Python/Django|FastAPI, Node/NestJS, Go, Rust/Axum, Elixir/Phoenix, Ruby/Rails, Java/Kotlin…). Take [1.12](01-product.md) into account.
      **Decision: Haskell, with Servant + warp + Lucid.** [1.12](01-product.md)'s preference is
      confirmed rather than revisited, and the fallback is not exercised.
      **Why the fallback was not taken.** [1.12](01-product.md) named Elixir/Phoenix as the
      alternative, and NOTES records what happened to its case: section 5 refused real time outright
      ([5.3](05-messaging.md), [5.9](05-messaging.md)), so OTP, LiveView and presence — the things
      that would have made Elixir the *better* answer rather than merely the faster one — have no
      consumer anywhere in this design. What is left of the Elixir argument is that it ships sooner,
      and [1.11](01-product.md) has already bought that time with no deadline and a 2–3× multiplier
      accepted in advance.
      **Servant, and what it is bought for.** [1.12](01-product.md) already budgets Servant's
      type-level error messages as a known gap, which is the tell that it was the expected choice. The
      API-as-a-type pays twice: `servant-lucid` renders HTML today, and if
      [10.1](10a-public-api.md) ever ships a public API it is a second set of endpoints in the same
      vocabulary with OpenAPI *generated* rather than hand-maintained ([11.5](11-stack.md)).
      Considered and declined: **Yesod**, which is the batteries-included shape [1.12](01-product.md)
      rejected on grounds of being unappealing to work in, and heavily Template-Haskell-driven besides;
      **Scotty/wai directly**, which is genuinely pleasant for an HTML-first app but gives up the
      type-level description and leaves [10.1](10a-public-api.md) with nothing to build on.
      **On reopening.** The fallback in [1.12](01-product.md) is not deleted, but switching language
      now means restarting the codebase, so it is invoked — if ever — on evidence from a built E0/E1
      (the vertical is deployed and the work is visibly not converging), never on a bad week early on
      while the four gaps [1.12](01-product.md) itemises are still being closed.
- [x] 11.3 Frontend: React/Vue/Svelte/HTMX+templates; UI kit (Tailwind + shadcn, Mantine, MUI…).
      **Decision: Lucid templates + HTMX + hand-written CSS. No UI kit, and no Node toolchain in the
      repository at all.**
      Lucid because typed HTML in Haskell needs no second template language, no template compiler and
      no build step — the pages are ordinary functions, checked by the same compiler as everything
      else, and `servant-lucid` joins them to [11.2](11-stack.md) directly.
      HTMX for the handful of places that genuinely want an interaction rather than a page load —
      search-as-you-type ([section 7](07-search-ux.md)), adding a tag ([3.3](03-collection.md)),
      upload progress ([section 8](08-media.md)), import job status
      ([10.4.5](10d-model-requirements.md)). It is one vendored, version-pinned file served from our
      own static directory: no CDN ([1.4](01-product.md) vetoes it, and a third-party script is
      [section 14](14-security.md)'s problem for no gain).
      **Every core flow must work with JavaScript disabled.** Forms post normally and redirect;
      HTMX is an enhancement over a working page, never the mechanism. This is what keeps
      [7.8](07-search-ux.md) free to decide anything it likes, and it is also the cheapest possible
      accessibility and robustness story.
      **Tailwind, and every UI kit, declined.** They solve a real problem — inventing CSS is slow —
      but they drag a Node toolchain, a `package.json`, a CSS build and a second CI step into a
      Haskell repository, purely for styling, and they put a second build artifact in front of a
      deploy that [11.11](11-stack.md) works to keep at one. shadcn/Mantine/MUI are React-shaped and
      are excluded by [11.1](11-stack.md) anyway. One hand-written stylesheet, plain modern CSS
      (custom properties, grid, `light-dark()`), no preprocessor.
      **An SPA was declined at [11.1](11-stack.md)**, not here.
- [x] 11.4 Database: PostgreSQL by default — agreed? Anything else (Redis for cache/sessions/queues)?
      **Decision: PostgreSQL, and nothing else. It is the only datastore in the system.**
      Agreed on the default, and the more useful half of the answer is the second question: **no
      Redis, and no second store of any kind.** [1.4](01-product.md) already vetoes it by name; this
      item records what Postgres is therefore doing, so that no later section quietly reintroduces a
      dependency by needing one of these:
      | Need | Where it lives | Fixed by |
      |---|---|---|
      | Catalogue, collections, messages | Tables | [section 2](02-catalogue-model.md)–[5](05-messaging.md) |
      | Search | FTS + `pg_trgm` | [1.4](01-product.md), [section 7](07-search-ux.md) |
      | Job queue | A table, `SKIP LOCKED` | [11.7](11-stack.md), [10.4.5](10d-model-requirements.md) |
      | Sessions | A table | [11.6](11-stack.md), [14.3](14-security.md) |
      | Rate limits | A table | [14.5](14-security.md) |
      Images are the one thing that is *not* in Postgres — [1.4](01-product.md) argues for object
      storage from the start and [section 8](08-media.md) owns it. Nothing else leaves the database.
      **Version and extensions:** one current major (17 or 18), pinned identically across dev, CI and
      production — [section 12](12-infrastructure.md) picks the number when it picks the host. Only
      extensions shipped with a stock Postgres, `pg_trgm` today; nothing that needs a custom image,
      because [section 12](12-infrastructure.md) should be able to use the distribution's package.
      **The database layer** is `hasql` with hand-written SQL. [1.12](01-product.md) flags this as the
      area with no prior equivalent, which argues for the option with the fewest new concepts and the
      most transferable knowledge: statements are ordinary values — composable, individually testable —
      the explicit codec is exactly the boundary [11.9](11-stack.md) wants, and no Template Haskell is
      required. The cost is codec boilerplate across roughly fifteen tables, accepted.
      Considered and declined: **`postgresql-simple`**, the closest call, marginally less ceremony but
      a looser row boundary and no unit of work as tidy as a `Statement`; **`rel8`/`opaleye`**, a
      typed relational DSL that is impressive and would remove the boilerplate, but stacks a real
      learning cliff on an area with zero prior experience and introduces a second schema definition
      beside the migrations; **`persistent`/`esqueleto`**, declined for the same reasons as Yesod at
      [11.2](11-stack.md). All queries sit behind the service layer
      ([10.4.4](10d-model-requirements.md)), so this is the most reversible decision in the section.
- [x] 11.5 API style: REST / GraphQL / tRPC. Will there be a public API for third-party clients? (requirements — [section 10.1](10a-public-api.md); note that tRPC is a poor fit for a public API outside TypeScript)
      **Decision: REST, if and when there is an API at all — and there is none in the MVP.**
      **Whether** a public API exists is [10.1.1](10a-public-api.md)'s question and is not answered
      here. [1.9](01-product.md) puts it out of the MVP; this item only fixes the shape it would take.
      **REST**, because both alternatives are excluded by decisions already taken. **tRPC** presumes a
      TypeScript client, and [11.3](11-stack.md) has no TypeScript anywhere. **GraphQL** is a second
      query engine, a resolver layer and an N+1 problem, bought so that clients can shape their own
      responses — against ~12,000 releases ([1.4](01-product.md)) and, at present, zero third-party
      clients. REST over the Servant API type also means the OpenAPI document is *generated*, which is
      most of [10.1.9](10a-public-api.md) done for free.
      **The frontend does not call an API, and this is deliberate.** Pages render from the service
      layer directly ([10.4.4](10d-model-requirements.md)); there is no JSON hop in the middle. The
      reuse boundary between the UI and any future API is therefore **the service layer, not HTTP** —
      which is what makes the rights model ([4.7](04-editing.md)), the verified-email barrier
      ([6.1](06-accounts.md)) and collection privacy ([3.7](03-collection.md)) automatically hold for
      both, as those items each require. [10.1.12](10a-public-api.md) should be read against this: the
      answer to "does the frontend dogfood the API" is that there is nothing to dogfood, and the
      protection it was reaching for is obtained lower down instead.
- [x] 11.6 Authentication: server-side sessions (cookie) vs JWT; an off-the-shelf solution (Auth.js, Keycloak, Supabase Auth) vs our own.
      **Decision: server-side sessions — an opaque random token in an `HttpOnly` cookie, session state
      in Postgres — implemented ourselves.**
      **JWT is not a close call here; it is excluded by [6.2](06-accounts.md).** Password reset and
      email change must invalidate *every* session. A stateless token cannot be revoked, so honouring
      that requirement means adding a denylist or a token-version check on every request — at which
      point the database round trip is back, the session table exists in all but name, and the only
      thing gained is a harder-to-reason-about failure mode.
      **Off-the-shelf declined.** Auth.js is Node, Keycloak is a JVM service beside our one process
      ([1.4](01-product.md)), Supabase Auth is a hosted dependency on the critical path of logging in.
      All three exist chiefly to solve federation, and [6.1](06-accounts.md) rejected OAuth outright —
      there is nothing to federate. What is left is email+password, one hash ([14.2](14-security.md))
      and a token table that [6.x](06-accounts.md) already specified.
      **This settles the question [06's working notes](06-accounts.md) left open** — there *is* a
      session table. Its columns, lifetime, rotation, cookie attributes and CSRF handling remain
      [14.3](14-security.md)'s to fix; note that [11.3](11-stack.md)'s plain HTML forms mean CSRF
      tokens are hand-rolled and must be a template concern, not an afterthought.
- [x] 11.7 Background jobs and queues (import, emails, image resizing): are they needed, and with what? (collection import/export makes them mandatory earlier than it seems — see [10.4.5](10d-model-requirements.md))
      **Decision: yes, mandatory from E1, and it is a Postgres table with `SELECT … FOR UPDATE SKIP
      LOCKED`, hand-written, running in-process.**
      Not an open question: [1.9](01-product.md) names the job queue as one of three things that must
      exist in E0–E1 even though the features they serve come later, because retrofitting it is the
      most expensive rework available. Known consumers: import ([10.2](10b-import.md)), export
      generation ([10.3](10c-export.md)), outbound email (four templates,
      [section 6](06-accounts.md)), and image derivatives if [section 8](08-media.md) needs them.
      **No Redis, no RabbitMQ, no external scheduler** ([1.4](01-product.md)). At this volume the
      queue is one table, a claim query and a retry column; a broker would be a second daemon to run,
      monitor and back up on a box sized for one.
      **In-process worker, and the consequence stated plainly:** a deploy kills whatever is running
      ([1.4](01-product.md) accepts ten seconds of downtime), so **every job must be restartable** —
      idempotent, or resumable from its own progress record. That is a design constraint on
      [10.2](10b-import.md)'s importer, not a footnote. Moving the worker to a second process later
      needs no schema change, so it is available if [section 12](12-infrastructure.md) wants it.
      **Failed jobs stay in the table** with an attempt count and the last error, visible rather than
      dropped — the same reasoning as NOTES' *an import never discards data silently*: a job that
      vanished is indistinguishable from a bug.
- [x] 11.8 Schema migrations, seed data, development fixtures.
      **Decision: numbered plain-SQL files, forward-only, embedded in the binary and applied at
      startup against a `schema_migration` table.**
      Plain SQL because [11.4](11-stack.md) already declined the DB layers that would have generated
      it, and for the same reason: one schema definition, written in the language the database
      actually speaks. Embedded via `file-embed` and applied by a small runner at startup because
      [11.11](11-stack.md) is working to keep the deploy at exactly one artifact — an external
      migration tool would be a second binary to install and version on the box.
      **Forward-only, no down-migrations.** A down-migration for anything that touched data is a
      fiction; a mistake is corrected by the next forward migration, and the safety net is
      [section 12](12-infrastructure.md)'s tested restore, which [1.4](01-product.md) already insists
      on.
      **Three distinct kinds of data, and they must not be conflated:**
      1. **Migrations** — schema, versioned, applied everywhere including production.
      2. **Seed data** — the curated vocabularies NOTES lists as a real roadmap task: genres, styles,
         credit roles, company roles, format descriptors, identifier types, country codes including
         `SU`/`YU`/`DD`/`CS`. Versioned SQL, applied idempotently, and applied in **production** too —
         the application does not work without it ([2.3.2](02-catalogue-model.md) and the rest of the
         *curated set* invariant). Its contents are [section 15](15-roadmap.md)'s to schedule.
      3. **Development fixtures** — a few users, a few hundred releases, an import in each state. One
         command, never run in production, and generated rather than hand-written so it can grow.
- [x] 11.9 Typing/validation at the boundaries, a shared schema for frontend and backend.
      **Decision: parse, don't validate — decode into domain types once at each edge, and never
      re-check downstream. There is no shared frontend/backend schema, because there is no separate
      frontend.**
      Domain types are newtypes with smart constructors in `musilka-domain` ([11.11](11-stack.md)); a
      value that exists is a value that is valid. The edges that must decode are: HTTP forms
      ([11.3](11-stack.md)), CSV on import (`cassava`; the Discogs specifics — comma-packed fields and
      commas inside label names, per NOTES — are [10.2](10b-import.md)'s), the database
      ([11.4](11-stack.md)'s codecs), and configuration at startup.
      **Decode failures are data, not exceptions.** Each edge yields either a domain value or a list
      of user-facing errors, because NOTES' *an import never discards data silently* requires the
      importer to **report** what it could not place — which is only possible if failures are values
      that survive to the end of the run.
      **The "shared schema" the item anticipates is moot** under [11.1](11-stack.md): the templates
      are compiled against the same types as the handlers, which is the strongest version of that
      guarantee available. It would only reappear if [10.1](10a-public-api.md) ships third-party
      clients, and there the artifact is an OpenAPI document generated from the Servant API type
      ([11.5](11-stack.md)) — never a hand-maintained schema to keep in sync.
- [x] 11.10 Testing: levels (unit/integration/e2e), target coverage, tools.
      **Decision: three levels, no coverage percentage, `hspec` + `hedgehog` + a real Postgres.**
      **No numeric coverage target, deliberately.** [1.1](01-product.md) sets the quality bar as
      something a reader of the repository judges, and a percentage is the one measure of that which
      is gameable by testing getters. At [1.11](01-product.md)'s pace a target becomes a chore that
      displaces real tests. What replaces it is a named list of things that must be covered:
      1. **Property tests on the pure domain** (`musilka-domain`, no IO): CSV field splitting;
         merge and unmerge as inverse operations ([4.4](04-editing.md)); an aggregate snapshot
         reconstructing its entity ([4.2](04-editing.md)); **export → import → export producing the
         same state**, which is [1.10](01-product.md)'s first success criterion turned into a test
         that runs on every push rather than a thing checked once by hand.
      2. **Integration tests per service-layer operation** against a **real Postgres** — a disposable
         database per run, never a mock, since [11.4](11-stack.md) writes SQL by hand and a mocked
         database would test nothing that can break. These carry the rights model
         ([4.7](04-editing.md)), the verified-email barrier ([6.1](06-accounts.md)) and privacy
         ([3.7](03-collection.md)), all of which are service-layer rules by decision.
      3. **A deliberately thin e2e smoke** over the round trip of [1.9](01-product.md): register,
         verify, import, edit, export. Enough to catch a broken wiring or template, not a second test
         suite to maintain.
      **Tools:** `hspec` as the runner with `hspec-hedgehog` for properties (`tasty` is the equivalent
      alternative and nothing turns on it). CI runs all three levels on every push — [1.1](01-product.md)
      requires CI, and level 2 is the reason [section 12](12-infrastructure.md) needs a Postgres
      service in the CI job.
- [x] 11.11 Monorepo or several repositories; package manager.
      **Decision: one repository; one Cabal project with three internal libraries; Cabal with a pinned
      `index-state`; no Node package manager at all.**
      **One repository** — there is one deployable ([11.1](11-stack.md)) and one author
      ([1.11](01-product.md)); several repositories would buy nothing and cost a synchronised commit
      on every change.
      **Three libraries, so that the dependency direction is enforced by the build rather than by
      convention:**
      | Library | Contains | May depend on |
      |---|---|---|
      | `musilka-domain` | Entities, vocabularies, merge rules, CSV parsing. Pure — no IO, no SQL. | — |
      | `musilka-app` | The service layer ([10.4.4](10d-model-requirements.md)), ports, job definitions. | `domain` |
      | `musilka-web` | Servant API, Lucid templates, sessions, the job runner. | `domain`, `app` |
      plus a thin executable that wires them. `domain` cannot import `app`; `app` cannot import `web`;
      the compiler enforces it. This is not ceremony — it is the specific claim
      [1.1](01-product.md) makes about clean architecture, made checkable, and it is what keeps
      [11.10](11-stack.md)'s level 1 honestly pure and [11.4](11-stack.md)'s DB choice reversible.
      **Cabal**, with GHC and the package index pinned (`cabal.project` + `index-state`, one GHC
      version, identical in CI). Stack adds a second resolver concept for no gain here. **Nix is not
      decided here** — it is a build-and-deploy question and belongs to
      [section 12](12-infrastructure.md), which may adopt it or use a multi-stage container; nothing
      in this section depends on the answer.
      **No `package.json`, no npm/pnpm, no bundler** — [11.3](11-stack.md)'s consequence, restated
      here because it is a repository-layout fact: the only static assets are one stylesheet and one
      vendored HTMX file, committed as they are served.

## Working notes

- **2026-08-15 — What this section hands on.**
  [Section 12](12-infrastructure.md) inherits the most: a single binary plus static assets as the
  deploy artifact ([11.11](11-stack.md)), migrations applied at startup ([11.8](11-stack.md)), a
  GHC toolchain and a Postgres service in CI ([11.10](11-stack.md)), the Nix-or-container question
  ([11.11](11-stack.md)), a stock Postgres with `pg_trgm` ([11.4](11-stack.md)), and the fact that a
  deploy kills running jobs ([11.7](11-stack.md)).
  [Section 14](14-security.md) inherits session and cookie mechanics ([11.6](11-stack.md)) and, less
  obviously, **CSRF tokens as a template concern** — plain HTML forms mean there is no framework
  doing it silently.
  [Section 7](07-search-ux.md) inherits `pg_trgm` availability and the guarantee that any page can be
  server-rendered, so [7.8](07-search-ux.md) is free either way.
  [Section 8](08-media.md) inherits that image processing happens in-process or via a system binary —
  there is no JS pipeline available to it ([11.3](11-stack.md)).
  [Section 10.1](10a-public-api.md) inherits REST, a generated OpenAPI document, and the rule that
  the reuse boundary is the service layer rather than HTTP ([11.5](11-stack.md)).
  [Section 15](15-roadmap.md) inherits [1.12](01-product.md)'s four gaps as real roadmap items —
  `ReaderT AppEnv IO` in the large, `hasql`, Servant's error messages, and build/deploy — plus the
  vocabulary seed data ([11.8](11-stack.md)).
- **2026-08-15 — The two calls most worth revisiting, and the two that are not.** `hasql` versus
  `postgresql-simple` ([11.4](11-stack.md)) was close, and the service layer makes it cheap to
  change; `hspec` versus `tasty` ([11.10](11-stack.md)) is arbitrary. By contrast, Servant
  ([11.2](11-stack.md)) and the three-library split ([11.11](11-stack.md)) get harder to reverse with
  every vertical, and the no-Node rule ([11.3](11-stack.md)) is the kind of line that only holds if
  it is never crossed once.
- **2026-08-15 — Parallel Haskell and Elixir branches: considered and declined.** Raised as a way to
  keep [1.12](01-product.md)'s fallback alive by building both. Declined because every decision in
  `design/` would be implemented twice at [1.11](01-product.md)'s pace, the branches would diverge
  with no way to merge across languages, and [1.10](01-product.md)'s success criteria need one
  deployed thing that a stranger can use. A timeboxed bakeoff spike — one vertical slice in each,
  loser deleted — was the defensible version and was also declined, in favour of committing now.

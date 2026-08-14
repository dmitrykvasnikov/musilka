# 11. Technology stack

**Priority:** P0 — but decided after sections 1–5

- [ ] 11.1 Architecture: monolith with server-side rendering / monolith API + SPA / an SSR framework (Next, Nuxt, Remix) / splitting into services (not recommended at the start). (see also [7.8](07-search-ux.md), [10.1.12](10a-public-api.md))
- [ ] 11.2 Backend: language and framework (Python/Django|FastAPI, Node/NestJS, Go, Rust/Axum, Elixir/Phoenix, Ruby/Rails, Java/Kotlin…). Take [1.12](01-product.md) into account.
- [ ] 11.3 Frontend: React/Vue/Svelte/HTMX+templates; UI kit (Tailwind + shadcn, Mantine, MUI…).
- [ ] 11.4 Database: PostgreSQL by default — agreed? Anything else (Redis for cache/sessions/queues)?
- [ ] 11.5 API style: REST / GraphQL / tRPC. Will there be a public API for third-party clients? (requirements — [section 10.1](10a-public-api.md); note that tRPC is a poor fit for a public API outside TypeScript)
- [ ] 11.6 Authentication: server-side sessions (cookie) vs JWT; an off-the-shelf solution (Auth.js, Keycloak, Supabase Auth) vs our own.
- [ ] 11.7 Background jobs and queues (import, emails, image resizing): are they needed, and with what? (collection import/export makes them mandatory earlier than it seems — see [10.4.5](10d-model-requirements.md))
- [ ] 11.8 Schema migrations, seed data, development fixtures.
- [ ] 11.9 Typing/validation at the boundaries, a shared schema for frontend and backend.
- [ ] 11.10 Testing: levels (unit/integration/e2e), target coverage, tools.
- [ ] 11.11 Monorepo or several repositories; package manager.

## Working notes

_(none yet)_

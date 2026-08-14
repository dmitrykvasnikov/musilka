# 10.4 What this already demands from the data model (P0 checklist)

**Priority:** P0 — build it into the architecture from the start ([why](#why-section-10-is-p0))
**Siblings:** [10.1 public API](10a-public-api.md) · [10.2 import](10b-import.md) · [10.3 export](10c-export.md) · [10.5 legal](10e-legal-sources.md)

## Why section 10 is P0

Even if we write the API and the importers much later, three things have to be settled **before**
designing the schema: external identifiers and data provenance (10.4.1–10.4.2), whether
"unresolved" collection items are allowed (10.4.3), and the rule that all mutations go through a
shared service layer — UI, API and importer are three callers of the same function (10.4.4).
Bolting this onto a finished database is far more expensive than designing it in now. That is why
10.1–10.4 are P0 despite being implemented late.

## Checklist

- [ ] 10.4.1 External IDs ([2.5.4](02-catalogue-model.md)): a separate table `external_ids (entity_type, entity_id, source, external_id, url, synced_at)` instead of `discogs_id`/`musicbrainz_id` columns on every entity — agreed? (more flexible, supports several sources and sync history right away)
- [ ] 10.4.2 Data provenance: do we mark where an entity/field came from (`user`, `import:discogs`, `import:musicbrainz`), and may that value be overwritten on a repeated sync?
- [ ] 10.4.3 Do we allow an "unresolved" collection item — with no `release_id`, holding raw text from the import? (see [10.2.3](10b-import.md); this is directly a schema question)
- [ ] 10.4.4 Service layer: every mutation (create a release, add to collection, change condition) is a single function called by the UI, the API and the importer alike. No domain logic in controllers or templates. Do we fix this as an architectural rule?
- [ ] 10.4.5 A background job queue is needed as early as import/export — do we build it into E1, even if we start with a primitive one (see [11.7](11-stack.md), [12.x](12-infrastructure.md))?
- [ ] 10.4.6 Public entity IDs (numeric autoincrement / UUID / slug): once they land in someone's export or in an API client, they are forever. What do we expose outwards?
- [ ] 10.4.7 The change log ([4.2](04-editing.md)) — do we expect to reuse it for incremental "what changed since X" delivery (API/webhooks/dumps)?
- [ ] 10.4.8 Storing uploaded import files: where ([8.2](08-media.md)), for how long, do we delete them after a successful apply — they contain the user's personal data.

## Working notes

_(none yet)_

# Preliminary recommendations

A starting point for the discussion, all open to challenge. These are **not** decisions — a decision
only counts once it is recorded under its item in a section file.

1. **Copy the Discogs model almost literally** (Artist / Master / Release / Label + the format structure + identifiers) — it was earned through 20 years of practice; simplify deliberately and selectively rather than by default. ([section 2](02-catalogue-model.md))
2. **Store `musicbrainz_id`/`discogs_id`** on entities from day one, even without an import — deduplication and synchronisation will then come cheap. ([2.5.4](02-catalogue-model.md), [10.4.1](10d-model-requirements.md))
3. **Build edit versioning into the schema right away** — bolting change history onto a live database is far more painful than designing with it. ([4.2](04-editing.md))
4. **PostgreSQL and its full-text search** at the start; a dedicated search engine when we hit the wall. ([7.2](07-search-ux.md), [11.4](11-stack.md))
5. **Make messages a simple mailbox with no real time** in the MVP; WebSocket can be added later without breaking the data model. ([5.2](05-messaging.md), [5.3](05-messaging.md))
6. **Collection export and CSV import from Discogs** are not a detail but the key reason a collector would try a new service at all. ([10.2](10b-import.md), [10.3](10c-export.md))
7. **The API, import and export can be implemented in E3, but three things must be built into E0–E1**: the service layer for all mutations ([10.4.4](10d-model-requirements.md)), the external-ID table with provenance ([10.4.1–10.4.2](10d-model-requirements.md)) and the background job queue ([10.4.5](10d-model-requirements.md)). Retrofitting these is the most expensive rework of everything discussed here.
8. **Be careful with expectations of the Discogs CSV**: it is essentially a list of `release_id`s with condition and notes, without release metadata. Without a metadata source (API/dump/manual linking) the import will produce a collection of empty shells — this has to be solved at the model stage, not at the importer stage. ([10.2.3](10b-import.md))
9. **Do export before import**: it is simpler, requires no external sources, and immediately gives the user the guarantee that "my data is not locked in".

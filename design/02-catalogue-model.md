# 2. Catalogue domain model

**Priority:** P0

The core of the project. This is where we decide how closely we copy Discogs.

## 2.1 Entity hierarchy

- [ ] 2.1.1 Do we confirm the chain `Artist → Album (master) → Release`? Or, in Discogs terms: `Artist → Master → Release`, where Master is the "abstract album" and Release is a concrete edition.
- [ ] 2.1.2 Is a Master mandatory? On Discogs a release can exist without a master (a single, a one-off). For us — must a release always belong to an album?
- [ ] 2.1.3 Do we need a `Label` level as a separate entity with its own page, or is it just a string on the release?
- [ ] 2.1.4 Do we need separate entities: `Track` (a global composition) vs `Tracklist item` (a position on a specific release)? Global tracks give you "which releases contain this song", but noticeably complicate the model and moderation.
- [ ] 2.1.5 Separate entities `Series` (edition series), `Studio`, `Publisher` — needed?

## 2.2 Artist

- [ ] 2.2.1 Aliases and "real name" — do we support them?
- [ ] 2.2.2 A band and its members (members / groups), with membership dates?
- [ ] 2.2.3 Do we distinguish a person-artist from a collective-artist?
- [ ] 2.2.4 How do we handle "Various Artists" and compilations?
- [ ] 2.2.5 Same-named artists (two different The Beat) — numbering as on Discogs (`Beat (2)`) or another disambiguation mechanism?
- [ ] 2.2.6 Fields: country, years active, biography, sites/links, photo. Which of these are needed in the MVP?

## 2.3 Album / Master

- [ ] 2.3.1 Fields: title, main artist(s), year of first release, type (album / EP / single / compilation / live / soundtrack / bootleg), genres, styles.
- [ ] 2.3.2 Genres and styles — a fixed vocabulary (as on Discogs) or free-form tags? Do we let users add new values?
- [ ] 2.3.3 Multi-artist albums, featuring, "Artist A & Artist B" — how do we model this (roles on the artist↔album relation)?
- [ ] 2.3.4 Does the album need an aggregated tracklist, or does the tracklist live only on the release?

## 2.4 Release (the main collection entity)

- [ ] 2.4.1 Mandatory field set: title (if it differs), date (year / full date / approximate), country, label(s), catalogue number(s), format.
- [ ] 2.4.2 Format as a structure: medium (Vinyl / CD / Cassette / File / …), unit count (2×LP), size (12"/7"), speed (33⅓/45), extra descriptors (Album, Reissue, Remastered, Limited Edition, Picture Disc, Coloured Vinyl). Do we copy the Discogs "format + qty + descriptions[]" model?
- [ ] 2.4.3 Identifiers: barcode (UPC/EAN), matrix/runout, mastering SID, mould SID, ASIN, ISRC — which do we support, and do we need a dedicated "identifier" type for them (type + value + description)?
- [ ] 2.4.4 Tracklist: position (`A1`, `B2`, `1-3`), title, duration, track artist (for compilations), nesting (index tracks / suite movements).
- [ ] 2.4.5 Credits: roles (producer, mixing, sleeve design, guitar…) at release level and at track level. Is the role vocabulary fixed or free-form?
- [ ] 2.4.6 Companies (pressed by, distributed by, phonographic copyright ℗, copyright ©) — needed in the MVP?
- [ ] 2.4.7 Free-form fields: notes, data quality, "unofficial/bootleg" flag.
- [ ] 2.4.8 Multiple release images (front / back / label / gatefold / media) — how many, which types.
- [ ] 2.4.9 How do we tell apart different pressings with identical data (the eternal collector's question)? Do we allow "near-duplicates"?

## 2.5 Import and external sources

**Priority:** P0 — affects both legal aspects and the model.

- [ ] 2.5.1 Do we import the catalogue from **MusicBrainz** (CC0, open dump, API available)?
- [ ] 2.5.2 Do we import from **Discogs** (data is CC0, but the API/dumps come with ToS and rate limits) — is that legally and technically acceptable?
- [ ] 2.5.3 Cover art: Cover Art Archive / user uploads / external links — what is legal and what is actually feasible.
- [ ] 2.5.4 Do we store external IDs (`musicbrainz_id`, `discogs_id`) on entities for deduplication and future synchronisation? (I recommend yes, even if there is no import yet; storage form — see [10.4.1](10d-model-requirements.md))
- [ ] 2.5.5 Importing a user's collection from a Discogs CSV — needed in the MVP? (this is a strong "switch to us" argument; details in [section 10.2](10b-import.md))
- [ ] 2.5.6 Do we need metadata from Spotify/Apple/Last.fm (cover art, previews, scrobbling)?

## Working notes

_(none yet)_

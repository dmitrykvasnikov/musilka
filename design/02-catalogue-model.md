# 2. Catalogue domain model

**Priority:** P0

The core of the project. This is where we decide how closely we copy Discogs.

## 2.1 Entity hierarchy

- [x] 2.1.1 Do we confirm the chain `Artist → Album (master) → Release`? Or, in Discogs terms: `Artist → Master → Release`, where Master is the "abstract album" and Release is a concrete edition.
      **Decision:** **Confirmed**, with `Label` as a fourth top-level entity ([2.1.3](02-catalogue-model.md))
      and nothing else ([2.1.4](02-catalogue-model.md), [2.1.5](02-catalogue-model.md)).
      Recommendation 1 ("copy the Discogs model almost literally") is accepted at this level: the
      Master/Release split is the one distinction collectors of physical media actually need, since
      [1.2](01-product.md) makes the *edition* the thing being collected. A model without it cannot
      answer "which pressings of this album exist", which is the question the product is for.
      The division of labour: **Release carries the facts printed on the object** (date, country,
      label, catalogue number, format, identifiers, tracklist), **Master carries the facts about the
      work** (title, credit, first-release year, type, genres, styles). Where a fact could sit on
      either, it goes on Release — the object is the ground truth, the master is an interpretation.
- [x] 2.1.2 Is a Master mandatory? On Discogs a release can exist without a master (a single, a one-off). For us — must a release always belong to an album?
      **Decision:** **Yes — mandatory. `release.master_id` is `NOT NULL`.** Chosen 2026-08-15 over
      the recommended nullable FK, with the import cost understood and accepted.
      What we get: one shape for every query and every page; no `master_id IS NULL` branch anywhere
      in the code; no second class of "orphan" release for the UI, the API ([10.1](10a-public-api.md))
      or the export ([10.3](10c-export.md)) to special-case. For a portfolio piece
      ([1.1](01-product.md)) the uniformity is worth more than the row count.
      **Accepted cost, stated plainly so it is not rediscovered later:**
      1. **The importer must invent a master per row.** A CSV row carries artist and title and
         nothing that identifies the *work*, so ~12,000 releases produce ~12,000 masters with one
         child each ([1.4](01-product.md)). Most master pages will be noise for a long time.
      2. **Master merge is therefore not a late feature — it is the mechanism by which the catalogue
         becomes correct at all.** It moves into early scope alongside release merge
         ([2.4.9](02-catalogue-model.md)) rather than waiting for [section 4](04-editing.md)'s
         moderation work, and it needs the same guarantees: repoint children, redirect the merged-away
         ID, never orphan a collection item ([section 3](03-collection.md)).
      3. **Duplicate masters are the normal state, not an error state.** Nothing in the UI may treat a
         one-child master as suspect, and no uniqueness constraint may be placed on master
         title+credit — see [2.4.9](02-catalogue-model.md), whose reasoning applies here unchanged.
      Considered and rejected: auto-creating masters but hiding single-child ones in the UI. It buys
      back the tidiness at the price of a rule every view has to remember, and "is this master real"
      becomes a count query in a dozen places.
- [x] 2.1.3 Do we need a `Label` level as a separate entity with its own page, or is it just a string on the release?
      **Decision:** **A separate entity with its own page.** Not negotiable at the schema level,
      because of a fact already established: Discogs pairs `label` against `catno` positionally, and
      label names may themselves contain commas (NOTES.md, constraint of 2026-08-14). A release
      therefore has an *ordered list of (label, catalogue number) pairs*, which is a join table no
      matter what — and once the join exists, pointing it at an entity rather than repeating a string
      costs nothing and buys the label page.
      Shape: `release_company(release_id, position, label_id, role, catno)`. The `role` column is what
      lets the same entity serve [2.4.6](02-catalogue-model.md)'s companies; it defaults to `label`,
      and `catno` is only meaningful for that role.
      Rejected: a plain string on the release. It makes "everything on Melodiya" impossible, and
      normalising it afterwards over ~3,000 label names ([1.4](01-product.md)) is exactly the
      irreversible schema mistake [1.4](01-product.md) exists to prevent.
- [x] 2.1.4 Do we need separate entities: `Track` (a global composition) vs `Tracklist item` (a position on a specific release)? Global tracks give you "which releases contain this song", but noticeably complicate the model and moderation.
      **Decision:** **No global `Track` entity. Tracklist items only, owned by the release.**
      A tracklist item is a row on one release and means nothing outside it.
      Rationale, in order of weight:
      1. **The data will not be there.** [1.4](01-product.md) estimates tracks at "~120,000 in
         theory, near zero in practice" — no import channel carries a tracklist ([1.5](01-product.md)),
         so every track in the database is one a human typed. Building composition identity on top of
         a near-empty table is modelling for a catalogue we do not have.
      2. **It is not one entity, it is two.** Doing this properly means the work/recording
         distinction (MusicBrainz needed both, and years to get them right). Doing it improperly —
         one `Track` matched by title — produces garbage the moment two different songs share a name,
         which is constantly.
      3. **The moderation cost lands on a solo project.** Every "is this the same song" judgement is a
         human decision, and [1.11](01-product.md) has 2–3 hours a day.
      This is a **deliberate, selective simplification** of the Discogs model, in the spirit of
      recommendation 1 rather than against it. It is also **additive to reverse**: tracklist items keep
      their titles, so a future `work` entity is a new table plus a linking pass, not a rewrite.
- [x] 2.1.5 Separate entities `Series` (edition series), `Studio`, `Publisher` — needed?
      **Decision:** **None of the three as separate entities.**
      - `Studio` and `Publisher` are companies, and companies are `Label` rows under a different role
        ([2.1.3](02-catalogue-model.md), [2.4.6](02-catalogue-model.md)). Discogs reached the same
        conclusion — its label entity carries pressing plants and studios too — and that unification
        is one of the parts of its model worth copying literally.
      - `Series` is **dropped**, not deferred. It is a real thing on real objects ("Blue Note 75",
        "Original Master Recording"), but at ~12,000 releases it would be a mostly-empty table with a
        moderation cost, and where it matters it is already expressible as a format descriptor
        ([2.4.2](02-catalogue-model.md)) or in notes ([2.4.7](02-catalogue-model.md)).
      Reopen only if the catalogue grows a genuine cluster of series-organised releases; it is an
      additive table, so the cost of being wrong here is low.

## 2.2 Artist

Deferred on 2026-08-15 so the rest of the model could close first, then taken up and closed the same
day.

- [x] 2.2.1 Aliases and "real name" — do we support them?
      **Decision:** **Yes to both — but they are two different mechanisms, and conflating them is the
      mistake to avoid.**
      **One artist, many names.** `artist_name(artist_id, kind, name)` with
      `kind ∈ primary, variant, real_name`, exactly one `primary` per artist. All rows go into one
      search index (Postgres FTS plus `pg_trgm`, per [1.4](01-product.md) and recommendation 4), so a
      search in either script finds the one entity.
      **This closes NOTES.md's `Кино` / `Kino` blocker**, parked on section 2 by
      [1.8](01-product.md). The band is one artist with `Кино` primary and `Kino` a variant — not two
      entities, not a translation string, and nothing in the application ever transliterates anything.
      `real_name` is a name row rather than a column so that a person may have several (birth name,
      legal name) and so that all names stay in one index.
      **Aliases in the other sense — separate personas with separate discographies** (Aphex Twin /
      AFX / Polygon Window) — are **not** name rows. They are distinct artists linked by
      [2.2.2](02-catalogue-model.md)'s relation table with type `alias_of`.
      **The test that separates the two, and it is a sharp one: same discography or different?** If
      the releases belong on one page, it is a name row. If they belong on two pages that point at
      each other, it is two artists and a relation. Applied consistently this never needs a judgement
      call twice.
      Note what `credited_as` on the credit does *not* do here ([2.3.3](02-catalogue-model.md)): it
      makes *display* faithful to one object. Name rows make *search* work across all of them.
      Different jobs, both needed.
- [x] 2.2.2 A band and its members (members / groups), with membership dates?
      **Decision:** **Yes, as a general artist-to-artist relation**, in the schema from day one:
      `artist_relation(from_artist_id, to_artist_id, type, begin_date, end_date)` with
      `type ∈ member_of, alias_of`. Dates are the partial-date type from
      [2.4.1](02-catalogue-model.md), both nullable — "joined 1978, still in the band" and "was in it
      at some point" are both expressible.
      The table costs nothing extra because [2.2.1](02-catalogue-model.md) needs it for `alias_of`
      regardless. **Editing membership is post-MVP UI**; the storage exists first so no migration is
      needed when it arrives.
      No constraint that a `member_of` source is a person — [2.2.3](02-catalogue-model.md) is
      nullable, and bands do contain bands. The UI may warn; the schema does not forbid.
- [x] 2.2.3 Do we distinguish a person-artist from a collective-artist?
      **Decision:** **Yes, as a nullable `artist_type ∈ person, group, other`.**
      **Nullable is the whole point** — an imported CSV row gives a name and nothing else
      ([1.5](01-product.md)), and a required type would mean guessing from the name, which is
      unguessable. Unset is the normal state for a thin artist and must never block anything.
      What it earns: `real_name` ([2.2.1](02-catalogue-model.md)) and members
      ([2.2.2](02-catalogue-model.md)) only make sense on one of the two, the artist page can label
      itself honestly, and search can separate a person from the band named after them.
- [x] 2.2.4 How do we handle "Various Artists" and compilations?
      **Decision:** **A reserved singleton artist row** — `is_special = true`, seeded at deploy
      alongside the vocabularies, not editable, not mergeable, not deletable.
      A compilation credits it exactly like any other artist, which is the point: **no read path
      anywhere gets a branch**. No template, query, API response ([10.1](10a-public-api.md)) or export
      ([10.3](10c-export.md)) needs to know that this artist is special.
      Rejected: a `is_various` boolean on the master (an `if` in every renderer, forever), and "empty
      credit means various" (`NULL` would then mean both "various artists" and "nobody has filled this
      in", which are opposite facts about data quality).
      **The compilation half is already handled elsewhere:** `compilation` is a secondary type on the
      master ([2.3.1](02-catalogue-model.md)), and the real per-track names live on the tracklist's
      own credit ([2.4.4](02-catalogue-model.md)). The singleton only answers "what goes in the
      billing".
      Seeded set is exactly one row (`Various`). Whether a blank artist column in an upload should
      also land somewhere reserved is an importer question for [10.2](10b-import.md), not a model
      question — the model is content for it to be a thin artist like any other.
- [x] 2.2.5 Same-named artists (two different The Beat) — numbering as on Discogs (`Beat (2)`) or another disambiguation mechanism?
      **Decision:** **A nullable free-text `disambiguation` field, MusicBrainz-style. No numbering.**
      Discogs' `Beat (2)` is an artefact of using the name as a key. We have surrogate IDs, so the
      number buys nothing — and it is **inside the name**, which means it leaks into search results,
      sort order, the CSV export and the sleeve credit, where it is simply wrong.
      The comment sits *beside* the name instead: stored plainly, never concatenated into it,
      rendered in parentheses by listings and search results where ambiguity exists, and omitted from
      exports. It is editorial, so it can be improved later without rewriting data other things point
      at.
      **Consequence for import**, which is where this field earns its keep: a name alone is ambiguous,
      so [10.2](10b-import.md)'s matcher cannot silently pick one. It must show candidates, and the
      disambiguation is the only thing that makes that list choosable by a human.
- [x] 2.2.6 Fields: country, years active, biography, sites/links, photo. Which of these are needed in the MVP?
      **Decision:** **Nothing is required beyond one `primary` name row.** Everything else is
      optional and present from day one, because they are all fields on a single edit form and the
      marginal cost of each is close to zero:
      - `disambiguation` ([2.2.5](02-catalogue-model.md)), `artist_type`
        ([2.2.3](02-catalogue-model.md)).
      - **country** — the same code table as [2.4.1](02-catalogue-model.md), historical entries
        included. One table, both uses.
      - **years active** — begin and end partial dates, same type as everywhere else.
      - **biography** — a free-text `profile` field. Free text is right here: it is prose, not a value
        drawn from a set, so the curated-vocabulary invariant does not apply.
      - **links** — `artist_link(artist_id, position, url, label)`, a table because there are several.
        **Not to be confused with external IDs** ([2.5.4](02-catalogue-model.md)): a link is an
        editorial URL a user typed and the site may display, an external ID is stored provenance we
        never dereference. Different things, different tables, different rules.
      - **photo — no. Artist images are out of the MVP**, and this is the one real exclusion in the
        item. Two reasons that compound: [1.4](01-product.md) already names images as the cost that
        scales with the catalogue and predicts they break the estimate first, and an artist photo is
        the most copyright-fraught image on the site — a press or live photograph with an identifiable
        owner, unlike a sleeve scan of an object the uploader holds
        ([section 13](13-legal.md)). Releases get images ([2.4.8](02-catalogue-model.md)); artists
        wait until there is a reason and a legal answer.

## 2.3 Album / Master

- [x] 2.3.1 Fields: title, main artist(s), year of first release, type (album / EP / single / compilation / live / soundtrack / bootleg), genres, styles.
      **Decision:** Fields are `title`, artist credit, `first_release_year`, type, genres, styles.
      Only **title and artist credit are required**; everything else is nullable or empty, per the
      thin-catalogue invariant (NOTES.md).
      - **`title`** — required, free text, recorded as it appears ([1.8](01-product.md)).
      - **artist credit** — required, and at least one entry. Shape settled in
        [2.3.3](02-catalogue-model.md): an ordered credit, so a split album is one master credited to
        two artists. A compilation credits the `Various` singleton
        ([2.2.4](02-catalogue-model.md)), which keeps "required" true without exceptions.
      - **`first_release_year`** — a nullable year, not a date. The master is an abstraction; a
        precise day belongs to a release ([2.4.1](02-catalogue-model.md)).
      - **type** — the listed vocabulary is not one axis, so it is not one column. Split, as
        MusicBrainz does: a nullable **primary type** (`album`, `ep`, `single`, `other`) plus a set of
        **secondary types** (`compilation`, `live`, `soundtrack`, `remix`, `dj_mix`, `spokenword`). A
        live compilation is then expressible; a single enum forces a lie.
      - **`bootleg` is not a master type.** It is a property of a *pressing*, and the same album has
        official and bootleg editions. It becomes `release.is_unofficial`
        ([2.4.7](02-catalogue-model.md)). This is the clearest case in the section of the
        [2.1.1](02-catalogue-model.md) rule: object facts go on the release.
      - **genres and styles** — see [2.3.2](02-catalogue-model.md).
- [x] 2.3.2 Genres and styles — a fixed vocabulary (as on Discogs) or free-form tags? Do we let users add new values?
      **Decision:** **A fixed, curated vocabulary. Users pick from it; they do not add to it directly.**
      Two kinds in one `tag` table: ~20 genres and a few hundred styles, seeded at deploy. Adding a
      value is a moderation action ([section 4](04-editing.md)) — a user requests, a moderator seeds.
      Rationale: free-form tags fragment immediately (`hip hop` / `hip-hop` / `HipHop`), and the whole
      value of the field is *filtering*, which fragmentation destroys. At ~12,000 releases
      ([1.4](01-product.md)) there is no long tail to justify the cost. Discogs' curated list is
      another part of the model earned through practice (recommendation 1).
      **They live on the Master, not the Release** — a direct consequence of
      [2.1.2](02-catalogue-model.md). Genre is a property of the work: a 2015 repress of a jazz record
      is still jazz, and duplicating the tags across every pressing would mean editing them n times.
      Now that every release has a master, there is no orphan case to worry about.
      Flat, not hierarchical: styles are not cleanly nested under genres in reality, and pretending
      otherwise buys a constraint we would immediately need to break.
- [x] 2.3.3 Multi-artist albums, featuring, "Artist A & Artist B" — how do we model this (roles on the artist↔album relation)?
      **Decision:** **An ordered artist credit**, not roles on a relation. A credit is a list of
      `(position, artist_id, credited_as, join_phrase)`, rendered by concatenating each name with the
      phrase that follows it.

      ```
      position  artist_id      credited_as   join_phrase
      0         Melvins(#12)   NULL          " / "
      1         Fantômas(#88)  NULL          NULL
      →  Melvins / Fantômas
      ```

      This passes all four requirements the split-album note sets out (see working notes): both
      artists are entities, so the split appears on both pages; order is preserved as printed;
      *X / Y*, *X & Y* and *X feat. Y* differ only in `join_phrase`, which a plain many-to-many cannot
      express; and it works unchanged at master level, which [2.1.2](02-catalogue-model.md) requires
      since a split is one master.
      **`credited_as` is nullable and normally `NULL`, meaning "use the artist's current primary
      name".** Copying the name in would freeze it — a later rename or correction would stop
      propagating. It is filled only when the object disagrees with the artist's primary name: the
      release credited to `Кино` where the artist's primary is `Kino`, or a sleeve misspelling worth
      preserving. Where a `credited_as` recurs, the UI should offer to add it as a name variant
      ([2.2.1](02-catalogue-model.md)) — display fidelity and searchability are separate jobs and
      both want filling.
      **`join_phrase` is free text, and is a deliberate exception to the curated-vocabulary
      invariant** (NOTES.md). The invariant's reason is that free text destroys grouping — but nothing
      groups or filters by join phrase; it is purely display. The real vocabulary is open in practice
      (` / `, ` & `, ` feat. `, ` with `, ` vs. `, ` meets `), and a closed list would force lies on
      the one field whose entire job is reproducing the sleeve.
      **Three tables, not one polymorphic table:** `master_artist_credit`, `release_artist_credit`,
      `track_artist_credit`, identical in shape. Postgres gets real foreign keys and no polymorphic
      joins; Haskell gets one shared type and one renderer. Near-duplicate DDL is a cheap price for
      referential integrity, and `track_artist_credit` is what closes
      [2.4.4](02-catalogue-model.md)'s per-track artist.
      **Not to be confused with role credits** ([2.4.5](02-catalogue-model.md)). These tables answer
      *who it is by* — the billing on the front of the sleeve. `release_credit` answers *who did
      what* — producer, guitar, sleeve design. Different questions, different tables, similar names;
      the `_artist_credit` suffix is what keeps them apart.
      Rejected: a simple many-to-many (loses the join phrase, so a split and a collaboration render
      identically); a single artist FK plus a display string (the second artist stops being an entity
      and the split vanishes from their page); MusicBrainz-style deduplicated shared credit entities
      (real value at their scale, pure indirection at ~12,000 releases — each credit is owned by its
      row).
- [x] 2.3.4 Does the album need an aggregated tracklist, or does the tracklist live only on the release?
      **Decision:** **Only on the release.** No tracklist on the master, aggregated or otherwise.
      An aggregate would have to be derived from releases that disagree with each other — different
      pressings genuinely have different tracks, orders and bonus material — so any aggregation is
      either a lie or a merge UI nobody asked for.
      Instead the master carries a nullable **`key_release_id`**, pointing at one of its own releases,
      and the master page shows that release's tracklist. Where it is unset, the master page shows no
      tracklist at all, which is honest. Nothing is derived, nothing is cached, nothing goes stale.

## 2.4 Release (the main collection entity)

- [x] 2.4.1 Mandatory field set: title (if it differs), date (year / full date / approximate), country, label(s), catalogue number(s), format.
      **Decision:** **Mandatory: `title`, `master_id`, artist credit. Everything else is optional.**
      That short list is forced by the thin-catalogue invariant (NOTES.md): a release created from a
      CSV row must be able to exist, and any field the importer cannot fill cannot be required.
      - **`title` is always stored**, not "only if it differs from the master". Storing it
        unconditionally removes a fallback rule from every read path, at the cost of one duplicated
        string per release.
      - **date** — a **partial date**: year, year-month, or full date, plus a nullable
        `date_is_approximate` flag for `ca. 1974` on the sleeve. Not three columns and not a string —
        one nullable date plus a precision enum, so ordering and range queries still work.
      - **country** — a nullable code, **with historical entries**. ISO 3166-1 alpha-2 alone is wrong
        for this product: a Melodiya pressing is from `SU`, and collectors of physical media
        ([1.2](01-product.md)) will not accept `RU`. The lookup table therefore seeds ISO plus
        `SU`, `YU`, `DD`, `CS` and similar, and codes are never retired.
      - **labels and catalogue numbers** — the ordered pair list from
        [2.1.3](02-catalogue-model.md). Possibly empty.
      - **format** — the structure from [2.4.2](02-catalogue-model.md). Possibly empty, because an
        import may not carry one.
- [x] 2.4.2 Format as a structure: medium (Vinyl / CD / Cassette / File / …), unit count (2×LP), size (12"/7"), speed (33⅓/45), extra descriptors (Album, Reissue, Remastered, Limited Edition, Picture Disc, Coloured Vinyl). Do we copy the Discogs "format + qty + descriptions[]" model?
      **Decision:** **Yes, copy it — with the descriptor vocabulary scoped per medium.**
      A release has an ordered list of format entries (an LP with a bonus 7", a CD+DVD set are both
      normal), each being `(position, medium, qty, descriptors[])`.
      - **`medium`** is a closed enum: `vinyl`, `cd`, `cassette`, `reel`, `minidisc`, `shellac`,
        `file`, `other`. Extending it is a seed change.
      - **`descriptors`** come from a controlled vocabulary **valid for that medium** — `45 RPM` and
        `12"` cannot land on a CD, `Picture Disc` cannot land on a cassette. This is the safety a
        typed model would give, enforced by a validation table rather than by the schema.
      - **The domain layer parses each row into a per-medium sum type** (`Vinyl VinylSpec | Cd CdSpec
        | …`), which is what [1.12](01-product.md) meant by "multi-format releases as sum types". The
        types live in Haskell, where they are cheap; the storage stays uniform, where rigidity is
        expensive.
      This satisfies [1.2](01-product.md) exactly: format is first-class and multi-format, and **no
      format-specific field is promoted onto the release** — `size` and `speed` are vinyl descriptors,
      not release columns.
      Rejected: a table per medium (every new medium or descriptor becomes a migration, and a
      half-known imported format has nowhere to go); a free-text format string (kills "show me all my
      cassettes", which is the point of [1.2](01-product.md)).
- [x] 2.4.3 Identifiers: barcode (UPC/EAN), matrix/runout, mastering SID, mould SID, ASIN, ISRC — which do we support, and do we need a dedicated "identifier" type for them (type + value + description)?
      **Decision:** **Yes to the dedicated type**, exactly as posed:
      `release_identifier(release_id, position, type, value, description)`.
      A release has many, of the same type — a double LP has four runout etchings — so nothing here
      can be a column on the release.
      - **`type`** is a closed enum: `barcode`, `matrix_runout`, `mastering_sid`, `mould_sid`,
        `label_code`, `rights_society`, `asin`, `other`. `description` is free text and carries what
        the enum cannot ("Side A runout, etched").
      - **`value` is stored verbatim**, spacing and all, because on a runout the exact glyphs are the
        evidence. For `barcode` a **normalised digits-only column is stored alongside** it, which is
        what duplicate detection matches on ([2.4.9](02-catalogue-model.md)).
      - **ISRC is release-level only in MVP.** It is genuinely per-track, and track-level identifiers
        are deferred rather than faked — see [2.4.4](02-catalogue-model.md).
- [x] 2.4.4 Tracklist: position (`A1`, `B2`, `1-3`), title, duration, track artist (for compilations), nesting (index tracks / suite movements).
      **Decision:** `release_track(release_id, sequence, position, title, duration_s, parent_id, kind)`.
      - **`sequence`** is an integer and is the *only* thing ordering ever uses. **`position`** is
        free text (`A1`, `B2`, `1-3`, or empty) and is display-only — it is a label printed on an
        object, and any attempt to parse it into structure breaks on the next release.
      - **`duration_s`** is a nullable integer of seconds. Sleeves print `4:07`; that is a rendering
        concern.
      - **`kind`** ∈ `track`, `heading` — a heading is an unnumbered row ("Suite in three parts") that
        carries no duration.
      - **nesting: `parent_id`, one level deep, enforced.** Enough for index tracks and suite
        movements, which is all that occurs in practice. Arbitrary depth buys recursive rendering and
        recursive validation for no observed case.
      - **track artist** — a `track_artist_credit` row set, the same ordered credit as everywhere
        else ([2.3.3](02-catalogue-model.md)). **Absent by default**, and absence means "as the
        release is credited" rather than "unknown" — so only compilations and guest spots carry one,
        and the common case stores nothing. This is where a compilation's real artists live, the
        release billing being the `Various` singleton ([2.2.4](02-catalogue-model.md)).
- [x] 2.4.5 Credits: roles (producer, mixing, sleeve design, guitar…) at release level and at track level. Is the role vocabulary fixed or free-form?
      **Decision:** **a curated role vocabulary plus a free-text detail field.** The role is a seeded
      enum-ish
      lookup (`Producer`, `Mixed By`, `Guitar`, `Sleeve Design`, …); the detail is free text and
      renders in brackets, giving Discogs' `Guitar [Slide]` without letting the role itself
      fragment. Same reasoning as [2.3.2](02-catalogue-model.md): the value of the field is grouping,
      and free text destroys grouping.
      **Release-level in the MVP, but `track_id` is nullable on the table from day one.** Adding a
      column to a populated credits table later is a migration we can trivially avoid by writing it
      now; this is the cheapest possible instance of recommendation 7's "retrofitting is the expensive
      rework".
      **What a role credit points at — settled 2026-08-15: an artist entity, always.** Never a raw
      name, and never "either". A raw-name option would create a second class of artist that no page
      links to and no search finds, which is the exact failure mode
      [2.3.3](02-catalogue-model.md) rejected the display-string design for. The friction is real —
      crediting a sleeve designer means creating an artist — but it is one click, and
      [2.2.6](02-catalogue-model.md) requires nothing of an artist but a name. The row carries its own
      nullable `credited_as`, on the same rule as [2.3.3](02-catalogue-model.md).
      **Table name: `release_credit`**, distinct from `release_artist_credit`
      ([2.3.3](02-catalogue-model.md)). This one is *who did what*; that one is *who it is by*.
- [x] 2.4.6 Companies (pressed by, distributed by, phonographic copyright ℗, copyright ©) — needed in the MVP?
      **Decision:** **Not in the MVP UI, but free in the schema from day one.** No separate table:
      companies are rows in [2.1.3](02-catalogue-model.md)'s `release_company` join, distinguished by
      `role` — `label`, `pressed_by`, `distributed_by`, `phonographic_copyright`, `copyright`,
      `made_by`, `recorded_at`, `mastered_at`. The MVP shows and edits only `label`.
      This costs nothing now because the join and its `role` column exist regardless, and it means
      turning companies on later is a UI change with no migration. It also settles the `Studio` and
      `Publisher` half of [2.1.5](02-catalogue-model.md).
- [x] 2.4.7 Free-form fields: notes, data quality, "unofficial/bootleg" flag.
      **Decision:** two of the three.
      - **`notes`** — yes, free text on the release. It is the natural home for everything the schema
        deliberately does not model, and [2.4.9](02-catalogue-model.md) leans on it. Whether it
        renders any markup is a [section 7](07-search-ux.md) question.
      - **data quality — no field.** Discogs' "Needs Vote / Complete and Correct" is the visible tip
        of a voting workflow we have not designed, and a status column with no process behind it is
        decoration that goes stale. The quality signal here is the edit history
        ([section 4](04-editing.md)), which is real data rather than an assertion. If a review queue
        is ever wanted it is [section 4](04-editing.md)'s to design, not a column here.
      - **`is_unofficial`** — yes, a boolean on the release. Covers bootlegs and unauthorised
        pressings. It is deliberately *not* a master type ([2.3.1](02-catalogue-model.md)), and
        deliberately not an enum: "promo" and "test pressing" are format descriptors
        ([2.4.2](02-catalogue-model.md)), not degrees of officialness.
- [x] 2.4.8 Multiple release images (front / back / label / gatefold / media) — how many, which types.
      **Decision:** many per release, `release_image(release_id, position, type, is_primary)`.
      **`type`** ∈ `front`, `back`, `sleeve` (gatefold, inner, insert), `media` (the disc, label or
      shell itself), `other`. Exactly one image per release may be `is_primary`; it is what lists and
      collection grids show.
      **Cap: 8 images per release.** This is not arbitrary — [1.4](01-product.md) already flags images
      as the one thing that scales with the catalogue rather than with users, at ~10 GB and real
      backup time, and names it as where the estimate breaks first. Eight is generous for a gatefold
      double LP and bounds the total.
      Storage, dimensions, byte limits, formats and derivatives are [section 8](08-media.md)'s;
      **the cap and the type vocabulary are the model's** and are decided here.
- [x] 2.4.9 How do we tell apart different pressings with identical data (the eternal collector's question)? Do we allow "near-duplicates"?
      **Decision:** **Yes, near-duplicates are allowed. There is no uniqueness constraint anywhere on
      the release, at any stage.** Not on `(artist, title, catno)`, not on `(master, format,
      country, year)`, not on barcode.
      This is the honest answer rather than the convenient one: two pressings really can differ only
      in a runout etching, and a constraint that forbids the second one makes the catalogue wrong in a
      way the user cannot fix. Given [1.5](01-product.md), a constraint would also mean the importer
      *silently discards a user's row*, which is unacceptable.
      Three mechanisms instead of a constraint:
      1. **A soft duplicate check at create time.** On save, likely matches are surfaced (normalised
         barcode from [2.4.3](02-catalogue-model.md), then trigram similarity on title plus catno —
         Postgres FTS and `pg_trgm`, per [1.4](01-product.md)'s veto list and recommendation 4). The
         user picks an existing release or confirms theirs is new. It advises; it never blocks.
      2. **Identifiers and notes are the disambiguation carriers.** Matrix/runout
         ([2.4.3](02-catalogue-model.md)) is what actually separates two otherwise-identical
         pressings, and `notes` ([2.4.7](02-catalogue-model.md)) carries the rest. This is why
         identifiers are a first-class table and not an afterthought.
      3. **Merge is a first-class, reversible operation** ([section 4](04-editing.md)) with a
         `merged_into` pointer on the losing row: the old ID keeps resolving and redirects, and
         collection items pointing at it follow ([section 3](03-collection.md)). Nothing is ever
         hard-deleted out from under a user's collection.
      **The same machinery serves masters**, which [2.1.2](02-catalogue-model.md) has now made a
      routine need rather than an occasional one. Build it once, for both.

## 2.5 Import and external sources

**Priority:** P0 — affects both legal aspects and the model.

- [x] 2.5.1 Do we import the catalogue from **MusicBrainz** (CC0, open dump, API available)?
      **Decision:** **No.** Closed by [1.5](01-product.md) and the "two channels in, and no others"
      invariant (NOTES.md): the server never downloads a dump or calls an external music database.
      Recorded here rather than merely cross-referenced because this item is where someone will look.
- [x] 2.5.2 Do we import from **Discogs** (data is CC0, but the API/dumps come with ToS and rate limits) — is that legally and technically acceptable?
      **Decision:** **No** — same invariant. Note the distinction that matters and is easy to lose:
      *we never call Discogs*, but a **user uploading their own Discogs export is fine and is the
      primary import path** ([2.5.5](02-catalogue-model.md), [10.2](10b-import.md)). The channel is
      the user's file, not our HTTP client.
- [x] 2.5.3 Cover art: Cover Art Archive / user uploads / external links — what is legal and what is actually feasible.
      **Decision:** **User uploads only.**
      - **Cover Art Archive: no** — fetching it is a server-side external call, forbidden by
        [1.5](01-product.md), regardless of its licensing being friendly.
      - **External links / hotlinking: no** — it is someone else's bandwidth, it rots, and it would
        make the catalogue's completeness depend on third parties staying up.
      - **Uploads: yes**, per [2.4.8](02-catalogue-model.md), with the mechanics in
        [section 8](08-media.md). The copyright question for user-uploaded sleeve scans is
        [section 13](13-legal.md)'s, and is not settled here.
- [x] 2.5.4 Do we store external IDs (`musicbrainz_id`, `discogs_id`) on entities for deduplication and future synchronisation? (I recommend yes, even if there is no import yet; storage form — see [10.4.1](10d-model-requirements.md))
      **Decision:** **Yes — store them, and never dereference them.**
      They arrive for free: a Discogs CSV row carries `release_id` (NOTES.md, 2026-08-14), so the
      importer has the ID in hand whether or not we keep it. Keeping it makes re-importing the same
      collection idempotent, makes two users' uploads of the same release land on one row, and
      preserves the user's own link back to where their data came from.
      "Never dereference" is the [1.5](01-product.md) invariant: the ID is a **stored fact about
      provenance**, not a handle we resolve. No enrichment, no sync, no lookup — not even once.
      **Storage form is [10.4.1](10d-model-requirements.md)'s and stays open**; recommendation 2 and
      the wording of that item both point at a separate table with provenance rather than columns per
      source, but this item does not decide it.
- [x] 2.5.5 Importing a user's collection from a Discogs CSV — needed in the MVP? (this is a strong "switch to us" argument; details in [section 10.2](10b-import.md))
      **Decision:** **Yes, in the MVP** — it is the only way the catalogue becomes non-empty
      ([1.4](01-product.md), [1.5](01-product.md)), so "MVP without import" is a product with nothing
      in it.
      **Sequenced after export**, per recommendation 9: export needs no external format knowledge and
      delivers the "your data is not locked in" promise of [1.3](01-product.md) immediately. Exact
      staging is [section 15](15-roadmap.md)'s.
      **What this section owed it is already paid.** A release created from a CSV row needs only
      title, master and credit ([2.4.1](02-catalogue-model.md)) — everything the CSV cannot supply is
      optional by construction. The one *new* obligation this decision creates is
      [2.1.2](02-catalogue-model.md)'s: the importer must mint a master per row.
      Partially unblocks [1.9](01-product.md); the rest waits on [2.2](02-catalogue-model.md).
- [x] 2.5.6 Do we need metadata from Spotify/Apple/Last.fm (cover art, previews, scrobbling)?
      **Decision:** **No, permanently.** Every one of them is a server-side call to an external
      service, forbidden by [1.5](01-product.md); they add API keys, quotas and ToS to the critical
      path, which [1.5](01-product.md) was specifically taken to avoid. Scrobbling is additionally
      out of scope on product grounds — this is a catalogue of physical objects
      ([1.2](01-product.md)), not a listening history.

## Working notes

- **2026-08-15 — Section closed. All 30 items decided.** [2.2](02-catalogue-model.md) and
  [2.3.3](02-catalogue-model.md) were deferred earlier the same day and then taken up once the rest
  of the model constrained them — which turned out to be the right order: mandatory master
  ([2.1.2](02-catalogue-model.md)) forced the credit design to work at master level, and the split
  album note below gave it a concrete test to pass. NOTES.md's `Кино`/`Kino` blocker is closed at
  [2.2.1](02-catalogue-model.md).
- **2026-08-15 — Two things called "credit", deliberately.** `*_artist_credit`
  ([2.3.3](02-catalogue-model.md)) is the billing — *who the record is by*, ordered, with join
  phrases. `release_credit` ([2.4.5](02-catalogue-model.md)) is roles — *who did what*, from a
  curated vocabulary. They look alike and are not: one is display-faithful and free-form, the other
  is grouped and closed. If the names cause trouble in implementation, rename the role table, not the
  artist one — the `_artist_credit` suffix is doing the work.
- **2026-08-15 — Three ways a name can vary, and they are not interchangeable.** This was the
  subtlest part of the section and is easy to get wrong later:
  `artist_name` variant — same artist, same discography, different spelling or script (`Кино` /
  `Kino`); makes **search** work ([2.2.1](02-catalogue-model.md)).
  `credited_as` on a credit — how *this one object* spells it; makes **display** faithful
  ([2.3.3](02-catalogue-model.md)).
  `alias_of` relation — a different persona with its own discography (Aphex Twin / AFX); makes
  **two pages** that point at each other ([2.2.2](02-catalogue-model.md)).
  The test for the first versus the third is "same discography or different?", and it is sharp enough
  to apply without a judgement call.
- **2026-08-15 — Mandatory master was chosen against the opening recommendation**, with the import
  cost laid out first. The trade accepted: uniform queries and no orphan-release class, paid for with
  ~12,000 one-child masters and merge machinery pulled forward into early scope. If that merge work
  turns out to dominate [section 4](04-editing.md), this is the item to revisit — going from
  `NOT NULL` to nullable is a cheap migration in that direction.
- **2026-08-15 — A recurring pattern worth naming**, since it decided five items the same way and
  will decide more: *where a value is drawn from a set, the set is curated and seeded, and free text
  sits beside it rather than inside it.* Genres and styles ([2.3.2](02-catalogue-model.md)), format
  descriptors ([2.4.2](02-catalogue-model.md)), identifier types plus `description`
  ([2.4.3](02-catalogue-model.md)), credit roles plus detail ([2.4.5](02-catalogue-model.md)),
  company roles ([2.4.6](02-catalogue-model.md)). The reason is always the same: the field exists to
  group or filter, and free text destroys grouping — but a curated set with no escape hatch makes
  users lie, so both are needed.
- **2026-08-15 — Split albums are the test case [2.2](02-catalogue-model.md) has to pass.** Raised
  while checking where multiple artists and multiple formats were decided — formats are settled
  ([2.4.2](02-catalogue-model.md): an ordered list of entries, so `2×LP + CD` is two rows and `qty`
  is a separate axis), multiple artists are not. A split LP is the sharpest available test of the
  deferred credit design, because it needs four things at once:
  1. **Both artists linkable as entities**, so "everything by X" finds the split. This is what a
     single-artist-plus-display-string cannot do.
  2. **Order preserved** — a split is *X / Y*, and which name comes first is a fact about the object.
  3. **A join phrase**, because *X / Y* (split), *X & Y* (collaboration) and *X feat. Y* (guest) have
     identical cardinality and different meaning. A plain many-to-many stores none of the three.
  4. **Side-level attribution**, which is [2.4.4](02-catalogue-model.md)'s deferred track artist, not
     the release credit.
  **Interaction with [2.1.2](02-catalogue-model.md), worth knowing before designing the credit:** a
  split is **one master with a two-artist credit**, not two masters — mandatory master forces that.
  So whatever credit design is chosen has to work at master level too, not only on the release.
- **2026-08-15 — Unresolved, deliberately: seeding the vocabularies.** Genres, styles, credit roles,
  format descriptors and country codes all need initial contents, and
  [1.5](01-product.md) forbids fetching them. Hand-written seed data is the assumed answer — it is
  our own list, not an extraction from anyone's database — but nobody has estimated the work. Belongs
  to [section 15](15-roadmap.md) as a real task, not a footnote.
- **2026-08-15 — Not asked by the agenda, and left open on purpose: soft delete.**
  [2.4.9](02-catalogue-model.md) needs a release to survive being merged away, and
  [section 3](03-collection.md) needs a collection item never to dangle. Whether that is
  `merged_into` alone, a `deleted_at`, or falls out of [section 4](04-editing.md)'s versioning is a
  [section 4](04-editing.md) question. Flagged here so it is not discovered during implementation.

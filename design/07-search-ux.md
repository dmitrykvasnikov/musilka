# 7. Search, navigation, UX

**Priority:** P1

**What this section is not free to decide.** [1.4](01-product.md)'s veto list already names every
dedicated search engine, so [7.2](07-search-ux.md) is a question about *how* to use Postgres rather
than whether to. [11.1](11-stack.md) and [11.3](11-stack.md) fixed server-rendered HTML with no build
step and no framework, which decides [7.8](07-search-ux.md) by construction and puts a hard ceiling
on what [7.4](07-search-ux.md) and [7.6](07-search-ux.md) may propose. [2.2.1](02-catalogue-model.md)
answered the multi-script question that [7.4](07-search-ux.md) still poses as open. What is genuinely
open here is which entities are searchable at all, what the result of a search *is*, and the screen
inventory — and the last of those is the item that turns [1.9](01-product.md)'s MVP paragraph into
something countable.

- [x] 7.1 What we search: artists, albums, releases, labels, tracks, users — unified search or separate?
      **Decision: one box, three result groups — artists, albums (masters), labels — plus an
      identifier shortcut. Releases are not a text-search result type, and tracks and users are not
      searchable at all.**
      **The split that decides this is between the two questions a collector actually asks.**
      *"Find me this album"* is artist-plus-title and is master-shaped: the work, not the pressing.
      *"What is this object in my hand"* is a catalogue number, a barcode or a matrix string, and is
      release-shaped — but it is an **exact lookup on [2.4.3](02-catalogue-model.md)'s identifiers**,
      not a full-text query. Treating both as "search over releases" would apply fuzzy matching to
      the one query where fuzziness is actively wrong.
      So the single box does this, in order:
      1. If the query normalises to a barcode or matches a catalogue number exactly
         ([2.4.3](02-catalogue-model.md)'s normalised forms), the release matches are shown first and
         a single match navigates straight to the release page.
      2. Otherwise, full text over artists, masters and labels, rendered as three labelled groups on
         one page. No tabs, no "search in:" dropdown — at [1.4](01-product.md)'s volume the whole
         result set fits on one screen, and a dropdown is a decision we would be asking the user to
         make on our behalf.
      **Releases are reached through their master, always.** Every release has one
      ([2.1.2](02-catalogue-model.md)), the master page lists its pressings, and the release is where
      you land from an identifier or from a link. This also keeps the index out of the table that
      grows fastest.
      **Tracks: not searchable.** [2.1.4](02-catalogue-model.md) refused a global track entity and no
      import channel carries a tracklist ([1.5](01-product.md)), so a "Tracks" group would be an
      empty heading on every results page for a year. Additive later: tracklist item titles are text
      in a table, and indexing them is one column and one index whenever there is anything in it.
      **Users: not searchable.** Profiles live at `/u/<nickname>` ([6.4](06-accounts.md)) and are
      reached from a public collection or wantlist. A people directory at tens of accounts
      ([1.4](01-product.md)) is a nickname list whose only consumer would be
      [section 5](05-messaging.md)'s new-conversation flow — which is outside
      [1.9](01-product.md)'s MVP anyway, and which [5.7](05-messaging.md) already guards reactively.
      **Companies ([2.4.6](02-catalogue-model.md)) are not searchable either**, for the same reason
      their UI is deferred: the table is empty in the MVP.
- [x] 7.2 Engine: PostgreSQL FTS/trigram (simpler, good enough for a long time) vs Meilisearch/Typesense/OpenSearch (better UX, extra infrastructure).
      **Decision: Postgres FTS with `pg_trgm`, and this is settled rather than chosen** —
      [1.4](01-product.md) vetoes every dedicated engine by name and
      [11.4](11-stack.md) records search as a table. What this item decides is the shape of the
      index, which nothing else has fixed.

      ```
      artist_name.search_tsv   GENERATED ALWAYS AS (to_tsvector('simple', unaccent(name))) STORED
      label.search_tsv         GENERATED ALWAYS AS (to_tsvector('simple', unaccent(name))) STORED
      master.search_tsv        maintained by trigger — title + every name row of every credited artist

      GIN on each search_tsv
      GIN (name gin_trgm_ops)  on artist_name and label, for fuzzy and for autocomplete
      ```

      - **`simple`, not `english` or `russian`.** We cannot know a row's language — a catalogue with
        `Кино` and `Coil` in it has no per-row language column and will not be given one — and
        stemming with the wrong dictionary is worse than not stemming. Nothing here is prose;
        it is proper names, where stemming has little to offer and `Beatles` → `beatl` has some
        capacity to surprise. `unaccent` handles the diacritic half of the problem
        (`Björk` / `Bjork`) and ships with a stock Postgres, so it is inside [11.4](11-stack.md)'s
        "extensions in the base image" rule.
      - **Two of the three documents are generated columns, and cannot drift.** That is the point of
        putting artist search on `artist_name` rather than on `artist`: [2.2.1](02-catalogue-model.md)
        requires every name row in one index, and one row per name gives that for free.
      - **`master.search_tsv` is the one denormalisation in the design, and it is trigger-maintained**
        — it must contain the credited artists' names, which live in other tables, or the single most
        common query in the product (`kino gruppa krovi`) matches nothing. Triggers on `master`,
        `master_credit` and `artist_name`; an artist rename touches the masters they are credited on,
        which at [1.4](01-product.md)'s volume is dozens of rows.
        **The test that makes this acceptable, and it is worth stating as a general rule:** a cache is
        allowed when it can be rebuilt from the truth by one statement and a stale value is
        detectable. `UPDATE master SET search_tsv = ...` regenerates the entire index in seconds. That
        is precisely what [3.1](03-collection.md)'s forbidden denormalisation is not — a title copied
        onto a collection item is indistinguishable from a deliberate edit, so it can never be
        rebuilt.
      - **Query pipeline: FTS first, trigram only as a fallback.** `websearch_to_tsquery` over the
        three documents; if that returns nothing, retry with trigram similarity and label the result
        "did you mean". Blending trigram distance into the primary ranking makes results
        unpredictable and slow for a gain nobody asked for.
      - **Ranking is `ts_rank` plus one tie-break**: exact prefix match on the primary name or title
        first. No popularity signal, because there is none — [5.11](05-messaging.md) removed ratings
        and view counts from the product.
      **When to revisit:** if the catalogue ever passes six figures of masters *and* a GIN scan is
      measurably the slow part of a page, which is two orders of magnitude away
      ([1.4](01-product.md)). The rewrite would be an external index behind the same service
      function, not a change to any page.
- [x] 7.3 Faceted filtering (genre, year, country, format, label) and sorting.
      **Decision: one filtered release list, reused in four places, with live counts.**
      The faceted surface is **not** the search results page — search answers "find the thing I can
      name" ([7.1](07-search-ux.md)) and filtering answers "narrow this set". They are different
      questions and one control cannot serve both without becoming a query builder.
      The same filtered-list component renders: the whole catalogue (`/releases`), an artist's
      releases, a label's releases, and a user's collection or wantlist.

      | Facet | Lives on | Note |
      |---|---|---|
      | Genre, style | master ([2.3.2](02-catalogue-model.md)) | joins through `release.master_id` |
      | Decade | release date ([2.4.1](02-catalogue-model.md)) | decade, not year — 60 buckets of three items is not a filter |
      | Country | release ([2.4.1](02-catalogue-model.md)) | includes `SU`, `DD`, `YU`; never remapped |
      | Format medium and descriptor | release ([2.4.2](02-catalogue-model.md)) | the filter [1.2](01-product.md) exists to support |
      | Label | `release_label` ([2.1.3](02-catalogue-model.md)) | |
      | Tags | `collection_item` ([3.3](03-collection.md)) | collection and wantlist views only |

      **Counts are computed live, on every request, with no cache and no counter columns** — the same
      decision [3.8](03-collection.md) made for statistics and for the same reason: a `GROUP BY` over
      thousands of rows is milliseconds and a denormalised count is a thing that drifts. This also
      makes [3.8](03-collection.md)'s breakdown tiles clickable, since a breakdown and a facet are the
      same query.
      **Sorting:** relevance (search results only), artist, title, release date, and — on collection
      and wantlist views only — date added. Deliberately short. No "popular", no "recently edited"
      (that is a change feed, deferred at [4.2](04-editing.md)), no random.
      **Filters are `GET` query parameters and every filtered view has a real URL.** This is not a
      style preference: [11.3](11-stack.md) requires every core flow to work with JavaScript
      disabled, and a filter that only exists in JS state is unlinkable, unbookmarkable and
      unindexable. HTMX swaps the list in place when it is available; the plain form submission is
      what actually happens.
      **Multi-select within a facet is `OR`, across facets is `AND`**, and that is the only combining
      rule — no negation, no nesting, no saved filters.
- [x] 7.4 Autocomplete and "similar" results, typo tolerance, RU↔EN transliteration (important for Russian-language queries).
      **Decision: autocomplete yes, typo tolerance as a fallback only, transliteration never — and
      the item's premise is already answered.**
      - **Transliteration was decided in [2.2.1](02-catalogue-model.md) and is now an invariant: the
        application never transliterates anything.** `Кино` and `Kino` are two name rows on one
        artist, both in the index ([7.2](07-search-ux.md)), so a query in either script finds the one
        entity. There is no transliteration table, no ICU dependency and no normalisation step, and
        this item may not reintroduce one.
      - **The gap this leaves is real and is accepted: a Latin-typed query for a Cyrillic *title*
        finds nothing.** `Gruppa Krovi` does not match `Группа крови`, because masters have one title
        ([2.3.1](02-catalogue-model.md)) and no variant rows. The route that works is the artist —
        found in either script, with the discography on their page — and at
        [1.4](01-product.md)'s volume that is one extra click, not a dead end. **Reopening is
        additive and specified:** a `master_title(master_id, kind, title)` table mirroring
        `artist_name`, indexed the same way. It is not built now because nothing would fill it: no
        import channel carries a second title ([1.5](01-product.md)), so every row would be
        hand-typed.
      - **Autocomplete: yes, and it matters most where it is least visible.** Every credit in the
        catalogue must point at an artist entity (NOTES.md invariant), so the artist picker on an
        edit form is on the critical path of [1.9](01-product.md)'s MVP — without it, crediting an
        artist means leaving the form to find an ID. Same for labels.
        Server-rendered, HTMX-driven, `hx-trigger="keyup changed delay:250ms"`, returning an `<li>`
        list from the trigram index. **It degrades to a plain text field plus a "find" button that
        submits**, per [11.3](11-stack.md) — the suggestion list is an enhancement, never the
        mechanism, and every picker must be usable without it.
        Also on the main search box, where it is a convenience rather than a requirement.
      - **Typo tolerance is the trigram fallback from [7.2](07-search-ux.md) and nothing more.** It
        runs only when the exact query returned no rows, and it says so ("Nothing found for X — did
        you mean Y"). Silent correction is how a search stops being predictable.
      - **"Similar releases": not built.** Similarity would have to be computed from genre and artist
        overlap, which is a recommendation feature with no data behind it at ~12,000 releases, and
        the honest version of "related" already exists as links: other pressings of this master,
        other releases by this artist, other releases on this label. Those are joins, not
        similarity.
- [x] 7.5 Key screens for the MVP — list the pages (artist, album, release, edit form, my collection, wantlist, search, profile, messages…).
      **Decision: 24 screens, listed here in full — call it ~30 templates, since a few rows bundle
      several near-identical ones.** The list is the point of the item: it is what turns
      [1.9](01-product.md)'s paragraph into something [section 15](15-roadmap.md) can size, and
      anything not on it is not in the MVP.

      | # | Screen | Path shape | Notes |
      |---|---|---|---|
      | 1 | Home | `/` | search box, recently added releases; not a dashboard |
      | 2 | Search results | `/search?q=` | three groups ([7.1](07-search-ux.md)) |
      | 3 | Release list, filtered | `/releases?…` | the component from [7.3](07-search-ux.md) |
      | 4 | Artist | `/artist/<id>/<slug>` | names, discography, members ([2.2.2](02-catalogue-model.md)) read-only |
      | 5 | Master (album) | `/master/<id>/<slug>` | credit, genres, its releases |
      | 6 | Release | `/release/<id>/<slug>` | the object: format, labels, identifiers, images, add-to-collection |
      | 7 | Label | `/label/<id>/<slug>` | releases on this label |
      | 8 | Entity edit form | `/<entity>/<id>/edit` | one template shape, four entities |
      | 9 | Entity create form | `/<entity>/new` | reached from search's "nothing found" and from pickers |
      | 10 | Duplicate warning | inline on create | [2.4.9](02-catalogue-model.md)'s advisory check |
      | 11 | Merge | `/<entity>/<id>/merge` | moderator only ([4.4](04-editing.md)); early, not late |
      | 12 | My collection | `/collection` | filters, live stats ([3.8](03-collection.md)), bulk actions ([3.10](03-collection.md)) |
      | 13 | My wantlist | `/wantlist` | |
      | 14 | Collection item form | `/collection/<id>/edit` | condition, notes, tags ([3.2](03-collection.md), [3.3](03-collection.md)) |
      | 15 | Public collection / wantlist | `/u/<nickname>/collection` | subject to [3.7](03-collection.md) |
      | 16 | Profile | `/u/<nickname>` | [6.3](06-accounts.md) |
      | 17 | Import upload | `/import` | file picker, format note |
      | 18 | Import job status | `/import/<id>` | progress, then the report ([10.2](10b-import.md)) |
      | 19 | Export | `/export` | one click, synchronous ([10.3.3](10c-export.md)) |
      | 20 | Register / verify / login | `/register`, `/verify`, `/login` | [6.1](06-accounts.md) |
      | 21 | Forgot / reset password | `/forgot`, `/reset` | [6.2](06-accounts.md) |
      | 22 | Settings | `/settings` | three toggles ([6.6](06-accounts.md)), email change, password change |
      | 23 | Account deletion | `/settings/delete` | export offered first ([6.5](06-accounts.md)) |
      | 24 | Static and errors | `/about`, `/terms`, `/privacy`, 403/404/500 | text owned by [section 13](13-legal.md) |

      **Not in the MVP, each already decided elsewhere:** the mailbox and conversation screens
      ([1.9](01-product.md) defers [section 5](05-messaging.md) entirely), the moderation queue and
      report screens ([4.6](04-editing.md)), the diff viewer and revision history
      ([4.2](04-editing.md) — the data is written, the screen is not built), the companies UI
      ([2.4.6](02-catalogue-model.md)), artist photos ([2.2.6](02-catalogue-model.md)), and any admin
      panel — [6.5](06-accounts.md) already established that disabling an account is an `UPDATE` a
      human runs.
      **The `<id>/<slug>` shape is provisional in one respect only:** what `<id>` actually is belongs
      to [10.4.6](10d-model-requirements.md) and is still open. The slug is cosmetic and ignored on
      read, so it may change freely; the identifier may not, once
      [10.3.2](10c-export.md) has put it in an exported file. This item is a second reason that item
      is urgent.
- [x] 7.6 Mobile: responsive / PWA / native app later? Barcode scanning with the camera — interesting?
      **Decision: responsive, mobile-first, one stylesheet. No PWA, no native app, no camera
      scanning.**
      **Responsive is not optional and is nearly free here.** A collector uses this standing in front
      of a shelf or in a shop; the pages are lists, forms and a grid of covers. Hand-written CSS
      ([11.3](11-stack.md)) with a single-column base and two breakpoints costs less than any
      framework would, and there is no component library imposing desktop assumptions to fight.
      **No PWA.** A service worker exists to serve an app offline and to receive push;
      [5.9](05-messaging.md) already turned down web push, and offline is meaningless for a catalogue
      whose entire content is server-side. A manifest with an icon would be the harmless 5% of a PWA,
      and even that is a thing to keep correct for no stated need.
      **No native app, at any stage.** It is a second codebase, a second release process and two
      store accounts, against [1.11](01-product.md)'s pace and [1.4](01-product.md)'s user count. This
      is not a deferral, it is out of the product.
      **Camera barcode scanning: no, and the field beside it is the point.** The value is *searching
      by barcode*, which [7.1](07-search-ux.md) already provides as a text field over
      [2.4.3](02-catalogue-model.md)'s normalised identifiers — and typing thirteen digits is not the
      hard part of cataloguing a record. Scanning would need either a vendored JS decoder (a large
      file, and [11.3](11-stack.md)'s no-build-step rule is the kind of line that only holds if it is
      never crossed once) or `BarcodeDetector`, which is not available across the browsers
      [9.6](09-nfr.md) will have to support. If it is ever reopened it is `BarcodeDetector` behind a
      feature check, filling the existing text field, with no library and no fallback path — the
      scanner types into the box, it does not become a second way in.
- [x] 7.7 Accessibility (a11y) and dark theme — do we keep these in the requirements?
      **Decision: both kept — accessibility as a short checkable list, dark theme via
      `prefers-color-scheme` with no toggle.**
      **This is the cheapest accessibility will ever be**, and that is the argument for doing it now
      rather than declaring it a P2. [11.3](11-stack.md) already requires that every core flow works
      as plain HTML with JavaScript disabled, which delivers keyboard operability, focus order and
      screen-reader semantics as a side effect. There is no widget library generating `<div>` buttons
      to remediate. The list, in full:
      - Semantic elements and one `<h1>` per page; landmarks (`<nav>`, `<main>`).
      - Every form control has a `<label>`; errors are associated with their field and repeated in a
        summary at the top of the form, because a failed submission is a full page render.
      - Visible focus styling, never removed.
      - Contrast at WCAG AA (4.5:1 body, 3:1 large) in **both** themes — the dark theme is where this
        is usually lost.
      - **`alt` text on every catalogue image, generated rather than requested.**
        [2.4.8](02-catalogue-model.md) gives every image a type, so the text is
        `"Front cover of <title> (<label>, <year>)"`; purely decorative images get `alt=""`. Asking
        uploaders to write alt text would produce empty strings and lies.
      - No `title`-attribute-only affordances, no colour as the sole carrier of meaning (relevant to
        [3.2](03-collection.md)'s condition grades, which must read as text, not as a coloured dot).
      **WCAG AA as a stated target, not as a certification claim.** We will not audit against the full
      standard; the list above is what we hold ourselves to and what a review can actually check.
      **Dark theme: `prefers-color-scheme` with CSS custom properties, and no in-app toggle.** The
      colours are a dozen variables redefined inside one media query. A toggle would need somewhere to
      persist the choice, and [6.6](06-accounts.md) closed the settings list at three columns
      explicitly against additions of this kind; a cookie-only toggle would be a fourth preference
      mechanism beside those columns. The OS preference is what users have already set, and honouring
      it silently is the whole feature.
- [x] 7.8 SEO: should public catalogue pages be indexable? (affects the choice of SSR — see [11.1](11-stack.md))
      **Decision: yes, indexable — and the choice this item worried about has already been made.**
      [11.1](11-stack.md) turned down the SPA and every page renders server-side
      ([11.5](11-stack.md)), so indexability costs nothing but a few tags. It is also the only way a
      stranger ever arrives, which [1.10](01-product.md)'s success criterion needs.
      **What is indexable:** artist, master, release and label pages, and `/releases` unfiltered.
      **What is not, and this is the part that matters:**
      - **Filtered and search URLs are `Disallow`ed in `robots.txt`.** [7.3](07-search-ux.md)'s facets
        multiply into a combinatorially infinite URL space; letting a crawler into it is a crawl trap
        that would also be, at [1.4](01-product.md)'s box size, the closest thing to a load test we
        will ever get.
      - **Collections and wantlists set to `link` get `noindex` and `X-Robots-Tag: noindex`, and never
        appear in the sitemap.** [3.7](03-collection.md) was explicit that `link` is an unguessable
        URL rather than an ACL; an unguessable URL in a search index is a guessable one.
        `private` returns 404 and needs no meta tag. `public` is indexable.
        [6.3](06-accounts.md) already forbids linking a `link` list from the profile; this is the same
        rule applied to robots.
      - **Everything behind login, every edit and create form, `/settings`, `/import` and `/export`:
        `noindex` and `Disallow`.**
      **Per page:** a `<title>` and meta description built from the entity, a self-referential
      `rel="canonical"` (so the cosmetic slug in [7.5](07-search-ux.md)'s URLs cannot fork a page into
      many), OpenGraph tags with the primary image ([2.4.8](02-catalogue-model.md)), and minimal
      schema.org JSON-LD (`MusicAlbum` on masters, `MusicRelease` on releases,
      `MusicGroup` on artists). JSON-LD is a template block with no infrastructure behind it; it is
      the one piece of SEO work here with any leverage.
      **A merged entity's old URL 301s to its winner** — [4.4](04-editing.md)'s `merged_into` pointer
      already keeps the old ID resolving, and a redirect is what makes that visible to a crawler
      rather than merely to a user. A soft-deleted entity ([4.5](04-editing.md)) returns 410 with its
      tombstone page.
      **Sitemap: one `sitemap.xml` generated on request from a single query**, no static file and no
      cron. At ~12,000 releases it is well inside the 50,000-URL limit and takes milliseconds; the day
      it does not, it becomes a sitemap index, and that day is not close.
      **Analytics is not decided here** — [13.4](13-legal.md) owns whether we have any at all.

## Working notes

- **2026-08-15 — Section closed. All 8 items decided.** Nothing was deferred and nothing new was
  added to the stack: search is three indexed columns and one trigger, filtering is `GROUP BY`,
  autocomplete is an HTMX fragment, and the theme is a media query. The only thing this section adds
  that did not exist before is `unaccent` beside `pg_trgm` — both stock extensions
  ([11.4](11-stack.md)).
- **2026-08-15 — Schema delta implied by the decisions above.** No new tables. Three columns and
  five indexes: `artist_name.search_tsv` and `label.search_tsv` as generated columns,
  `master.search_tsv` maintained by triggers on `master`, `master_credit` and `artist_name`; GIN
  indexes on all three plus `gin_trgm_ops` indexes on `artist_name.name` and `label.name`. Everything
  [7.3](07-search-ux.md) filters on already exists.
- **2026-08-15 — The one cache this design permits, and the rule that admits it.**
  `master.search_tsv` is denormalised on purpose, which sits beside [3.1](03-collection.md)'s
  invariant forbidding exactly that on a collection item. They are reconciled by a test worth reusing
  in later sections: **a cache is allowed when one statement rebuilds it from the truth and a stale
  value is therefore detectable.** A search vector passes; a title copied onto a collection item
  fails, because a copy that differs from its source is indistinguishable from an edit.
- **2026-08-15 — What this section hands on.**
  [Section 15](15-roadmap.md) inherits the concrete one: [7.5](07-search-ux.md)'s **24 screens** are
  the MVP's actual surface area, and the list is what [15.1](15-roadmap.md) and
  [15.3](15-roadmap.md) should slice rather than [1.9](01-product.md)'s paragraph.
  [10.4.6](10d-model-requirements.md) gains a second caller with a deadline: [7.5](07-search-ux.md)
  fixed every URL shape except what `<id>` is, and [10.3.2](10c-export.md) already makes that choice
  permanent on the first export.
  [Section 9](09-nfr.md) inherits two heaviest-query candidates for [9.1](09-nfr.md) — the faceted
  release list with live counts and the three-way search `UNION` — plus [9.6](09-nfr.md)'s browser
  support question, which [7.6](07-search-ux.md) leans on for the barcode refusal and
  [8.3](08-media.md) leans on for serving WebP with no fallback.
  [Section 12](12-infrastructure.md) inherits almost nothing, which is the intended result:
  no search daemon, no cache, no CDN, no build step.
  [Section 13](13-legal.md) owns the text of the four static pages [7.5](07-search-ux.md) lists and,
  at [13.4](13-legal.md), whether any analytics exist at all.
  [Section 14](14-security.md) should note that [7.3](07-search-ux.md)'s filters are `GET`
  parameters reflected into the page, so the escaping rule from
  [5.5](05-messaging.md)/[6.3](06-accounts.md) applies to query echoes too.
- **2026-08-15 — The known weakness, recorded so it is not rediscovered as a bug.** A Latin-typed
  query for a Cyrillic **title** finds nothing ([7.4](07-search-ux.md)). Artists are safe in any
  script ([2.2.1](02-catalogue-model.md)); titles have no variant rows. The workaround is the artist
  page, the fix is an additive `master_title` table, and the reason it is not built now is that
  nothing would fill it — no import channel carries a second title ([1.5](01-product.md)).
  Expect this to be the first search complaint from a real user, and note that it is one migration
  away from fixed.
- **2026-08-15 — Where this lands in the roadmap.** Search is not a late feature here: the artist and
  label **pickers** ([7.4](07-search-ux.md)) are a precondition of the edit forms, because every
  credit must resolve to an artist entity (NOTES.md), so the trigram index ships with the first edit
  form rather than with the search page. The faceted list ([7.3](07-search-ux.md)) is built once for
  the collection view and reused for the catalogue. [7.8](07-search-ux.md)'s tags and sitemap are the
  last thing on the list and take an afternoon.

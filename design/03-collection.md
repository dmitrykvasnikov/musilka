# 3. User collection and wantlist

**Priority:** P0

- [x] 3.1 Is the unit of a collection a concrete **release** (not an album)? Can a user own several copies of the same release (different conditions/prices)?
      **Decision:** **The unit is a concrete release, and yes — several copies are allowed.**
      A `collection_item` is an *instance of ownership*, not a `(user, release)` pair, so there is no
      unique constraint on that pair. Owning two copies is ordinary in this hobby (one to play, one
      sealed; two pressings that turned out to be the same edition), and the fields in
      [3.2](03-collection.md) differ per copy — a shared row could not hold two conditions.
      Release-level rather than master-level follows directly from [1.2](01-product.md) and
      [2.1.1](02-catalogue-model.md): the *edition* is the thing being collected, and after an
      import most masters are duplicates anyway ([2.1.2](02-catalogue-model.md)), so a master-level
      collection would be a collection of noise.
      **The item holds no copy of catalogue data** — no denormalised title, artist or year. That is
      what makes [3.11](03-collection.md)'s "release was edited" case a non-event rather than a
      synchronisation problem.
- [x] 3.2 Collection item fields: media and sleeve condition (Goldmine grading: M/NM/VG+/VG/G+/G/P), purchase date, purchase place, purchase price + currency, current valuation, personal notes, personal rating, "listened / not listened".
      **Decision:** condition, date, place, note and rating — **no money of any kind**, and no
      listened flag.

      ```
      collection_item(id, user_id, release_id, added_at,
                      media_condition_id, sleeve_condition_id,
                      purchased_on, purchased_at, note, rating)
      ```

      **Condition** is two separate fields, both nullable, drawn from a seeded ordered vocabulary —
      the curated-set invariant, and here the set is genuinely closed because Goldmine is a published
      scale: `M, NM, VG+, VG, G+, G, F, P`. The lookup carries a `rank` so that "show me everything
      VG+ or better" is a comparison rather than a list of strings. Sleeve needs three values media
      does not: `Generic`, `No Cover`, `Not Graded`. Nullable because the catalogue is thin by
      construction and so is a collection — most imported rows carry no grade at all.
      **Purchase date and place** are kept; place is free text (a shop, a fair, a person), because it
      is a memory rather than a field to group by.
      **Rating** is 1–5 integers, personal and independent of any catalogue-wide rating (we have
      none, and at our user count an aggregate rating would be one person's opinion wearing a
      average's clothes).
      **No purchase price, no currency, no current valuation.** Chosen 2026-08-15 to keep anything
      money-shaped out of the model for now, consistent with [1.1](01-product.md)'s refusal to pay
      up front for a commercial future.
      - *Valuation* would be dead on arrival regardless: [1.5](01-product.md) forbids fetching market
        data, so it could only be a number the user types once and never revisits, feeding a "total
        collection value" statistic that is fiction by the second year.
      - *Purchase price* is a real thing collectors record, and it is **deferred rather than
        refused**. Reopening it is two nullable columns (`price_amount`, `price_currency` as ISO
        4217) on a table nothing else references — additive, no backfill, no change of meaning. If
        it returns, it is owner-only like the note ([3.7](03-collection.md)) and is **never
        converted between currencies**, since we have no rate source and may not fetch one.
      - **The one real cost, and the obligation it creates:** a Discogs collection export can carry a
        price column, and what we do not store on import is gone for that upload. It is recoverable
        (the file is the user's), so the price is acceptable — but the importer must **report the
        columns it discarded** rather than dropping them silently. Recorded against
        [10.2](10b-import.md).
      **No "listened / not listened".** A boolean set once at import and never maintained becomes a
      field that lies; anyone who genuinely tracks a backlog can tag it ([3.3](03-collection.md)),
      which is the same information without a column that decays.
- [x] 3.3 Folders (as on Discogs) or tags? Or both?
      **Decision:** **Tags only.** One mechanism, many per item: `collection_item_tag(item_id,
      tag_id)`, tags owned by the user and freely named (this is *personal* labelling, so the
      curated-vocabulary invariant does not apply — nobody else has to group by them).
      Tags strictly contain folders: a Discogs folder becomes a tag on import with no loss, and an
      item can be both "to sell" and "Soviet pressings", which a folder tree cannot express without
      duplication.
      Rejected: **folders**, despite being what the target user knows, because single membership is a
      constraint we would have to justify and cannot. Rejected: **both**, which is two grouping
      mechanisms the user must choose between every time they file something.
      Accepted cost: no sidebar tree. Grouping is a filter over tags, which puts more weight on
      [section 7](07-search-ux.md) to make filtering feel like navigation.
- [x] 3.4 Custom user-defined fields — needed?
      **Decision:** **No.** A per-user schema is a real feature — definitions, types, validation,
      rendering, export columns, API representation — bought for a need nobody has demonstrated, and
      [3.3](03-collection.md)'s tags plus a free-text note already absorb most of what people
      actually use them for.
      **What this means for import**, which is the only place it bites: Discogs custom-field columns
      have nowhere to go. They are appended to the item's `note`, labelled with their column name, and
      **reported to the user as folded**, under the same obligation [3.2](03-collection.md) puts on
      dropped price columns. Lossy in shape, not in content.
- [x] 3.5 Wantlist: priority, maximum price, note, "release was updated" notification. A separate entity or a flag on the collection item?
      **Decision:** **A separate table**, not a flag.

      ```
      wantlist_item(id, user_id, release_id, priority, note, added_at)
      ```

      A flag on `collection_item` would mean creating a row for something the user does not own,
      which corrupts every count in [3.8](03-collection.md) and every export in
      [10.3](10c-export.md); each would then need a "but not the wanted ones" clause forever.
      **Release-level, like the collection** ([3.1](03-collection.md)). Considered and rejected:
      wanting a *master* ("any pressing of this album"), which is a real collector behaviour but
      unusable while imports mint a duplicate master per row ([2.1.2](02-catalogue-model.md)).
      Additive later if master merge ever makes masters trustworthy.
      **`priority`** is a small ordered vocabulary (low / normal / high), not a free integer — it
      exists to sort, and three levels sort fine.
      **No maximum price**, consistent with [3.2](03-collection.md); it returns with purchase price
      or not at all.
      **No "release was updated" notification in the MVP.** There is no notification system in
      [1.9](01-product.md)'s scope. It is cheap later precisely because [4.2](04-editing.md) stores a
      revision per mutation: the query is "revisions on releases in my wantlist since I last looked".
- [x] 3.6 "For sale" / "for trade" statuses — yes/no (see [1.7](01-product.md)).
      **Decision:** **No**, directly from the [1.7](01-product.md) invariant — nothing order-shaped
      in the model, at any stage. A "for sale" status is seller state; that it would live on a
      collection item rather than a catalogue entity changes nothing.
      A user who wants to remember which records they intend to part with can tag them
      ([3.3](03-collection.md)). That is a private label with no counterparty, and it is deliberately
      not a status other users can see or search on — the moment it is visible, it is a listing.
- [x] 3.7 Collection privacy: public / by link / private; configured for the whole collection or per item (e.g. hiding prices). Must be enforced in the API too — see [10.1.5](10a-public-api.md).
      **Decision:** **Two settings, one per list, plus fields that are owner-only unconditionally.**

      ```
      user.collection_visibility   public | link | private
      user.wantlist_visibility     public | link | private

      never visible to anyone but the owner, with no setting to change it:
        note, purchased_on, purchased_at   (and price, if 3.2 ever reopens)
      ```

      Separate settings because the two lists have opposite social purposes: a wantlist is worth
      publishing precisely so other people see it, while a collection is a list of valuable objects
      in your home. Forcing them to share one setting makes the useful case impossible.
      `link` means an unguessable URL, not an ACL — the list is served to anyone holding it, and it
      is honest about that rather than pretending to be security.
      Rejected: **per-item visibility**. It is genuinely wanted by some collectors, but it puts a
      clause in every listing query and a control on every row, to hide things a private collection
      already hides.
      **Enforcement is in the service layer** ([10.4.4](10d-model-requirements.md)), not in the
      views, so [10.1.5](10a-public-api.md) inherits it rather than reimplementing it. A visibility
      rule that lives in a template is a rule the API does not have.
- [x] 3.8 Collection statistics: count, total value, breakdown by genre/year/format/country — what do we show in the MVP.
      **Decision:** **Counts and breakdowns only, computed live, no money.**
      In the MVP: total items and distinct releases; distinct artists; breakdown by format
      descriptor, by genre, by decade, and by country. Nothing else.
      **No total value** — [3.2](03-collection.md) stores no prices, so the tile has nothing behind
      it. It returns with prices, and until then its absence is honest rather than a gap.
      **Computed live on every request, with no materialised counters and no cache.** A collection is
      thousands of rows, not millions ([1.4](01-product.md)); a `GROUP BY` answers this in
      milliseconds, and a denormalised counter is a thing that drifts. Cite this item if a
      statistics-shaped caching proposal appears in [section 9](09-nfr.md).
      Breakdowns read through to the catalogue (genre lives on the master, country on the release),
      which is another reason [3.1](03-collection.md) forbids denormalising catalogue data onto the
      item.
- [x] 3.9 Collection export (CSV/JSON) — needed? (I would do it right away: it matters for trust; formats and mechanics — [section 10.3](10c-export.md))
      **Decision:** **Yes, and early** — it is inside [1.9](01-product.md)'s MVP round trip and
      recommendation 9's sequencing rule (export before import), which
      [section 15](15-roadmap.md) has already inherited.
      The argument is trust, not utility: a collector asked to re-enter years of cataloguing needs to
      know they can leave. Export also happens to be the cheapest possible proof that the model holds
      together, and it needs no external source, which makes it the natural first vertical slice.
      Formats, column set and mechanics are [10.3](10c-export.md)'s to decide. Two things this
      section binds it to: owner-only fields ([3.7](03-collection.md)) appear in the owner's own
      export and nowhere else, and tags ([3.3](03-collection.md)) must survive the round trip, since
      they are the only grouping a user has.
- [x] 3.10 Bulk operations: add many releases, change the folder of a selection, delete.
      **Decision:** **Three, deliberately: tag a selection, untag a selection, remove a selection.**
      "Add many releases" is not a UI feature — it is what import does ([10.2](10b-import.md)), and
      building a second bulk-add path in the interface duplicates the importer badly.
      No bulk condition or rating editing: those are per-copy facts ([3.1](03-collection.md)), and a
      bulk grade is almost always wrong. Anyone who really needs it can export, edit, and re-import
      once import exists.
      "Remove" here is removal from a collection — deleting the user's own row, a genuine hard
      delete. It is unrelated to [4.5](04-editing.md)'s soft deletion, which protects *catalogue*
      entities other people point at; nobody points at your collection item.
- [x] 3.11 What happens to a collection item if the release in the catalogue is edited/deleted/merged into another? (see [4.4](04-editing.md), [4.5](04-editing.md))
      **Decision:** three cases, all answered by [section 4](04-editing.md), none of them requiring
      anything of this section beyond holding a foreign key.
      1. **Edited — nothing happens.** The item stores no copy of catalogue data
         ([3.1](03-collection.md)), so a corrected title simply appears. This is the whole reason
         that rule exists.
      2. **Merged — the item follows.** [4.4](04-editing.md) repoints every reference at merge time,
         so `release_id` is rewritten to the winner. The user is not notified and does not need to
         be: the object in their hands did not change, only our record of which entry describes it.
      3. **Deleted — the item survives.** [4.5](04-editing.md) is soft-delete only, so the row keeps
         resolving; the item renders with a marker saying the catalogue entry was removed, and it
         still counts in [3.8](03-collection.md). It is never hidden and never silently dropped —
         the invariant is that nothing a collection points at is hard-deleted, and the visible
         consequence is that a user's list never shrinks without their action.

## Working notes

- **2026-08-15 — Tags are the load-bearing decision here.** Having refused folders
  ([3.3](03-collection.md)), custom fields ([3.4](03-collection.md)), for-sale status
  ([3.6](03-collection.md)) and per-item privacy ([3.7](03-collection.md)), tags are now the answer
  to four separate user needs. If [section 7](07-search-ux.md) makes tag filtering awkward, all four
  refusals get worse at once — this is the item to revisit first if the collection UI feels thin.
- **2026-08-15 — Money is deferred in exactly one place, and it is written down.** `price_amount` and
  `price_currency` on `collection_item`, plus `max_price` on `wantlist_item`, plus the total-value
  tile in [3.8](03-collection.md). Nothing else in the design has to move if they return. Kept here
  so a future reopening is a checklist rather than an investigation.

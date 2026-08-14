# Cross-cutting design notes

Only what belongs to **no single section** lives here. A note about one section goes in that
section file's `Working notes`.

Keep entries short and dated (`YYYY-MM-DD`). When a note hardens into a decision, move it into the
relevant item as a `**Decision:**` block and delete it from here.

## Agreed invariants

Rules that hold across the whole design and must not be quietly broken by a later decision.
Each one names the item it was decided in.

_(none yet)_

## Rejected approaches

What we considered and turned down, and why — so we do not re-litigate it in three months.

_(none yet)_

## Constraints discovered

Facts found while designing that constrain later choices (external ToS, format quirks, volumes).

_(none yet)_

## Open questions blocking other items

Questions that must be answered before some other item can be closed. Format: what is blocked ← what we are waiting on.

_(none yet)_

## Needs verification against reality

Claims in the agenda that are assumptions until checked against a real file, API or document.

- [10.2.2](10b-import.md) — the actual Discogs collection CSV column set, on a real export.
- [10.2.7](10b-import.md) — whether `instance_id` is present in the CSV export.
- [10.5.1](10e-legal-sources.md) — current Discogs API terms and rate limits.

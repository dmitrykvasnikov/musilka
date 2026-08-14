# 15. Roadmap

**Priority:** P0 — the outcome of the discussion

- [ ] 15.1 Slice everything above into stages: E0 (skeleton), E1 (MVP), E2, E3…
- [ ] 15.2 Record an explicit **out-of-scope** list for the first version.
- [ ] 15.3 Define the "vertical slice" for the first week of work (for example: artist → album → release → add to collection, without edits and messages).
- [ ] 15.4 Risks and open questions requiring research (data import, search, volumes).
- [ ] 15.5 Decide separately at which stage the public API ([10.1](10a-public-api.md)), import ([10.2](10b-import.md)) and export ([10.3](10c-export.md)) appear, and which "hooks" for them already land in E0–E1 ([10.4](10d-model-requirements.md)).
- [ ] 15.6 Once the key items are closed — produce `PLAN.md` with concrete tasks.

## Working notes

**2026-08-14 — shape of the plan (proposal, not decided).** Mirror the `design/` split, but cut by
**stage rather than by design section**: `plan/E0-skeleton.md`, `plan/E1-mvp.md`, … A coding session
inside E1 never needs E3, which is the property that makes the split pay; cutting by module instead
would mean opening most of the folder every session.

- Do not create `plan/` before [15.1](15-roadmap.md) fixes the stage list — the file names come from
  it, and E0 alone may be small enough for a single file.
- Task IDs permanent, same rule as design items, and each task cites the design item that justifies
  it (`T-14 external_ids table (per 10.4.1)`). That backlink is the main reason to keep the plan in
  the repo next to the design.
- No stored progress counters — derive with `grep`, as in `CLAUDE.md`.
- Depends on [12.8](12-infrastructure.md): if work is tracked in a real issue tracker rather than
  markdown, `plan/` shrinks to a thin index and most of this note is moot. Settle 12.8 first.

# Musilka

A catalogue of music releases with user collections, wantlists, catalogue editing and messaging —
a Discogs-shaped project. Currently in the **design phase**: no code yet, the work is closing the
open questions in `design/`.

## Language

All chat, documents, commit messages and code comments are in **English**. The product's own UI
language is a separate, still-undecided question ([1.8](design/01-product.md)) — do not assume it
follows from this rule.

## Design sessions

The agenda lives in `design/`, one file per section:

| File | Section | Priority |
|------|---------|----------|
| `01-product.md` | Product: scope and goals | P0 |
| `02-catalogue-model.md` | Catalogue domain model | P0 |
| `03-collection.md` | User collection and wantlist | P0 |
| `04-editing.md` | User editing of the catalogue | P0 |
| `05-messaging.md` | Messaging between users | P0 (module) |
| `06-accounts.md` | Accounts and profiles | P1 |
| `07-search-ux.md` | Search, navigation, UX | P1 |
| `08-media.md` | Images and media | P1 |
| `09-nfr.md` | Non-functional requirements | P1 |
| `10a-public-api.md` | 10.1 Our public API | P0 |
| `10b-import.md` | 10.2 Importing a user's collection | P0 |
| `10c-export.md` | 10.3 Export | P0 |
| `10d-model-requirements.md` | 10.4 What this demands from the data model | P0 |
| `10e-legal-sources.md` | 10.5 Legal aspects, ToS of external sources | P1 |
| `11-stack.md` | Technology stack | P0 (after 1–5) |
| `12-infrastructure.md` | Infrastructure and process | P2 |
| `13-legal.md` | Legal and organisational | P1 |
| `14-security.md` | Security | P1 |
| `15-roadmap.md` | Roadmap | P0 (outcome) |

Plus `design/NOTES.md` (cross-cutting notes) and `design/recommendations.md` (opening proposals).

**Reading:** read `design/NOTES.md` and **only the section files under discussion**. Do not read all
of `design/` — that is the whole reason it is split.

**Progress is derived, never stored.** There is no status table to keep in sync; count from the
files when you need to know:

```sh
grep -c '^- \[ \]' design/*.md          # open items per file
grep -rn '^- \[~\]' design/             # deferred, waiting on something
```

**Recording a decision:** in the section file, under its item, as

```
- [x] 4.3 Question text stays exactly as it was
      **Decision:** what we decided and why.
```

Never delete or reword the question — the decision goes underneath it. Updating the checkbox
(`[ ]` → `[x]`, `[~]` deferred, `[-]` dropped) is the whole bookkeeping; nothing else to update.

**Item IDs are permanent.** `10.4.1` stays `10.4.1` forever, even if neighbouring items are removed.
Never renumber — cross-references and git history hang on those IDs. New items take the next free
number in their subsection. `grep -rn "10.4.1" design/` finds every mention.

**Notes:** a note about one section goes in that file's `## Working notes`. `design/NOTES.md` is for
cross-cutting material only — invariants spanning sections, rejected approaches, discovered
constraints, open blocking questions. Do not copy decisions there; the section file is the single
source of truth for what was decided.

**Do not invent decisions.** If an item is unticked, it is open, and the recommendations in
`design/recommendations.md` are proposals rather than settled choices. When work needs an unsettled
answer, ask rather than assume.

## Repository

Nothing is implemented yet. **The design is closed** — every item is decided except
[10.2.2](design/10b-import.md), [12.1](design/12-infrastructure.md) and
[12.5](design/12-infrastructure.md), which are `[~]` deferred by choice.

## Plan sessions

The backlog lives in `plan/`, one file per **stage** ([15.6](design/15-roadmap.md),
[12.8](design/12-infrastructure.md)), and it is where implementation work is tracked.

| File | Stage |
|------|-------|
| `plan/E0-skeleton.md` | E0 — running under systemd behind nginx, CI green, backups restore (T-1 … T-33, T-191 … T-194) |
| `plan/E1-mvp.md` | E1 — the collector's round trip, nine slices (T-34 … T-160) |
| `plan/E2-messaging.md` | E2 — messaging (T-161 … T-173) |
| `plan/E3-depth.md` | E3 — depth (T-174 … T-190) |

**Localhost is the target environment** until further notice: [1.10](design/01-product.md) was
amended on 2026-08-15 to withdraw the "a second person uses it" criterion, so a public deployment is
deferred and non-gating. The rule that keeps that honest — an invariant in `design/NOTES.md` —
is that **anything localhost cannot exercise is marked as unexercised, never as done**, and exactly
four things are on that list: TLS issuance, mail deliverability, an off-box backup, an external
pinger.

Plus `plan/README.md` (index and conventions). **Reading:** the stage file being worked on, and the
design sections its tasks cite — not the whole folder, which is the reason for the split.

Same conventions as `design/`: **task IDs are permanent and global** (`T-58` stays `T-58`; a new task
takes the next free number in the whole directory), progress is derived (`grep -c '^- \[ \]'
plan/*.md`), and the checkbox is the whole bookkeeping.

**Every task cites the design item that justifies it** — `T-58 external_id table (per 10.4.1)` — so
that `grep -rn "10.4.1" design/ plan/` finds both the decision and the work. A task without its
citation is a defect.

**`plan/` never decides anything.** If work needs a design answer that is not there, reopen the
design item; do not settle it in a task.

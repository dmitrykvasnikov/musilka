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

Nothing is implemented yet. `PLAN.md` gets written once the key items are closed
([15.6](design/15-roadmap.md)); implementation starts from there.

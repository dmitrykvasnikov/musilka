# Plan

The backlog, and the single source of truth for what is being built
([12.8](../design/12-infrastructure.md)). GitHub Issues stays enabled as an inbox for reports from
outside; anything acted on becomes a task here, citing the issue.

Cut by **stage**, not by design section — the file names come from
[15.1](../design/15-roadmap.md). A coding session inside E1 never needs E3, which is the property
that makes the split pay.

| File | Stage | What it is | Rough size |
|------|-------|------------|------------|
| [`E0-skeleton.md`](E0-skeleton.md) | E0 | It is on the internet, CI is green, backups restore | 3–4 weeks |
| [`E1-mvp.md`](E1-mvp.md) | E1 | [1.9](../design/01-product.md)'s collector's round trip, in nine slices | 4–5 months |
| [`E2-messaging.md`](E2-messaging.md) | E2 | [Section 5](../design/05-messaging.md) as a plain mailbox | 3–4 weeks |
| [`E3-depth.md`](E3-depth.md) | E3 | Moderation tooling and the schema shipped switched off | open-ended |

**E4 has no file on purpose.** The public API ([10.1](../design/10a-public-api.md)) and the
catalogue dump ([10.3.4](../design/10c-export.md)) are named in
[15.1](../design/15-roadmap.md) so that "later" has a name, not because either is planned. Writing
tasks for them would make them look scheduled.

## Conventions

Deliberately the same as `design/`, so there is one set of rules to remember.

**A task is one sitting.** [1.11](../design/01-product.md) fixes the pace at 2–3 hours a day, and a
task that cannot be finished in one is two tasks. The sizes in the table above already carry
[1.11](../design/01-product.md)'s 2–3× multiplier.

**Every task cites the design item that justifies it.** That backlink is the main reason the plan
lives in the repository next to the design: `grep -rn "10.4.1" design/ plan/` finds both the decision
and the work.

```
  - [ ] T-58 external_id table (per 10.4.1)
        Anything worth saying about how, or what it is waiting on.
```

(indented here only so that the example does not turn up in the `grep` counts below)

**Task IDs are permanent and global.** `T-14` stays `T-14` for ever, across every file, even if
neighbouring tasks are removed. New tasks take the next free number in the whole directory. Never
renumber — commit messages cite these ([12.7](../design/12-infrastructure.md)).

**Checkbox states**, as in `design/`:

| | |
|---|---|
| `[ ]` | open |
| `[x]` | done |
| `[~]` | blocked or deferred, waiting on something named in the task |
| `[-]` | dropped |

**Progress is derived, never stored.** There is no status table, no percentage and no board.

```sh
grep -c '^- \[ \]' plan/*.md          # open tasks per stage
grep -rn '^- \[~\]' plan/             # blocked, and on what
grep -rn 'T-14' plan/ design/         # everything about one task
```

**Never delete or reword a task that has been done.** Tick it. If it turned out to be wrong, mark it
`[-]` and write why underneath — the same discipline `design/` uses for decisions.

## What this plan does not decide

Nothing here reopens a design decision. Where a task looks like a choice, the choice was made in
`design/` and the task is its implementation; where the design left something open, the task carries
`[~]` and names what it is waiting on. The three open things in the whole design as of
2026-08-15 are [10.2.2](../design/10b-import.md) (a real Discogs collection export, blocks E1.7
alone) and [12.1](../design/12-infrastructure.md)/[12.5](../design/12-infrastructure.md) (hosting
and mail vendors, block the first deploy).

Read [`design/NOTES.md`](../design/NOTES.md) before adding a task. Its invariants are what stop a
plausible-looking task from quietly breaking a closed section.

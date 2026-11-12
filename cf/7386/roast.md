# Design pre-review: standby PANIC on restart after VM truncation

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7386](https://commitfest.postgresql.org/patch/7386/); [thread](https://www.postgresql.org/message-id/flat/CAL4mQLAp562c1rCgg2Dqx6TBSdOk6vOvaLFjwFG9E6k4uwvJJw%40mail.gmail.com), 11 of 11 messages read; no code read.

## Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium). No objection survived at levels 1a, 1b or 2. The thread states the problem with a reproducer and a field report, and three people confirmed it on four branches. v5 forgets invalid VM pages when truncation is replayed, as XLogTruncateRelation() already does for the main fork, so it follows existing practice. Confidence is medium because the material does not show v5's commit message, file list or code.

Not checked: The material shows only the v1 test patch, so v5's code, commit message and file list were not judged. Whether forgetting invalid VM pages at truncation replay could hide a real invalid-page reference cannot be checked without the code.

## Prior attempts

None found before 2026-09-27.

## Objections a reviewer is likely to raise

No objection held up in the counter-check.

## Questions for the author

1. Does v5 pass with wal_consistency_checking = all on every branch it targets?
2. Does v5 cover the DELETE, HOT update and cross-page update cases that the v4 test checks with full_page_writes = off?
3. Which branches does v5 target, and which test goes into 17, where the v5 test does not apply?
4. Why do 18.4 and 17.10 pass the reproducer while 18.6 and 17.11 fail, if the bug dates from the first VM-clear redo commit?
5. Will the CommitFest entry be updated to point at v5 instead of the v4 approach?

## Touch points

- **WAL / replication / standby:** Both fixes change how the startup process replays VM-clear and truncation records on a standby.
- **backpatchability:** The entry targets stable branches, and the v5 test does not apply on 17.
- **maintenance burden ("who maintains this in 10 years"):** The fix adds invalid-page handling for the VM fork in redo code that needs care on every supported branch.

## Who would commit it, and when

- Owner: qualified committer engaged. Melanie Plageman wrote v5 and prefers the truncation-replay fix; the owner needs to know the visibility map redo code and invalid-page tracking in recovery.
- Priority: proceed; audience a sizeable minority; review cost medium. A standby PANIC on restart in released minor versions affects users of streaming replication, and v5 reuses the main-fork mechanism without new user-visible surface. Review cost is medium because no fix patch was read and the change is in recovery code that goes to stable branches.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

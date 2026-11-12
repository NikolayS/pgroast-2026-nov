# Design pre-review: Fix SSI conflicts between index-only scans and heap writes

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7344](https://commitfest.postgresql.org/patch/7344/); [thread](https://www.postgresql.org/message-id/flat/CAHWVJhFfP6YmMmQwuUb%3Da5w%2BNcSOv1Aw%2BZN5Z%2BN_yUGeyG7kHw%40mail.gmail.com), 3 of 3 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). The problem is stated with a reproducer and is a correctness bug in SERIALIZABLE, so levels 1a and 1b pass. The one surviving objection, maintenance-fit-1, is a fixable design point: the proposal does not state the new ordering rule or say which other paths that clear the all-visible bit must follow it. Someone who knows SSI and the visibility map has to confirm that the two rechecks close the race. No one but the author has posted.

Not checked: Without the code, this review cannot confirm that the two rechecks close both windows, that the writer-side check is safe where it runs, or what the visibilitymap.c change does.

## Prior attempts

None found before 2026-09-24.

## Objections a reviewer is likely to raise

### 1. Correctness rests on a writer-side ordering rule that nothing enforces

A later patch adds or reworks a heap path that clears an all-visible bit, for example a new update fast path, and omits the second conflict-in check. Every existing test still passes, and SERIALIZABLE users again commit anomalous schedules. Nobody notices until another GDB-driven report arrives.

The fix adds a second conflict check at two call sites, heap_delete() and heap_update(), and pairs it with a reader-side recheck. The commit message does not state the new invariant: any code that clears an all-visible bit while a predicate lock could be taken must recheck afterwards. The proposal also does not say whether other VM-clearing paths, or table AMs that support index-only scans, must follow the same rule. Nothing turns this rule into an assertion or a shared helper.

**Question:** Can the invariant be stated in a comment and put into one helper or assertion that every VM-clearing path must use, with the other paths checked against it?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#1](https://www.postgresql.org/message-id/CAHWVJhFfP6YmMmQwuUb%3Da5w%2BNcSOv1Aw%2BZN5Z%2BN_yUGeyG7kHw%40mail.gmail.com); `maintenance-fit-1`</sub>

## Questions for the author

1. Which other paths that clear an all-visible bit, such as heap_insert or heap_lock_tuple, have the same race, and why are they safe if left unchanged?
2. Can the commit message and a code comment state the new ordering rule between clearing the all-visible bit and the conflict-in check?
3. Which supported branches have the bug, and is the patch meant for back-patching?
4. What changes in visibilitymap.c, and does it alter any exported function signature?
5. What does the second conflict-in check and the VM recheck cost on an update-heavy workload and on a SERIALIZABLE index-only-scan workload?

## Touch points

- **ABI / extension API:** The patch touches visibilitymap.c without saying what changes; other table AMs with index-only scans may need the same rule.
- **backpatchability:** This is a bug fix for SERIALIZABLE, but the proposal does not say which supported branches have the bug.
- **maintenance burden ("who maintains this in 10 years"):** Every heap path that clears an all-visible bit must now repeat the conflict-in check, and nothing enforces that.
- **performance on the hot path:** heap_delete(), heap_update() and SERIALIZABLE index-only scans get an extra check; no measurement is given.

## Who would commit it, and when

- Owner: silent. Someone who knows the SSI predicate-lock code and the heap visibility-map locking rules would have to own this.
- Priority: needs champion; audience a sizeable minority; review cost medium. The bug affects users who rely on SERIALIZABLE, and the patch is small, about 19 KiB with tests. It touches heapam.c and SSI ordering, which few reviewers can sign off, and no one has replied in the thread.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

# Design pre-review: Fix: VACUUM FULL or CREATE INDEX fails with error "missing chunk number 0 for toast value XXX"

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6824](https://commitfest.postgresql.org/patch/6824/); [thread](https://www.postgresql.org/message-id/flat/1779078055.630771314%40fmail2.qdit), 7 of 7 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). The bug is reproducible, demand is shown, and no surviving objection questions the goal. Two fixable design points remain. standards-compat-1 asks what the fix reports when a chunk is missing, so that real corruption stays visible. maintenance-fit-1 asks where the tolerant handling lives, since CLUSTER, VACUUM FULL and REPACK still need it. Both need someone who knows heap visibility and TOAST to sign.

Not checked: The aggressive-detoasting patch is in another thread and was not read. Its error reporting, its cost on wide toasted columns and its effect on the table AM interface could not be judged.

## Prior attempts

None found before 2026-05-18.

## Objections a reviewer is likely to raise

### 1. Skipping tuples with missing TOAST data hides real corruption from CREATE INDEX

A table has a real lost TOAST chunk on a RECENTLY_DEAD tuple. Today CREATE INDEX or VACUUM FULL stops with XX001, and monitoring alerts. With the skip-on-missing design the command succeeds silently, and the operator learns of the damage only if a later read of a live tuple hits it.

Message #2 proposes ignoring TOAST visibility errors during index builds, and message #3 links a patch based on aggressive detoasting. Message #4 raises the masking concern, and no message answers it. Message #6 shows operators depend on XX001 meaning corruption. The proposal does not say whether a skipped chunk is logged, gets another SQLSTATE, or is limited to tuples the code can prove were removed by the race, and that choice changes what existing monitoring sees after upgrade or a back-patch.

**Question:** When a RECENTLY_DEAD tuple's TOAST data is missing, what does the fix report, and how does it tell this race apart from real corruption?

<sub>fixable, level 2, design; messages [#2](https://www.postgresql.org/message-id/CAEze2WgRUfoR%3DtSQQm8zexVNeBbEYi9ZSPztrK63ObyfPxKpBw%40mail.gmail.com), [#3](https://www.postgresql.org/message-id/ak2eB2XgrvDw3VGN%40paquier.xyz), [#4](https://www.postgresql.org/message-id/1783496056.499330176%40fmail1.qdit), [#6](https://www.postgresql.org/message-id/4D42CDAE-C881-4AE4-886B-C83EF2468C08%40yandex-team.ru); `standards-compat-1`</sub>

### 2. Missing-chunk tolerance lives in each caller, so new paths can regress

Someone later adds or changes a path that reads TOAST data of RECENTLY_DEAD tuples, such as a REPACK variant, a table AM's index build scan, or an extension's rewrite. That path does not repeat the detoast-and-tolerate step, and no test or assertion flags it. Users then get XX001 "missing chunk" errors again, found only through corruption alerts in production.

Message #2 says missing TOAST data is to be expected for RECENTLY_DEAD tuples, so every reader of such tuples must handle it. The fix in message #3 covers only the heap index build. It says CLUSTER, VACUUM FULL and REPACK need separate, deeper work. The material shows no shared helper or table AM contract, so each table AM and each future maintenance path must find out about this race on its own.

**Question:** Will the tolerant handling sit in one shared helper or a documented table AM rule that future copy and build paths must use, and what test fails if a path skips it?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#2](https://www.postgresql.org/message-id/CAEze2WgRUfoR%3DtSQQm8zexVNeBbEYi9ZSPztrK63ObyfPxKpBw%40mail.gmail.com), [#3](https://www.postgresql.org/message-id/ak2eB2XgrvDw3VGN%40paquier.xyz); `maintenance-fit-1`</sub>

## Questions for the author

1. When a RECENTLY_DEAD tuple's TOAST data is missing, what does the fix report, and how does it tell this race apart from real corruption?
2. Will the tolerant handling sit in one shared helper that index build, CLUSTER, VACUUM FULL and REPACK all use, and what test fails if a path skips it?
3. How will copy_for_cluster() handle a RECENTLY_DEAD tuple whose external TOAST value is copied by reference and is already gone?
4. Which patch should reviewers of this CF entry read: the OldestXmin prototype, the aggressive-detoasting patch, or both?
5. Will the fix be back-patched to the supported branches where the bug reproduces, and does it change any struct or callback that extensions use?

## Touch points

- **ABI / extension API:** The prototype adds fields to VacuumParams in vacuum.h, and the detoasting fix changes what the heap index build scan passes to callbacks.
- **backpatchability:** The entry is tagged Bugfix and the bug reproduces on 17.10, but the target version is 20.
- **maintenance burden ("who maintains this in 10 years"):** Each path that reads TOAST data of RECENTLY_DEAD tuples needs the tolerant handling, and CLUSTER, VACUUM FULL and REPACK are not yet covered.
- **performance on the hot path:** Aggressive detoasting runs in every index build scan, and its cost on large toasted columns is not shown.

## Who would commit it, and when

- Owner: qualified committer engaged. Michael Paquier is listed as author and posted the detoasting patch in message #3. The owner must know heap visibility horizons, TOAST access and the table rewrite code.
- Priority: proceed; audience a sizeable minority; review cost medium. The race raises XX001 corruption alerts in a production fleet and was reported twice, so the fix has clear value. The detoasting patch was not read; it touches heap index builds, and the rewrite paths still need a design.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

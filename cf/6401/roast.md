# Design pre-review: Resetting snapshots during the first phase of [CREATE |RE]INDEX CONCURRENTLY

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6401](https://commitfest.postgresql.org/patch/6401/); [thread](https://www.postgresql.org/message-id/flat/CADzfLwWAiimwngq38x8Ji26AtYEmSqnFD7aa4kVGF4ObDswHMA%40mail.gmail.com), 13 of 13 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). No surviving objection is fatal, and demand for the goal is established by the PG14 feature that [P1] reverted only for safety. The open points are fixable: measurements (maintenance-fit-2, demand-1), a written invariant for the unique-build path (prior-attempts-1), and enforcement of the new AM rules (maintenance-fit-1). The design changes snapshot handling where an error gives a silently incomplete index, so a committer who knows that area has to sign for it.

Not checked: The code was not read, so the claims that xmin stays valid during a reset, that the parallel handshake is correct, and that the unique recheck is complete are unchecked. Only 3 of 13 messages were shown.

## Prior attempts

- [Revert changes to CONCURRENTLY that "sped up" Xmin advance](https://www.postgresql.org/message-id/flat/17485-396609c6925b982d@postgresql.org) (similar idea; Reverted, 2022-05-31). "In other words, Xmin advancement was sped up, but at the cost of corrupting the resulting index." Different now: P1 let vacuum ignore the build backend's xmin while the build kept an old snapshot. This proposal replaces the snapshot itself, so the build always holds a registered snapshot and pruning removes only tuples dead to it.
- [avoid bloat from CREATE INDEX CONCURRENTLY](https://www.postgresql.org/message-id/flat/20170228052302.r7nvt4sbnsgu6job@alvherre.pgsql) (similar idea; Returned with feedback, 2017-03-07). "However, now that I read the patch: this is a horribly ugly hack." ([message](https://www.postgresql.org/message-id/1983.1488288643%40sss.pgh.pa.us)) Different now: P2 released the snapshot once after the first scan, for btree only, through an undocumented snapmgr API. This proposal resets during the scan and between tuples afterwards, through a documented AM call and an opt-in flag, for all built-in AMs.

## Objections a reviewer is likely to raise

### 1. On by default for every concurrent build, with no measurements

An operator upgrades and runs REINDEX CONCURRENTLY on a large unique index. The build now resets snapshots every 100ms, compares keys per tuple in _bt_load(), and may push many entries to validation once the 5% budget runs out. It runs longer, and the material gives no way to predict by how much or how much xmin advances.

Message #1 says no significant _bt_load slowdown was seen but shows no numbers. Message #11 calls the 0.05 default "not measure". Sorting and tuplesort merges still pin one snapshot, so the share of a long build that gains is unknown. Two new user GUCs for one command will be hard to remove later if their defaults turn out wrong.

**Question:** On a large table, how far does xmin advance, and how do build time and validation work for unique builds change with the default settings?

<sub>fixable, level 1b, performance on the hot path; messages [#1](https://www.postgresql.org/message-id/CADzfLwWAiimwngq38x8Ji26AtYEmSqnFD7aa4kVGF4ObDswHMA%40mail.gmail.com), [#11](https://www.postgresql.org/message-id/CADzfLwVHrrCNoJ0_dGctFNLo2Xm3UabZk4mG7t9Km9S8kNzAYw%40mail.gmail.com); `maintenance-fit-2`</sub>

### 2. Gain unmeasured while the validation scan still pins xmin

An operator runs REINDEX CONCURRENTLY on a large, busy table, expecting vacuum to keep up. The first scan now releases xmin, but the tuplesort phase and the full validation scan still hold one snapshot each. The operator sees the same horizon holdback for a large part of the run and the same bloat.

The demand for xmin advance during CIC is real: the project committed it once and reverted it only for safety [P1], and [P2] tried earlier. This set fixes only part of the run, and message #1 leaves the second phase to the separate STIR entry. The material has no numbers on how much of a long build's xmin holdback goes away. It does add two GUCs, an AM flag, a new C function and a reworked unique-build path, and a committer would likely ask whether that gain justifies that cost before the second phase lands.

Precedent: [Revert changes to CONCURRENTLY that "sped up" Xmin advance](https://www.postgresql.org/message-id/flat/17485-396609c6925b982d@postgresql.org) (Reverted, 2022-05-31)

Precedent: [avoid bloat from CREATE INDEX CONCURRENTLY](https://www.postgresql.org/message-id/flat/20170228052302.r7nvt4sbnsgu6job@alvherre.pgsql) (Returned with feedback, 2017-03-07)

**Question:** On a large table under write load, how much shorter is the xmin holdback across a whole CIC or REINDEX CONCURRENTLY run with this set, without STIR?

<sub>fixable, level 1b, desirability; messages [#1](https://www.postgresql.org/message-id/CADzfLwWAiimwngq38x8Ji26AtYEmSqnFD7aa4kVGF4ObDswHMA%40mail.gmail.com), [#11](https://www.postgresql.org/message-id/CADzfLwVHrrCNoJ0_dGctFNLo2Xm3UabZk4mG7t9Km9S8kNzAYw%40mail.gmail.com); `demand-1`</sub>

### 3. Safety argument for missed tuples is informal, as in the reverted PG14 feature

A busy table gets HOT updates and pruning while CREATE INDEX CONCURRENTLY rotates its snapshot. In a case the stress test does not cover, validation skips a live tuple, and the new index silently misses it. Users see wrong query results or amcheck errors months later, as in the 2022 case.

[P1] reverted a feature whose goal was the same: let xmin advance during CIC. The revert reason was that pruning removed tuples the build still relied on, and validation did not repair this. The proposal's safety claim is one informal sentence in message #1, a README.HOT edit, and a stress test that the author says in message #11 was mostly AI-written and not yet reviewed closely. v8 also adds a new path where unique builds leave entries out for validation to insert.

Precedent: [Revert changes to CONCURRENTLY that "sped up" Xmin advance](https://www.postgresql.org/message-id/flat/17485-396609c6925b982d@postgresql.org) (Reverted, 2022-05-31)

**Question:** What written invariant shows that validate_index still inserts every live tuple when the first scan and the unique-build load use rotating snapshots and pruning runs between rotations?

<sub>fixable, level 2, prior rejection; messages [#1](https://www.postgresql.org/message-id/CADzfLwWAiimwngq38x8Ji26AtYEmSqnFD7aa4kVGF4ObDswHMA%40mail.gmail.com), [#11](https://www.postgresql.org/message-id/CADzfLwVHrrCNoJ0_dGctFNLo2Xm3UabZk4mG7t9Km9S8kNzAYw%40mail.gmail.com); `prior-attempts-1`</sub>

### 4. New TOAST and snapshot rules for every index AM, enforced by nothing

A later patch changes the GIN accumulator, a BRIN opclass, or a GiST buffered-build path so that it keeps a datum across pages without detoasting it. Tests pass because a reset seldom lands between the copy and the use while vacuum removes that TOAST value. Months later a user's CREATE INDEX CONCURRENTLY fails or reads freed data.

0002-0007 add contracts that every build path must keep: detoast retained values, hold no extra snapshot, call index_build_rotate_snapshot() only after the scan is closed, and follow the parallel reset protocol. 0003 and 0004 show these rules are easy to miss, since GIN and BRIN both broke them. Only opt-in for out-of-core AMs answers the concern in message #10. In-core AMs still carry the rules, and a missed rule surfaces only as a rare race.

**Question:** Which assertion or test fails right away when a build path keeps an undetoasted datum or an extra snapshot across a reset?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#10](https://www.postgresql.org/message-id/CAEze2WhwwAB_DVAWfunO%3D_1QZu-J%2BThbpVbvdoOyycswQETFfw%40mail.gmail.com), [#11](https://www.postgresql.org/message-id/CADzfLwVHrrCNoJ0_dGctFNLo2Xm3UabZk4mG7t9Km9S8kNzAYw%40mail.gmail.com); `maintenance-fit-1`</sub>

### 5. Duplicate-check budget exposed as a GUC with an unmeasured default

An operator builds a unique index concurrently on a table where the key column is updated often. The fetch budget runs out, many equal-key entries move to the validation scan, and the build takes longer. The only lever is index_build_duplicate_check_scale_factor, whose default 0.05 the author calls not measured, and whose meaning depends on build internals.

The author says in message #11 that 0.05 is "not measure". The setting describes internal mechanics, separate sort and load budgets and a floor of 1000 fetches, which a later design change such as the ReadStream batching suggested in message #11 would make obsolete. The documentation hunk places it next to post_auth_delay, so it may be meant as a developer option, but the proposal does not say so. Once a setting ships, removing or redefining it breaks configuration files.

**Question:** Is index_build_duplicate_check_scale_factor meant for users or only for developers, and what measurement supports 0.05 as the default?

<sub>fixable, level 2, new GUC; messages [#11](https://www.postgresql.org/message-id/CADzfLwVHrrCNoJ0_dGctFNLo2Xm3UabZk4mG7t9Km9S8kNzAYw%40mail.gmail.com), [#12](https://www.postgresql.org/message-id/CADzfLwV%2Bng11CQPju9HxJvK_rLAZ0_Zozno_%2B4Td26ZL1V_%3DrA%40mail.gmail.com); `standards-compat-2`</sub>

## Questions for the author

1. What written invariant shows that validation still inserts every live tuple when the first scan and the unique-build load use rotating snapshots and pruning runs between resets?
2. On a large table under write load, how much shorter is the xmin holdback across a whole CREATE INDEX CONCURRENTLY run with this set, without STIR?
3. How much do build time and validation work change for a large unique build with the default settings, compared with master?
4. Which assertion or test fails right away when a build path keeps a TOAST pointer or an extra snapshot across a reset?
5. What measurements support the defaults of 100ms for the reset interval and 0.05 for index_build_duplicate_check_scale_factor?

## Touch points

- **ABI / extension API:** IndexAmRoutine gains amcanresetsnapshot, and AMs get index_build_rotate_snapshot() plus new rules for retained TOAST values and snapshots.
- **new GUC:** Adds concurrent_index_reset_snapshot_interval and index_build_duplicate_check_scale_factor.
- **maintenance burden ("who maintains this in 10 years"):** The series changes snapmgr.c, heapam, tuplesort, nbtsort.c and every built-in index AM, and adds rules that each build path must keep.
- **performance on the hot path:** Unique builds compare keys per tuple in _bt_load(), and 0005 touches the shared parallel table scan code.

## Who would commit it, and when

- Owner: qualified committer engaged. Álvaro Herrera, who committed and reverted the PG14 feature [P1], takes part in the thread, so far only on the stress test. The owner needs to know snapshot management, heap pruning and nbtree builds.
- Priority: proceed; audience a sizeable minority; review cost high. Operators of large, busy databases gain, and the project already accepted the goal once [P1]. The seven-patch series changes snapshot, heap, sort and index AM code, so review is costly; it needs a rebase and the measurements first.

---

[roast-full.md](roast-full.md) has 1 more objection(s) that held, 1 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 4.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

# Design pre-review: Preserve statistics targets with ALTER TABLE ALTER COLUMN TYPE

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7214](https://commitfest.postgresql.org/patch/7214/); [thread](https://www.postgresql.org/message-id/flat/CAN4CZFNZwcCgi-igaD%3DLH1ubxMBqJJS%2Bp4ZnOKKdCi9duaMu_w%40mail.gmail.com), 5 of 5 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No objection survived at levels 1a or 1b: the problem is stated, and the fix restores settings the user chose. The one surviving objection, standards-compat-1, is fixable. The entry is tagged Bugfix, both patches add fields to parse nodes, and the thread names no target branch. A committer has to decide the branch scope and which thread carries 0002.

Not checked: Without the code, the gram.y change in 0001 and the way DefineIndex() applies targets to partition indexes could not be checked. The only review in the thread tested idx_scan counters, which the patches do not claim to change.

## Prior attempts

None found before 2026-08-14.

## Objections a reviewer is likely to raise

### 1. Bug fix tag implies a backpatch, but parse node structs change

A committer backpatches both patches as a bug fix. An extension built against the earlier minor release creates an IndexStmt or CreateStatsStmt with makeNode() at the old size and passes it to DefineIndex() or CreateStatistics(). The server reads the new field past the end of the allocation, and the user sees wrong targets or a crash after a minor upgrade.

The CommitFest entry is tagged Bugfix, and the brief notes that the material does not say whether back-branches are meant. Both patches add fields to parse nodes in parsenodes.h, which back branches avoid changing. The proposal does not say whether the fix is master-only or how a back-branch version would avoid the struct change.

**Question:** Is this fix meant only for master, and if not, how would a back-branch version keep IndexStmt and CreateStatsStmt unchanged?

<sub>fixable, level 2, backpatchability; messages [#1](https://www.postgresql.org/message-id/CAN4CZFNZwcCgi-igaD%3DLH1ubxMBqJJS%2Bp4ZnOKKdCi9duaMu_w%40mail.gmail.com), [#4](https://www.postgresql.org/message-id/CAN4CZFOp2tQQ_OmjRJha%3D%2B2BuFBoCrm4QPFJFpoa0aRmL6y0Mw%40mail.gmail.com); `standards-compat-1`</sub>

## Questions for the author

1. Is this fix meant only for master, and if not, how would a back-branch version keep IndexStmt and CreateStatsStmt unchanged?
2. Why does 0001 change gram.y and parse_utilcmd.c, and does create statistics gain any new user-visible syntax?
3. Which of the two patches does the other series now carry, and should this entry keep only the patch that the other series does not include?
4. Does a target set directly on a partition's index still revert without notice when the rebuild recurses from the parent, and is that acceptable for a fix of silent loss?

## Touch points

- **ABI / extension API:** IndexStmt and CreateStatsStmt each gain a field in parsenodes.h, and extensions build these nodes.
- **SQL-standard conflict or non-standard syntax:** Patch 0001 touches gram.y, and the commit message does not say whether new syntax appears.
- **planner behaviour change:** Kept statistics targets change what analyze collects after the rebuild, so plans can differ from today's post-rebuild plans.
- **backpatchability:** The entry is tagged Bugfix, but both patches change parse node structs that back branches normally keep stable.
- **maintenance burden ("who maintains this in 10 years"):** The rebuild path in ATPostAlterTypeParse() gains two more properties to capture and reapply.

## Who would commit it, and when

- Owner: silent. Needs a committer who knows the alter table rebuild path in tablecmds.c and the extended statistics code. Melanie Plageman took a modified 0002 into another series, but nobody in the thread has reviewed the approach of 0001.
- Priority: proceed; audience niche; review cost low. Only users who set statistics targets on expression indexes or extended statistics are affected, but the loss is silent. Both patches are about 12 KB each, stay in the rebuild path, and follow the pattern already used for the comment and the owner.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 1 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

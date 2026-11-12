# Design pre-review: pg_upgrade fails when FK constraint with same name exists on partitioned table and its partition

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7062](https://commitfest.postgresql.org/patch/7062/); [thread](https://www.postgresql.org/message-id/flat/bcbd40ec-9914-4a12-b750-e997dca0442e%40postgrespro.ru), 5 of 5 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The failure is real and reproducible, but the design is not settled. The check also fails for clusters that would upgrade fine (standards-compat-1), and the thread has not weighed a warning, a narrow query, or a pg_dump ordering fix (maintenance-fit-2). Demand rests on one report, and the one committer in the thread doubted the check is worth adding (demand-1). The release scope is also open (standards-compat-2).

Not checked: Without the code, the query's handling of multi-level partition hierarchies and of the 11-16 catalogs could not be judged. The false-positive counts come from the reviewer's report in message #5 only.

## Prior attempts

- [pg_upgrade fails with non-standard ACL](https://www.postgresql.org/message-id/flat/f85991ad-bbd4-ad57-fde4-e12f0661dbf0@postgrespro.ru) (similar idea; Returned with feedback, 2021-11-26). "I'm not generally enthusiastic about adding pg_upgrade code to predict whether the dump will fail to restore, because such code will never be as good as just trying the restore." ([message](https://www.postgresql.org/message-id/20210127112135.GB2481576%40rfd.leadboat.com)) Different now: [P14] predicted failing GRANT and REVOKE commands on changed catalog objects and wrote a fix script. This proposal predicts a duplicate FK name conflict and gives only a hint.

## Objections a reviewer is likely to raise

### 1. Broad check makes --check fail for clusters that upgrade fine

A DBA's upgrade pipeline runs pg_upgrade --check as a gate. A cluster whose partition sorts before its parent, which would restore cleanly, now fails the gate. The DBA must rename production constraints before upgrading, or skip the check, to get past a failure that would never happen.

Message #5 reports 5 false positives against 5 real failures in 13 cases, so half of the reports block upgrades that work today. Today a --check failure means the upgrade will fail. The broad query turns some failures into advice, and the proposal still makes them fatal. The reviewer prefers the broad query so that a future pg_dump sort change cannot hide cases, but that trade costs users now for a risk that a test could catch. [P14] doubts that pg_upgrade code can predict restore failures as well as a real restore does.

Precedent: [pg_upgrade fails with non-standard ACL](https://www.postgresql.org/message-id/flat/f85991ad-bbd4-ad57-fde4-e12f0661dbf0@postgrespro.ru) (Returned with feedback, 2021-11-26)

**Question:** Why should cases that restore cleanly fail --check, instead of producing a warning or being excluded by the narrow query plus a test that pins pg_dump's sort order?

<sub>fixable, level 2, design; messages [#2](https://www.postgresql.org/message-id/aihVOD_Q23zuHBE1%40alvherre.pgsql), [#4](https://www.postgresql.org/message-id/CADMs5OO%3DCBQYn3S-PZqk5fLfn2kYmnBKfvKb8aDUm_xXxcFJ0g%40mail.gmail.com), [#5](https://www.postgresql.org/message-id/CADMs5OOPeCcJ8WxTA-Eu8untmULq0yd%3DYQW-i_j2xzf8b1s94w%40mail.gmail.com); `standards-compat-1`</sub>

### 2. The check covers pg_upgrade only, but pg_dump ordering causes the failure

A user runs pg_dump on a 17 database with this layout and restores it into 18 with pg_restore. No check runs on that path, so the restore fails with the same "already exists" error. The user sees no hint.

Message #5 shows the restore works when the partition's own ADD CONSTRAINT comes before the ancestor's, because the clone then gets a generated name. A dump-order dependency in the new pg_dump could make that order hold for both pg_upgrade and plain dump/restore. Message #2 declines changes to pg_dump, but gives no technical reason. A committer would likely ask whether an ordering rule in pg_dump costs less than a permanent check that only helps one tool.

**Question:** Why is an ordering dependency in pg_dump, which would also fix plain dump/restore, worse than a check confined to pg_upgrade?

<sub>fixable, level 2, design; messages [#2](https://www.postgresql.org/message-id/aihVOD_Q23zuHBE1%40alvherre.pgsql), [#5](https://www.postgresql.org/message-id/CADMs5OOPeCcJ8WxTA-Eu8untmULq0yd%3DYQW-i_j2xzf8b1s94w%40mail.gmail.com); `maintenance-fit-2`</sub>

### 3. Master-only check misses most users who hit this upgrade

A user upgrades from 17 to 18 or 19 with that release's pg_upgrade. --check still reports "Clusters are compatible" and the restore fails as in message #1. The new check never runs for them because it exists only in the PG20 pg_upgrade binary.

The check runs in the new cluster's pg_upgrade, and the affected source versions are 11 through 17. The thread targets master only (message #3), and the CommitFest entry is in the PG20 cycle. Users who upgrade from 17 to 18 or 19 in the next few years get no protection. A committer would likely ask whether a fix tagged Bugfix should be back-patched to the pg_upgrade of 18 and 19. The thread does not discuss this.

**Question:** Is the check intended for back-patch to the pg_upgrade of 18 and 19, and if not, how do users upgrading from 17 into those releases learn about this case?

<sub>fixable, level 2, backpatchability; messages [#1](https://www.postgresql.org/message-id/bcbd40ec-9914-4a12-b750-e997dca0442e%40postgrespro.ru), [#3](https://www.postgresql.org/message-id/d2358b70-67c5-4590-a498-c2a1a0afb355%40postgrespro.ru); `standards-compat-2`</sub>

### 4. One report of an unusual schema; no wider demand shown

The check ships and runs on every pg_upgrade from 11-17 for years, but almost no cluster has a redundant NOT VALID FK on a partition with the parent's FK name. A user who does hit the case today gets a clear pg_restore error, renames the constraint, and reruns pg_upgrade with the old cluster intact.

The thread shows one reproducer from one reporter (message #1), with no field reports, -general or -bugs threads, or second user. Álvaro says the partition constraint has little use, and he is not convinced a check is worth it (message #2). The author agrees the fix is a drop or rename (message #3). The current failure happens before data transfer and names the constraint, so the cost of the workaround is one retry.

**Question:** How many users besides the original reporter have hit this restore failure, and did anyone find it hard to recover from?

<sub>fixable, level 1b, demand; messages [#1](https://www.postgresql.org/message-id/bcbd40ec-9914-4a12-b750-e997dca0442e%40postgrespro.ru), [#2](https://www.postgresql.org/message-id/aihVOD_Q23zuHBE1%40alvherre.pgsql), [#3](https://www.postgresql.org/message-id/d2358b70-67c5-4590-a498-c2a1a0afb355%40postgrespro.ru); `demand-1`</sub>

## Questions for the author

1. Why should cases that restore cleanly fail --check, instead of producing a warning or being excluded by a narrow query with a test that pins pg_dump's sort order?
2. Why is an ordering dependency in pg_dump, which would also fix plain dump and restore, worse than a check confined to pg_upgrade?
3. How many users besides the original reporter have hit this restore failure, and did anyone find it hard to recover from?
4. Is the check intended for back-patch to the pg_upgrade of 18 and 19, and if not, how do users upgrading from 17 learn about this case?
5. Does the author accept the v2-0002 changes to the report and the hint?

## Touch points

- **on-disk format / pg_upgrade:** The patch adds a new fatal check to pg_upgrade --check in src/bin/pg_upgrade/check.c.
- **pg_dump / restore round-trip:** The failure comes from the order of pg_dump's ADD CONSTRAINT statements; the proposal leaves that order unchanged.
- **backpatchability:** The entry is tagged Bugfix but targets master only, while affected upgrades from 17 go into 18 and 19.
- **maintenance burden ("who maintains this in 10 years"):** The check and its version-gated TAP test stay as long as pg_upgrade accepts old clusters from 11 through 17.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** The check predicts a restore failure that the real run already reports, an approach [P14] records doubts about.

## Who would commit it, and when

- Owner: qualified committer engaged. Álvaro Herrera replied in message #2 but was not convinced, and has not replied to the patch or to v2. The owner needs to know pg_upgrade checks and partition FK handling.
- Priority: fix design first; audience niche; review cost low. The patch touches one file and affects only clusters with a same-named NOT VALID FK on a partition and its parent. The thread must first settle whether cases that would upgrade fine should fail the check.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 4 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

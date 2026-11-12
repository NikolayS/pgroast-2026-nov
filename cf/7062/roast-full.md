# Design pre-review: pg_upgrade fails when FK constraint with same name exists on partitioned table and its partition

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7062](https://commitfest.postgresql.org/patch/7062/) (Needs review); authors: Arseny Kositsin (arseny114)
- Thread: [pg_upgrade fails when FK constraint with same name exists on partitioned table and its partition](https://www.postgresql.org/message-id/flat/bcbd40ec-9914-4a12-b750-e997dca0442e%40postgrespro.ru), started 2026-06-08
- Messages read: 5 of 5
- Patch prose read: v1-pg_upgrade-check-for-duplicate-FK-names-on-partition.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The failure is real and reproducible, but the design is not settled. The check also fails for clusters that would upgrade fine (standards-compat-1), and the thread has not weighed a warning, a narrow query, or a pg_dump ordering fix (maintenance-fit-2). Demand rests on one report, and the one committer in the thread doubted the check is worth adding (demand-1). The release scope is also open (standards-compat-2).

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, the query's handling of multi-level partition hierarchies and of the 11-16 catalogs could not be judged. The false-positive counts come from the reviewer's report in message #5 only.

## The proposal, restated

**Problem.** pg_upgrade from Postgres 11 through 17 can fail in pg_restore when a partition has its own NOT VALID foreign key with the same name as an FK on its parent. pg_upgrade --check reports "Clusters are compatible" and the real run then fails with "constraint ... for relation ... already exists". This affects users who upgrade clusters with such partitions.

**Proposed design.** pg_upgrade --check gets a new check that queries the old cluster for NOT VALID FKs on partitions that share a name with an FK on the parent and have conparentid = 0. The check runs only for old clusters from version 11 through 17. pg_dump output is unchanged. v2 adds quoted, schema-qualified names in the report, "may fail" wording, a hint that prefers rename, and a TAP test.

**User-visible surface.**
- New pg_upgrade --check failure that lists affected partition FKs as schema.table.constraint, quoted in v2.
- New hint text: rename the partition's constraint, or drop it when it matches the parent's FK (v2 wording).
- New TAP test src/bin/pg_upgrade/t/010_duplicate_fk_names.pl, which runs only when oldinstall points to a version 11-17 install.

**What the author claims.**
- Only NOT VALID FKs are affected, because before commit b663b9436e7 ATTACH PARTITION did not link them to the parent's FK.
- CHECK, PRIMARY KEY, UNIQUE, EXCLUDE and NOT NULL constraints cannot cause this conflict.
- Whether the restore fails depends on pg_dump's sort order, and the check flags both orders on purpose.
- The reviewer reports that v1 catches all 5 failing cases out of 13 tested, and also flags 5 cases that would have upgraded fine.
- The reviewer argues a narrow query would copy pg_dump's internal sort order and could silently miss cases if that order changes.

**Thread state.** Álvaro Herrera doubted a --check was worth adding and declined to change pg_dump or pg_upgrade restore behaviour. João Rito reviewed v1, posted v2-0002 (report and hint) and v2-0003 (test), but neither the author nor a committer has replied to v2.

## 2. Prior attempts

### pg_upgrade fails with non-standard ACL (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/2215/](https://commitfest.postgresql.org/patch/2215/)
- **Thread:** [https://www.postgresql.org/message-id/flat/f85991ad-bbd4-ad57-fde4-e12f0661dbf0@postgrespro.ru](https://www.postgresql.org/message-id/flat/f85991ad-bbd4-ad57-fde4-e12f0661dbf0@postgrespro.ru)
- **Date:** decided 2021-11-26; started 2019-07-18
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Noah Misch (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** The author stopped responding after Noah Misch's March 2021 review, so the entry sat in Waiting on Author until Daniel Gustafsson returned it in November 2021. That review showed that the generated script still let the final pg_upgrade fail on a REVOKE ... FROM PUBLIC, and that the check compared obj_ident without obj_type. Noah Misch also had doubts about the approach, since pg_upgrade code that predicts restore failures cannot match a real restore.
- **Reason (quote):** "I'm not generally enthusiastic about adding pg_upgrade code to predict whether the dump will fail to restore, because such code will never be as good as just trying the restore." ([message](https://www.postgresql.org/message-id/20210127112135.GB2481576%40rfd.leadboat.com))
- **What is different this time:** [P14] predicted failing GRANT and REVOKE commands on changed catalog objects and wrote a fix script. This proposal predicts a duplicate FK name conflict and gives only a hint.


## 3. Objections a reviewer is likely to raise

### 1. Broad check makes --check fail for clusters that upgrade fine

fixable · level 2, rubric item 10 · design, maintenance burden ("who maintains this in 10 years") · `standards-compat-1`

A DBA's upgrade pipeline runs pg_upgrade --check as a gate. A cluster whose partition sorts before its parent, which would restore cleanly, now fails the gate. The DBA must rename production constraints before upgrading, or skip the check, to get past a failure that would never happen.

Message #5 reports 5 false positives against 5 real failures in 13 cases, so half of the reports block upgrades that work today. Today a --check failure means the upgrade will fail. The broad query turns some failures into advice, and the proposal still makes them fatal. The reviewer prefers the broad query so that a future pg_dump sort change cannot hide cases, but that trade costs users now for a risk that a test could catch. [P14] doubts that pg_upgrade code can predict restore failures as well as a real restore does.

**Precedents.**
- [pg_upgrade fails with non-standard ACL](https://commitfest.postgresql.org/patch/2215/) (Returned with feedback, 2021-11-26); thread: [link](https://www.postgresql.org/message-id/flat/f85991ad-bbd4-ad57-fde4-e12f0661dbf0@postgrespro.ru)

Thread: [#2 (Álvaro Herrera, 2026-06-09)](https://www.postgresql.org/message-id/aihVOD_Q23zuHBE1%40alvherre.pgsql), [#4 (João Rito, 2026-11-10)](https://www.postgresql.org/message-id/CADMs5OO%3DCBQYn3S-PZqk5fLfn2kYmnBKfvKb8aDUm_xXxcFJ0g%40mail.gmail.com), [#5 (João Rito, 2026-11-10)](https://www.postgresql.org/message-id/CADMs5OOPeCcJ8WxTA-Eu8untmULq0yd%3DYQW-i_j2xzf8b1s94w%40mail.gmail.com)

**Question:** Why should cases that restore cleanly fail --check, instead of producing a warning or being excluded by the narrow query plus a test that pins pg_dump's sort order?

Counter-check: held. Message #5 confirms that v1 reports 5 cases that upgrade fine, and the check still fails for them. The reviewer accepted that cost, but no committer has, and the thread does not discuss a warning or a narrow query with a test.

### 2. The check covers pg_upgrade only, but pg_dump ordering causes the failure

fixable · level 2, rubric item 12 · design, pg_dump / restore round-trip, doesn't fit what Postgres is about (extension / external tool / already solved) · `maintenance-fit-2`

A user runs pg_dump on a 17 database with this layout and restores it into 18 with pg_restore. No check runs on that path, so the restore fails with the same "already exists" error. The user sees no hint.

Message #5 shows the restore works when the partition's own ADD CONSTRAINT comes before the ancestor's, because the clone then gets a generated name. A dump-order dependency in the new pg_dump could make that order hold for both pg_upgrade and plain dump/restore. Message #2 declines changes to pg_dump, but gives no technical reason. A committer would likely ask whether an ordering rule in pg_dump costs less than a permanent check that only helps one tool.

Thread: [#2 (Álvaro Herrera, 2026-06-09)](https://www.postgresql.org/message-id/aihVOD_Q23zuHBE1%40alvherre.pgsql), [#5 (João Rito, 2026-11-10)](https://www.postgresql.org/message-id/CADMs5OOPeCcJ8WxTA-Eu8untmULq0yd%3DYQW-i_j2xzf8b1s94w%40mail.gmail.com)

**Question:** Why is an ordering dependency in pg_dump, which would also fix plain dump/restore, worse than a check confined to pg_upgrade?

Counter-check: held. Message #2 declines a pg_dump change without a technical reason, and message #4 agrees on scope only. The plain-restore scenario is weaker than stated, since pg_restore continues after the error by default, but the design question stands.

### 3. Master-only check misses most users who hit this upgrade

fixable · level 2, rubric item 6 · backpatchability · `standards-compat-2`

A user upgrades from 17 to 18 or 19 with that release's pg_upgrade. --check still reports "Clusters are compatible" and the restore fails as in message #1. The new check never runs for them because it exists only in the PG20 pg_upgrade binary.

The check runs in the new cluster's pg_upgrade, and the affected source versions are 11 through 17. The thread targets master only (message #3), and the CommitFest entry is in the PG20 cycle. Users who upgrade from 17 to 18 or 19 in the next few years get no protection. A committer would likely ask whether a fix tagged Bugfix should be back-patched to the pg_upgrade of 18 and 19. The thread does not discuss this.

Thread: [#1 (Arseny Kositsin, 2026-06-08)](https://www.postgresql.org/message-id/bcbd40ec-9914-4a12-b750-e997dca0442e%40postgrespro.ru), [#3 (Arseny Kositsin, 2026-07-23)](https://www.postgresql.org/message-id/d2358b70-67c5-4590-a498-c2a1a0afb355%40postgrespro.ru)

**Question:** Is the check intended for back-patch to the pg_upgrade of 18 and 19, and if not, how do users upgrading from 17 into those releases learn about this case?

Counter-check: held. The check runs in the new version's pg_upgrade, and message #3 targets master only. Upgrades from 17 into 18 or 19 keep the failure, and the thread does not discuss a back-patch.

### 4. One report of an unusual schema; no wider demand shown

fixable · level 1b, rubric item 3 · demand, doesn't fit what Postgres is about (extension / external tool / already solved), maintenance burden ("who maintains this in 10 years") · `demand-1`

The check ships and runs on every pg_upgrade from 11-17 for years, but almost no cluster has a redundant NOT VALID FK on a partition with the parent's FK name. A user who does hit the case today gets a clear pg_restore error, renames the constraint, and reruns pg_upgrade with the old cluster intact.

The thread shows one reproducer from one reporter (message #1), with no field reports, -general or -bugs threads, or second user. Álvaro says the partition constraint has little use, and he is not convinced a check is worth it (message #2). The author agrees the fix is a drop or rename (message #3). The current failure happens before data transfer and names the constraint, so the cost of the workaround is one retry.

Thread: [#1 (Arseny Kositsin, 2026-06-08)](https://www.postgresql.org/message-id/bcbd40ec-9914-4a12-b750-e997dca0442e%40postgrespro.ru), [#2 (Álvaro Herrera, 2026-06-09)](https://www.postgresql.org/message-id/aihVOD_Q23zuHBE1%40alvherre.pgsql), [#3 (Arseny Kositsin, 2026-07-23)](https://www.postgresql.org/message-id/d2358b70-67c5-4590-a498-c2a1a0afb355%40postgrespro.ru)

**Question:** How many users besides the original reporter have hit this restore failure, and did anyone find it hard to recover from?

Counter-check: held. The thread shows one reporter and one reproducer, and Álvaro in message #2 doubts the check is worth adding. The author's point that the error is not obvious does not show that the case is common.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | The patch adds a new fatal check to pg_upgrade --check in src/bin/pg_upgrade/check.c. |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | **yes** | The failure comes from the order of pg_dump's ADD CONSTRAINT statements; the proposal leaves that order unchanged. |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The entry is tagged Bugfix but targets master only, while affected upgrades from 17 go into 18 and 19. |
| maintenance burden ("who maintains this in 10 years") | **yes** | The check and its version-gated TAP test stay as long as pg_upgrade accepts old clusters from 11 through 17. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | The check predicts a restore failure that the real run already reports, an approach [P14] records doubts about. |

## 5. Questions the author must answer before anyone reviews code

1. Why should cases that restore cleanly fail --check, instead of producing a warning or being excluded by a narrow query with a test that pins pg_dump's sort order?
2. Why is an ordering dependency in pg_dump, which would also fix plain dump and restore, worse than a check confined to pg_upgrade?
3. How many users besides the original reporter have hit this restore failure, and did anyone find it hard to recover from?
4. Is the check intended for back-patch to the pg_upgrade of 18 and 19, and if not, how do users upgrading from 17 learn about this case?
5. Does the author accept the v2-0002 changes to the report and the hint?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Álvaro Herrera replied in message #2 but was not convinced, and has not replied to the patch or to v2. The owner needs to know pg_upgrade checks and partition FK handling.
- **Liability:** A fatal pg_upgrade check with known false positives, kept while old clusters from 11 through 17 are supported, plus a TAP test that runs only with an old install.
- **Author follow-up:** The author returned after six weeks with a patch and offered to add a test, but has not replied to v2.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch touches one file and affects only clusters with a same-named NOT VALID FK on a partition and its parent. The thread must first settle whether cases that would upgrade fine should fail the check.

## Appendix

### Refuted in the counter-check

- `prior-attempts-1` Predictive restore check blocks upgrades that would succeed. Refuted (not specific): Duplicate of standards-compat-1, which states the same false-positive problem and names the two alternatives more exactly.
- `maintenance-fit-1` Broad prediction check blocks upgrades that would succeed. Refuted (not specific): Duplicate of standards-compat-1. The added maintenance point is weak, because the broad query does not depend on pg_dump's sort order and applies only to 11-17 old clusters.
- `already-exists-1` The real pg_upgrade run already detects this, with fewer wrong answers. Refuted (not specific): Duplicate of demand-1 and standards-compat-1. Its own point is weak: the real run needs the old server stopped, so it cannot give the advance notice that --check gives on a running cluster.
- `demand-2` Per-pattern restore-failure checks do not pass the accumulation test. Refuted (no failure scenario): The scenario depends on future checks that the material does not show. check.c already holds many version-gated checks, this one costs only users with 11-17 old clusters, and the remaining demand point duplicates demand-1.

### Review angles

- prior-attempts: 1 objection(s). [P8] is about Unicode version changes, and its recorded reason does not apply here.
- standards-compat: 2 objection(s). The proposal leaves pg_dump output, SQL syntax, GUCs and psql unchanged. The two objections concern the --check contract and the release scope.
- replication-ondisk: 0 objection(s). The patch adds a read-only catalog query to pg_upgrade --check on the old cluster. It does not change on-disk format, WAL, the catalogs, the wire protocol or any extension-facing API, and pg_dump output stays the same. Whether a restore-predicting check is worth its false positives ([P14]) is a design question for other angles.
- security: 0 objection(s). The proposal has no privilege or exposure aspect. The new check is a read-only catalog query that pg_upgrade already runs as the install user against the old cluster. It adds no SQL surface, role, GUC or default, and v2 quotes the reported names. Not checked without the code: whether the report file is written with the same permissions as other pg_upgrade report files.
- maintenance-fit: 2 objection(s). Hot-path cost is not a concern here, because the check runs only during pg_upgrade against 11-17 old clusters.
- already-exists: 1 objection(s). One objection only. The material names no extension or other core feature that already detects this conflict.
- demand: 2 objection(s). Demand is the weak point: one reporter, and a committer who doubts the check is worth adding.

### Precedents retrieved

- P1: [pg_upgrade: Support for upgrading to checksums enabled](https://commitfest.postgresql.org/patch/5206/) (cf, Returned with feedback, 2025-04-03) - judged not relevant; reason summarised from its thread
- P2: [Reducing memory usage of creating foreign key between partitioned tables](https://commitfest.postgresql.org/patch/2330/) (cf, Rejected, 2020-09-12) - judged not relevant; reason summarised from its thread
- P3: [Revert structural changes to not-null constraints](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6f8bb7c1e9610dd7af20cdaf74c4ff6e6d678d44) (revert, Reverted, 2024-05-13) - judged not relevant
- P4: [pg_upgrade version and path checking](https://commitfest.postgresql.org/patch/2095/) (cf, Returned with feedback, 2019-09-30) - judged not relevant; reason summarised from its thread
- P5: [avoid bitmapOR-ing indexes for scan conditions implied by partition constraint](https://commitfest.postgresql.org/patch/2644/) (cf, Returned with feedback, 2021-02-01) - judged not relevant; reason summarised from its thread
- P6: [pg_upgrade data type check connection overhead reduction](https://commitfest.postgresql.org/patch/4200/) (cf, Returned with feedback, 2024-02-01) - judged not relevant; reason summarised from its thread
- P7: [Revert "Make pg_dump emit ATTACH PARTITION instead of PARTITION OF"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a1ec7402e9a0392f95bc79542af0efcd5c6e7929) (revert, Reverted, 2019-05-06) - judged not relevant
- P8: [pg_upgrade check for Unicode update](https://commitfest.postgresql.org/patch/5563/) (cf, Withdrawn, 2025-03-18) - used in the review; reason summarised from its thread
- P9: [pg_upgrade documentation improvement](https://commitfest.postgresql.org/patch/583/) (cf, Returned with feedback, 2016-10-03) - judged not relevant; reason summarised from its thread
- P10: [Foreign Key Arrays](https://commitfest.postgresql.org/patch/2966/) (cf, Returned with feedback, 2021-09-14) - judged not relevant
- P11: [Foreign Key Arrays](https://commitfest.postgresql.org/patch/1252/) (cf, Returned with feedback, 2018-10-02) - judged not relevant
- P12: [Revert "Add basic TAP test setup for pg_upgrade"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=58ffe141eb37c3f027acd25c1fc6b36513bf9380) (revert, Reverted, 2017-09-22) - judged not relevant
- P13: [Revert "Use Foreign Key relationships to infer multi-column join selectivity".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=77ba610805e7ef9ba9c9a593ea8b1ca8f98f8bcb) (revert, Reverted, 2016-06-07) - judged not relevant
- P14: [pg_upgrade fails with non-standard ACL](https://commitfest.postgresql.org/patch/2215/) (cf, Returned with feedback, 2021-11-26) - used in the review; reason summarised from its thread
- P15: [Tab completion for ATTACH PARTITION](https://commitfest.postgresql.org/patch/4561/) (cf, Returned with feedback, 2024-01-29) - judged not relevant
- P16: [pg_dump: optimize dumpFunc()](https://commitfest.postgresql.org/patch/5147/) (cf, Withdrawn, 2024-08-02) - judged not relevant
- P17: [pg_upgrade: avoid failing if tablespace dirs exist and are empty](https://commitfest.postgresql.org/patch/2789/) (cf, Rejected, 2020-10-27) - judged not relevant
- P18: [Revert temporal primary keys and foreign keys](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=8aee330af55d8a759b2b73f5a771d9d34a7b887f) (revert, Reverted, 2024-05-16) - judged not relevant
- P19: [improve large object handling in pg_upgrade](https://commitfest.postgresql.org/patch/3841/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P20: [Revert "Allow ON CONFLICT .. DO NOTHING on a partitioned table."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f05230752d53c4aa74cffa9b699983bbb6bcb118) (revert, Reverted, 2017-03-31) - judged not relevant
- P21: [Allow logical replication via inheritance root table](https://commitfest.postgresql.org/patch/4225/) (cf, Returned with feedback, 2023-08-31) - judged not relevant
- P22: [Autovacuum (analyze) on partitioned tables for ATTACH/DETACH/DROP commands](https://commitfest.postgresql.org/patch/3188/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P23: [table inheritance versus column compression and storage settings](https://commitfest.postgresql.org/patch/4813/) (cf, Returned with feedback, 2024-03-21) - judged not relevant
- P24: [Improve pg_dump dumping publication tables](https://commitfest.postgresql.org/patch/2728/) (cf, Withdrawn, 2021-01-15) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 14; wall time: 142 s; cost reported by the CLI: $1.69
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 88 text field(s) checked, 1 finding(s), 1 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:17:13.670Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

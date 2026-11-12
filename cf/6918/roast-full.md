# Design pre-review: Global temporary tables

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6918](https://commitfest.postgresql.org/patch/6918/) (Needs review); authors: Dean Rasheed (deanr)
- Thread: [Global temporary tables](https://www.postgresql.org/message-id/flat/CAEZATCUbTj5MraZsimV8VYd1M3zrhP8soVJRZ%2BWvYyea6moefg%40mail.gmail.com), started 2026-06-21
- Messages read: 47 of 47
- Patch prose read: v12-0001-Save-temporary-table-ON-COMMIT-actions-to-pg_cla.patch, v12-0002-Basic-support-for-global-temporary-tables.patch, v12-0003-Add-support-for-indexes-on-global-temporary-tabl.patch, v12-0004-Add-support-for-global-temporary-sequences.patch, v12-0005-Support-relation-statistics-on-global-temporary-.patch, v12-0006-Support-local-vacuuming-of-global-temporary-tabl.patch, v12-0007-Allow-catalog-tables-to-be-global-temporary-and-.patch, v12-0008-Add-pg_temp_statistic_ext_data-global-temporary-.patch (commit message, file list and doc hunks only; no code review)
- Subsystem invariants checked: invariants/nbtree.md

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

Demand is shown and the problem is stated, so levels 1a and 1b pass. Level 2 has open design points the list must settle before code review. Message #1 says an idle session can hold back datfrozenxid with no remedy (replication-ondisk-1). No one has written the comparison with the competing series that message #25 asked for (maintenance-fit-3). Nothing fails loudly when code bypasses the session overlay (prior-attempts-1). All are fixable.

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, this review could not check whether DISCARD ALL covers global temporary data, when usage entries are registered relative to privilege checks, how pg_upgrade handles these relations, or what the command-start invalidation check costs.

## The proposal, restated

**Problem.** Postgres has no table whose definition is permanent and shared while its data is private to each session. Reviewers named two groups who want this: users moving from other systems, and workloads that use temporary tables heavily and fragment the catalogs.

**Proposed design.** `create global temp table` adds a relpersistence 'g' table that gets local storage and local buffers the first time a session uses it. A shared hash table tracks use across backends and blocks a rewrite or scan of a table that another session is using. Per-session filenode, statistics, frozen XIDs and index valid/ready flags live in backend memory, column and extended statistics go into new global temporary catalogs, and each PGPROC publishes tempfrozenxid so that datfrozenxid can advance.

**User-visible surface.**
- Syntax `CREATE GLOBAL TEMP[ORARY] TABLE` and `CREATE GLOBAL TEMP SEQUENCE`; `GLOBAL` no longer ignored
- New pg_class columns: reloncommit, and relpersistence value 'g'; psql \d shows the ON COMMIT action
- Functions pg_gtr_info(), pg_gtrs_in_use(), pg_gtr_index_info()
- New catalogs pg_temp_statistic and pg_temp_statistic_ext_data; pg_stats, pg_stats_ext, pg_stats_ext_exprs include them
- pg_stat_activity columns tempfrozenxid and tempminmxid; new `DISCARD GLOBAL TEMP`; ambuildempty() gains a fork number argument

**What the author claims.**
- The design avoids the problems of an earlier, older global temporary table patch.
- Each session runs VACUUM now and then, so datfrozenxid can still advance.
- Keeping session state in memory removed fragile flushing code and cut the series from 11 to 9 patches.
- Indexes that are not ready in a session close the partial uniqueness problem without blocking CREATE INDEX in other sessions.
- Saving ON COMMIT to pg_class is useful and could go in on its own.

**Thread state.** Many reviewers tested the series and found bugs, which the author fixed through v12; Haibo Yan then posted fixes for two ON CONFLICT bugs and raised an open policy question on ADD CONSTRAINT ... USING INDEX. Andrew Dunstan has a competing patch series, and no one has written the comparison of the two designs that John Naylor asked for.

## 2. Prior attempts

### Global temporary table (same idea)

- **Link:** [https://commitfest.postgresql.org/patch/2349/](https://commitfest.postgresql.org/patch/2349/)
- **Thread:** [https://www.postgresql.org/message-id/flat/1A1A6EDC-D0EC-47B0-BD21-C2ACBAEA65E4@alibaba-inc.com](https://www.postgresql.org/message-id/flat/1A1A6EDC-D0EC-47B0-BD21-C2ACBAEA65E4@alibaba-inc.com), [https://www.postgresql.org/message-id/flat/0566FEB0-74C2-4C84-BEFC-62DBD5E658D3@alibaba-inc.com](https://www.postgresql.org/message-id/flat/0566FEB0-74C2-4C84-BEFC-62DBD5E658D3@alibaba-inc.com)
- **Date:** decided 2022-06-30; started 2019-10-11
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Andres Freund, Robert Haas (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Andres Freund held that the patch could not be merged with anything close to its current design, calling it unmaintainable, and said fundamental design work was needed first, possibly through a generic way to split catalog tables into local and non-local storage. Robert Haas found the patch too big and unpolished to review: one large code drop with weak comments and code quality, where reviewers kept finding crash bugs. He asked that it be split into independently committable patches and said reaching a committable state may require a near-total rewrite. Given these two opinions that it could not be reviewed effectively as it stood, the entry was marked Returned with Feedback.
- **Reason (quote):** "I think there's just no way that it can be merged with anything close to the current design - it's unmaintainable. The need for the feature doesn't change that." ([message](https://www.postgresql.org/message-id/20220302211759.k3y2bcfchkukjt7y%40alap3.anarazel.de))
- **What is different this time:** This series is split into nine separately reviewable patches, keeps per-session relation and index state in memory, and allows rewrites such as REPACK and VACUUM FULL. The recorded objection was to a different design, so it applies only loosely.

### global temporary table v37 (same idea)

- **Link:** [https://commitfest.postgresql.org/patch/2845/](https://commitfest.postgresql.org/patch/2845/)
- **Thread:** [https://www.postgresql.org/message-id/flat/1A1A6EDC-D0EC-47B0-BD21-C2ACBAEA65E4@alibaba-inc.com](https://www.postgresql.org/message-id/flat/1A1A6EDC-D0EC-47B0-BD21-C2ACBAEA65E4@alibaba-inc.com)
- **Date:** decided 2020-11-25; started 2019-10-11
- **Outcome:** Withdrawn (cf)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (AI summary of the linked thread; check it there):** The visible messages do not show why. The entry was marked Withdrawn on the same day the author posted a rebased version fixing the whitespace problems and the relcache reference leak that Pavel Stehule had reported, and the thread shows review still ongoing rather than an objection to the design.
- **Reason (quote):** quote unavailable
- **What is different this time:** That entry was withdrawn with no recorded reason, so it does not show what this proposal would need to fix.

### Global temporary tables (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/2233/](https://commitfest.postgresql.org/patch/2233/)
- **Thread:** [https://www.postgresql.org/message-id/flat/73954ab7-44d3-b37b-81a3-69bdcbb446f7@postgrespro.ru](https://www.postgresql.org/message-id/flat/73954ab7-44d3-b37b-81a3-69bdcbb446f7@postgrespro.ru)
- **Date:** decided 2020-02-01; started 2019-07-31
- **Outcome:** Returned with feedback (cf)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (AI summary of the linked thread; check it there):** The patch was waiting on author for the whole commitfest, probably because it no longer applied, and there was little activity, so the commitfest manager marked it returned with feedback. He said the feature was not considered undesirable and suggested combining efforts with a separate global temporary tables thread where most of the discussion was happening. Earlier design objections, such as Craig Ringer's to using shared buffers for temporary data, are visible but are not given as the reason for closing.
- **Reason (quote):** "As there has been very little activity since then, I've marked it as returned with feedback." ([message](https://www.postgresql.org/message-id/20200201114954.mpmpfmdaxsnnj3ux%40development))
- **What is different this time:** That attempt closed for inactivity. This proposal uses local buffers and has active review and steady new versions, so the recorded reason does not apply.


## 3. Objections a reviewer is likely to raise

### 1. Idle sessions that used a global temporary table can block wraparound protection

fixable · level 2, rubric item 7 · WAL / replication / standby, catalog changes, maintenance burden ("who maintains this in 10 years") · `replication-ondisk-1`

A connection pooler keeps 200 backends open for weeks, and each touched a global temporary table once. Their tempfrozenxid values stop datfrozenxid and clog truncation from advancing. Autovacuum cannot fix this, so the DBA learns about it from wraparound warnings and must find and terminate or DISCARD each backend.

Message #1 and the 0006 commit message state that datfrozenxid advances only if each session runs VACUUM from time to time, and autovacuum is excluded. Local temporary tables have the same weakness, but with a global temporary table any application session that reads a shared definition can pin the horizon. Message #37 item 10 also reports that a database-wide VACUUM opens every global temporary table. Exposing tempfrozenxid in pg_stat_activity (message #28) helps diagnosis but does not provide protection.

Thread: [#1 (Dean Rasheed, 2026-06-21)](https://www.postgresql.org/message-id/CAEZATCUbTj5MraZsimV8VYd1M3zrhP8soVJRZ%2BWvYyea6moefg%40mail.gmail.com), [#28 (Dean Rasheed, 2026-07-06)](https://www.postgresql.org/message-id/CAEZATCUhknD0kAkXVJeehvP10sEAwV8LCCNecrRvbBLm2vavCQ%40mail.gmail.com), [#37 (Zsolt Parragi, 2026-07-14)](https://www.postgresql.org/message-id/CAN4CZFPEZm_eZair%3D%3DzLSA%3D2Tca%3DWZU3Ee%2BddTeo-UaMkHJcVA%40mail.gmail.com), [#45 (Dean Rasheed, 2026-09-17)](https://www.postgresql.org/message-id/CAEZATCUvPKqEvJUn%2BTP21yD_-DX%3DVqaJ_ggKTK1ArEGFgfRisQ%40mail.gmail.com)

**Question:** What bounds the age of tempfrozenxid for a backend that is idle but still connected, and is a backend that never wrote to a global temporary table exempt from it?

Counter-check: held. Message #1 states the problem and says it is not clear what can be done; v12 adds only monitoring columns and a manual DISCARD. Local temporary tables share the weakness, which lowers the weight, but message #37 item 10 widens the exposure and has no visible answer.

### 2. Permanent catalog shape chosen before the two designs are compared

fixable · level 2, rubric item 6 · design, catalog changes, on-disk format / pg_upgrade, maintenance burden ("who maintains this in 10 years") · `maintenance-fit-3`

The project releases reloncommit, relpersistence 'g' and two new global temporary catalogs. If Andrew Dunstan's approach, with ON COMMIT stored as a reloption and different wraparound handling, later proves better, the switch requires catalog changes plus pg_upgrade and pg_dump compatibility code for the released format.

Message #4 says neither design is a clear winner on the catalog or on wraparound. In message #25 John Naylor asked for a comparison of the operational and maintenance tradeoffs, and no one has written it. The catalog choice has already changed twice in this thread: pg_temp_class and pg_temp_index were added and then removed (messages #40 and #43). Catalog columns and relpersistence values are hard to take back once released.

Thread: [#4 (Andrew Dunstan, 2026-06-22)](https://www.postgresql.org/message-id/251A02BE-A148-4FEF-90EB-5E9559310B8E%40dunslane.net), [#25 (John Naylor, 2026-07-05)](https://www.postgresql.org/message-id/CANWCAZbAFmULzmenAG3kLk68d2Vt%2B3k9pBUSrdxYxUDwUoaj0Q%40mail.gmail.com), [#40 (Dean Rasheed, 2026-09-02)](https://www.postgresql.org/message-id/CAEZATCUn-uA2tvmHcBxexZkQYdm0AU4VDW8reRzbcGDFsLypAg%40mail.gmail.com), [#43 (Dean Rasheed, 2026-09-15)](https://www.postgresql.org/message-id/CAEZATCW9q3B%3D_Nf_qorFXqXbDW5HRMKox_fUaxMGQProdctaWg%40mail.gmail.com)

**Question:** Where do the two series differ on catalog layout and wraparound handling, and why is this design's choice better for long-term maintenance?

Counter-check: held. John Naylor asked for the comparison in message #25 and the thread contains none. The cost of a later switch is smaller than stated, because catalog columns can change between major versions and pg_dump emits ON COMMIT as syntax.

### 3. Session-local pg_class and pg_index overlay may repeat the earlier maintainability objection

fixable · level 2, rubric item 9 · prior rejection, catalog changes, maintenance burden ("who maintains this in 10 years"), ABI / extension API · `prior-attempts-1`

A future patch, or an extension, reads pg_index.indisready or pg_class.relfilenode directly instead of through the GetEffective_* helpers. On a global temporary table that code then acts on another session's base value, for example by skipping an index or trusting stale freeze data. No test or assertion fails until a user hits it.

The earliest global temporary table attempt [P1] was turned away as unmaintainable, with a call for a generic way to split catalog state into local and non-local parts. v11 dropped pg_temp_class and pg_temp_index in favor of an in-memory overlay applied in ScanPgRelation and a set of effective-value helpers (message #41, message #43). Reviewers have already found consumers that bypassed the overlay: a direct indisvalid read in repack.c (message #44) and an infer_arbiter_indexes assertion that is now wrong for global temporary tables (message #46). The thread does not say how such misses would be caught after commit.

**Precedents.**
- [Global temporary table](https://commitfest.postgresql.org/patch/2349/) (Returned with feedback, 2022-06-30); thread: [#1](https://www.postgresql.org/message-id/flat/1A1A6EDC-D0EC-47B0-BD21-C2ACBAEA65E4@alibaba-inc.com), [#2](https://www.postgresql.org/message-id/flat/0566FEB0-74C2-4C84-BEFC-62DBD5E658D3@alibaba-inc.com)

Thread: [#40 (Dean Rasheed, 2026-09-02)](https://www.postgresql.org/message-id/CAEZATCUn-uA2tvmHcBxexZkQYdm0AU4VDW8reRzbcGDFsLypAg%40mail.gmail.com), [#41 (Haibo Yan, 2026-09-09)](https://www.postgresql.org/message-id/CABXr29FsL7FF7A8n07MpM_7fs5vou4GuKCjitQaZoVTrDxBbeg%40mail.gmail.com), [#43 (Dean Rasheed, 2026-09-15)](https://www.postgresql.org/message-id/CAEZATCW9q3B%3D_Nf_qorFXqXbDW5HRMKox_fUaxMGQProdctaWg%40mail.gmail.com), [#44 (Haibo Yan, 2026-09-16)](https://www.postgresql.org/message-id/CABXr29FA3sMWtYde4RxD-UJR5-SKsFSB12Vz%3DGdbCsNaPYX8TQ%40mail.gmail.com), [#46 (Haibo Yan, 2026-09-18)](https://www.postgresql.org/message-id/CABXr29EikqFkBSU9Q5wX9oVOj%3Dy3vjL5bJTRsuKBFgM3jDGwww%40mail.gmail.com)

**Question:** What mechanism, such as an assertion or a check, makes a direct read of a session-local pg_class or pg_index field fail loudly for a global temporary relation?

Counter-check: held. Messages #44 and #46 show two readers that bypassed the overlay, both found by review, and the material names no check that fails loudly. [P1] supports this only loosely, because its recorded reason concerns a different design.

### 4. DISCARD ALL does not clear global temporary table data

fixable · level 2, rubric item 13 · security / privileges, SQL-standard conflict or non-standard syntax · `standards-compat-1`

A connection pooler resets each server connection with DISCARD ALL before giving it to the next client. Client A fills a global temporary table and disconnects. Client B gets the same backend and reads A's rows, and nothing reports an error.

Message #37 showed that rows and sequence values survive DISCARD ALL. Message #39 answers with a separate DISCARD GLOBAL TEMP that works independently of DISCARD TEMP, and the brief lists it as new syntax. The material does not say whether DISCARD ALL now includes it. Poolers rely on DISCARD ALL meaning that no session state carries over to the next client.

Thread: [#37 (Zsolt Parragi, 2026-07-14)](https://www.postgresql.org/message-id/CAN4CZFPEZm_eZair%3D%3DzLSA%3D2Tca%3DWZU3Ee%2BddTeo-UaMkHJcVA%40mail.gmail.com), [#39 (Dean Rasheed, 2026-07-15)](https://www.postgresql.org/message-id/CAEZATCVOsXa1dnuJvqpbQ_VBiOXfgdMUW2rADbRSX6Ze0KnB5g%40mail.gmail.com)

**Question:** Does DISCARD ALL discard global temporary data and sequence state, and if not, how does a pooler user learn that DISCARD ALL is no longer a full reset?

Counter-check: held. Message #37 shows rows and sequence state surviving DISCARD ALL, and message #39 describes DISCARD GLOBAL TEMP as independent of DISCARD TEMP. The visible prose does not say whether DISCARD ALL now includes it, so the pooler scenario stands.

### 5. pg_class.relfrozenxid of zero breaks existing wraparound monitoring queries

fixable · level 2, rubric item 12 · catalog changes, maintenance burden ("who maintains this in 10 years") · `standards-compat-2`

A DBA runs the wraparound query from the maintenance chapter, or a monitoring tool built on it. Each global temporary table has relkind 'r' and relfrozenxid 0, so age() returns its maximum value. The alert fires at once, and the real per-session horizon sits in a set-returning function the tool never calls.

Patch 0006 sets pg_class.relfrozenxid and relminmxid to zero for global temporary relations. The query in maintenance.sgml still filters on relkind IN ('r','m') without change. relpages and reltuples likewise become values that pg_gtr_info() overrides, so tools that read pg_class get base values with no sign that these are not the real ones. Haibo Yan raised the base-versus-effective model in message #15, and the answer so far is documentation, which existing tools will not read.

Thread: [#15 (Haibo Yan, 2026-06-30)](https://www.postgresql.org/message-id/CABXr29F5yVOf6johLW7WY-eXEaJrHq1_f6QFjhUT5nOEaHtfFA%40mail.gmail.com), [#22 (Dean Rasheed, 2026-07-03)](https://www.postgresql.org/message-id/CAEZATCWMYVFcvzPuGDwP1T%3D53zUs8%2BE2BGGi3YZYS3nh1thRDQ%40mail.gmail.com), [#28 (Dean Rasheed, 2026-07-06)](https://www.postgresql.org/message-id/CAEZATCUhknD0kAkXVJeehvP10sEAwV8LCCNecrRvbBLm2vavCQ%40mail.gmail.com)

**Question:** How do the documented wraparound query and common external tools that read pg_class.relfrozenxid avoid reporting every global temporary table at maximum age?

Counter-check: held. age() returns its maximum for an invalid XID, and the maintenance.sgml hunk keeps the `relkind IN ('r', 'm')` query unchanged. The documented query would report every global temporary table at maximum age.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | Adds pg_class.reloncommit, relpersistence 'g' and two new catalogs; patch 0002 touches pg_upgrade/info.c. |
| WAL / replication / standby | **yes** | Standby behaviour for global temporary tables is not defined; publications and subscriptions reject them. |
| wire protocol | no |  |
| catalog changes | **yes** | New catalogs pg_temp_statistic and pg_temp_statistic_ext_data; pg_class.relfrozenxid is zero for global temporary relations. |
| ABI / extension API | **yes** | ambuildempty() gains a fork number argument, so every out-of-tree index access method must change. |
| SQL-standard conflict or non-standard syntax | **yes** | GLOBAL in CREATE TEMP TABLE gets a meaning; the default ON COMMIT action differs from the standard. |
| new GUC | no |  |
| planner behaviour change | **yes** | Selectivity functions read pg_temp_statistic and in-memory relation statistics for global temporary tables. |
| pg_dump / restore round-trip | **yes** | pg_dump must emit GLOBAL TEMP tables and sequences with their ON COMMIT action; patches 0002 and 0004 touch pg_dump.c. |
| psql UX conventions | **yes** | \d shows the ON COMMIT action; tab completion and describe.c change. |
| security / privileges | **yes** | A session's usage entry can block the owner's DDL, and DISCARD ALL may leave data for the next pooled client. |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | Every future reader of about ten pg_class and pg_index fields must use the session overlay helpers. |
| performance on the hot path | **yes** | Message #43 runs the invalidation check at every transaction start and command start, for all sessions. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. What bounds the age of tempfrozenxid for a backend that is idle but connected, and is a backend that never wrote to a global temporary table exempt?
2. Where do this series and Andrew Dunstan's differ on catalog layout and wraparound handling, and why is this design's choice better for long-term maintenance?
3. What assertion or check makes a direct read of a session-local pg_class or pg_index field fail loudly for a global temporary relation?
4. Does DISCARD ALL discard global temporary table data and sequence state, and if not, how does a pooler user learn that it is no longer a full reset?
5. What may a hot standby session do with a global temporary table, and which tests show that catalog-wide queries on a standby work?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. It needs a committer who knows the relcache, catalog and VACUUM freeze-horizon code. The material does not mark any participant as a committer.
- **Liability:** The owner signs for the session overlay of pg_class and pg_index fields, the shared usage hash, the PGPROC freeze fields, two new catalogs and the ambuildempty() change. Released syntax and catalog shape stay indefinitely.
- **Author follow-up:** The author posted twelve versions in under five months and fixed the bugs reviewers reported, which suggests he stays for fixes.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** a sizeable minority
- **Maintenance burden:** high
- **Review cost:** high
- **Why:** Demand is shown by two independent series, earlier attempts [P1][P3][P4] and users citing migration and catalog bloat. The nine patches touch the relcache, VACUUM, the planner and every index access method, and the wraparound and design-comparison questions are open.

## Appendix

### Held up in the counter-check, not listed above

- `standards-compat-3` (fixable, level 2) Default ON COMMIT action differs from the standard and Oracle. Not listed: not among the strongest objections.
- `replication-ondisk-2` (fixable, level 2) Physical standby behaviour for global temporary tables is not defined. Not listed: not among the strongest objections.
- `security-2` (fixable, level 2) A role without privileges can register usage and block the owner's DDL. Not listed: not among the strongest objections.
- `maintenance-fit-4` (fixable, level 2) No measurements of overhead for sessions that never use the feature. Not listed: not among the strongest objections.

### Refuted in the counter-check

- `standards-compat-4` Existing CREATE GLOBAL TEMP scripts change meaning on upgrade. Refuted (already handled): Postgres has emitted a deprecation warning for GLOBAL in temporary table creation for many releases. The CREATE TABLE compatibility section says a later version might adopt the standard meaning, so existing users have had the warning the objection asks for.
- `replication-ondisk-3` pg_class.relfrozenxid of zero for tables breaks freeze monitoring. Refuted (not specific): Duplicate of standards-compat-2.
- `security-1` Using a global temporary table may bypass the TEMP privilege. Refuted (no failure scenario): TEMP governs creating objects in a session's temporary schema; it is not a limit on storage use. A role granted INSERT on any table can already fill storage, and creating the global temporary table needs CREATE on a regular schema, so no privilege is bypassed.
- `maintenance-fit-1` Session overlay of pg_class and pg_index fields fails silently when bypassed. Refuted (not specific): Duplicate of prior-attempts-1. Its claim that [P1] rejected "the same overlay pattern" is also not in the recorded reason for [P1].
- `maintenance-fit-2` Idle sessions can hold back datfrozenxid with no automatic remedy. Refuted (not specific): Duplicate of replication-ondisk-1.

### Review angles

- prior-attempts: 1 objection(s). Only the [P1] maintainability reason still applies. [P5]'s reason about invisible dependencies does not apply, because the definitions here stay in the shared catalogs. [P9] has no recorded reason.
- standards-compat: 4 objection(s). Objections are limited to the user-facing contract. pg_dump round-trip of ON COMMIT was fixed in message #36, and the material does not let me check pg_upgrade handling.
- replication-ondisk: 3 objection(s). WAL is not involved because global temporary tables are always unlogged. Logical replication is excluded through publication and subscription checks, which the thread settled.
- security: 2 objection(s). Statistics exposure looks handled: pg_temp_statistic is limited to superuser, and pg_stats filters rows by privilege. Publications and subscriptions are rejected for global temporary tables. These points could not be checked without the code.
- maintenance-fit: 4 objection(s). Objections listed above.
- already-exists: 0 objection(s). Nothing in core gives a shared definition with session-local data. Local temp tables, created per session, give private data but keep the catalog churn that message #8 names as the main problem. Extensions that emulate this by creating a local temp table on first use have the same churn. They also cannot change relpersistence, local buffers, per-session statistics or the datfrozenxid calculation without core changes, so core placement is justified.
- demand: 0 objection(s). The material shows real, lasting demand: two independent implementations (messages #1, #2), several earlier attempts since 2019 [P1][P3][P4], the SQL-standard syntax, and users citing migration (#5), catalog bloat (#8) and production workflows (#11, #24). Costs that reach non-users, such as the ambuildempty() signature change and pg_stats becoming a UNION ALL view, are design questions for other angles.

### Precedents retrieved

- P1: [Global temporary table](https://commitfest.postgresql.org/patch/2349/) (cf, Returned with feedback, 2022-06-30) - used in the review; reason summarised from its thread
- P2: [Forbid accessing other session's temporary tables](https://commitfest.postgresql.org/patch/5392/) (cf, Withdrawn, 2024-11-30) - judged not relevant; reason summarised from its thread
- P3: [Global temporary tables](https://commitfest.postgresql.org/patch/2233/) (cf, Returned with feedback, 2020-02-01) - used in the review; reason summarised from its thread
- P4: [global temporary table v37](https://commitfest.postgresql.org/patch/2845/) (cf, Withdrawn, 2020-11-25) - used in the review; reason summarised from its thread
- P5: [Temporary tables that do not bloat pg_catalog (a.k.a fast temp tables)](https://commitfest.postgresql.org/patch/688/) (cf, Rejected, 2016-09-20) - used in the review; reason summarised from its thread
- P6: [Allow tablespaces on RAM-based partitions for temporary objects](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Tablespaces) (do_not_want, Not worth doing, date unknown) - judged not relevant; reason summarised from its thread
- P7: [We broke the defense against accessing other sessions' temp tables](https://commitfest.postgresql.org/patch/6073/) (cf, Withdrawn, 2026-05-14) - judged not relevant; reason summarised from its thread
- P8: [Prepare Transaction support for ON COMMIT DROP temporary tables](https://commitfest.postgresql.org/patch/1954/) (cf, Returned with feedback, 2019-02-01) - judged not relevant; reason summarised from its thread
- P9: [Update relfrozenxmin when truncating temp tables](https://commitfest.postgresql.org/patch/3358/) (cf, Returned with feedback, 2024-01-22) - used in the review
- P10: [\[PATCH\] Equivalence Class Filters](https://commitfest.postgresql.org/patch/3524/) (cf, Returned with feedback, 2022-12-17) - judged not relevant
- P11: [Revert "Skip redundant anti-wraparound vacuums"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=dd9ac7d5d80608a640bb82cffb6a805ce84cf112) (revert, Reverted, 2020-03-30) - judged not relevant
- P12: [Add support for ON UPDATE/DELETE actions on ALTER CONSTRAINT](https://commitfest.postgresql.org/patch/1533/) (cf, Returned with feedback, 2021-04-08) - judged not relevant
- P13: [allow specifying action when standby encounters incompatible parameter settings](https://commitfest.postgresql.org/patch/3619/) (cf, Withdrawn, 2022-07-25) - judged not relevant
- P14: [rich constructor for plpy.Error class, possible raise structured exception](https://commitfest.postgresql.org/patch/387/) (cf, Returned with feedback, 2016-02-03) - judged not relevant
- P15: [Revert "Temporarily instrument postgres_fdw test to look for statistics changes."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=04e7ecadf64d18f67ca1d0632d8ab71f120ca5e3) (revert, Reverted, 2018-03-08) - judged not relevant
- P16: [Revert refactoring of restore command code to shell_restore.c](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2f6e15ac93c58c1140e4a4affe61e78f7346497a) (revert, Reverted, 2023-02-05) - judged not relevant
- P17: [Revert "Temporarily add some probes of tenk1's relallvisible in create_index.sql."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=91998539b227dfc6dd091714da7d106f2c95a321) (revert, Reverted, 2022-04-15) - judged not relevant
- P18: [Prevent extension creation in temporary schemas](https://commitfest.postgresql.org/patch/1969/) (cf, Returned with feedback, 2019-03-07) - judged not relevant
- P19: [autovacuum may skip tables when session_authorization/role is set on database](https://commitfest.postgresql.org/patch/4702/) (cf, Returned with feedback, 2024-01-11) - judged not relevant
- P20: [Session Start/End hooks](https://commitfest.postgresql.org/patch/1318/) (cf, Returned with feedback, 2017-11-28) - judged not relevant
- P21: [ALTER TABLE: emit NOTICE when ONLY is omitted but no partition recursion occurs](https://commitfest.postgresql.org/patch/6424/) (cf, Withdrawn, 2026-01-22) - judged not relevant
- P22: [background sessions](https://commitfest.postgresql.org/patch/873/) (cf, Returned with feedback, 2017-04-06) - judged not relevant
- P23: [session username in default psql prompt](https://commitfest.postgresql.org/patch/4856/) (cf, Withdrawn, 2024-03-23) - judged not relevant
- P24: [Support target_session_attrs=read-only and eliminate the added round-trip to detect session status.](https://commitfest.postgresql.org/patch/1148/) (cf, Returned with feedback, 2017-11-28) - judged not relevant

### Warnings

- attachment limit (8) reached; v12-0009-Add-DISCARD-GLOBAL-TEMP.patch not read
- attachment limit (8) reached; v1-0001-Basic-support-for-global-temporary-tables.patch not read
- attachment limit (8) reached; v1-0002-Support-indexes-on-global-temporary-tables.patch not read
- attachment limit (8) reached; v1-0004-Support-global-temporary-catalog-tables-and-add-p.patch not read
- attachment limit (8) reached; v1-0003-Support-global-temporary-sequences.patch not read
- attachment limit (8) reached; v1-0005-Add-relation-statistics-columns-to-pg_temp_class.patch not read
- attachment limit (8) reached; v1-0007-Add-pg_temp_statistic-global-temporary-catalog-ta.patch not read
- attachment limit (8) reached; v1-0009-Add-pg_temp_index-global-temporary-catalog-table.patch not read
- attachment limit (8) reached; v1-0008-Add-pg_temp_statistic_ext_data-global-temporary-c.patch not read
- attachment limit (8) reached; v1-0006-Add-relfrozenxid-and-relminmxid-columns-to-pg_tem.patch not read

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 14; wall time: 249 s; cost reported by the CLI: $9.96
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 5 of 24 kept
- Tone check: 128 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:08:53.598Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

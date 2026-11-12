# Design pre-review: Global temporary tables

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6918](https://commitfest.postgresql.org/patch/6918/); [thread](https://www.postgresql.org/message-id/flat/CAEZATCUbTj5MraZsimV8VYd1M3zrhP8soVJRZ%2BWvYyea6moefg%40mail.gmail.com), 47 of 47 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). Demand is shown and the problem is stated, so levels 1a and 1b pass. Level 2 has open design points the list must settle before code review. Message #1 says an idle session can hold back datfrozenxid with no remedy (replication-ondisk-1). No one has written the comparison with the competing series that message #25 asked for (maintenance-fit-3). Nothing fails loudly when code bypasses the session overlay (prior-attempts-1). All are fixable.

Not checked: Without the code, this review could not check whether DISCARD ALL covers global temporary data, when usage entries are registered relative to privilege checks, how pg_upgrade handles these relations, or what the command-start invalidation check costs.

## Prior attempts

- [Global temporary table](https://www.postgresql.org/message-id/flat/1A1A6EDC-D0EC-47B0-BD21-C2ACBAEA65E4@alibaba-inc.com) (same idea; Returned with feedback, 2022-06-30). "I think there's just no way that it can be merged with anything close to the current design - it's unmaintainable. The need for the feature doesn't change that." ([message](https://www.postgresql.org/message-id/20220302211759.k3y2bcfchkukjt7y%40alap3.anarazel.de)) Different now: This series is split into nine separately reviewable patches, keeps per-session relation and index state in memory, and allows rewrites such as REPACK and VACUUM FULL. The recorded objection was to a different design, so it applies only loosely.
- [global temporary table v37](https://www.postgresql.org/message-id/flat/1A1A6EDC-D0EC-47B0-BD21-C2ACBAEA65E4@alibaba-inc.com) (same idea; Withdrawn, 2020-11-25). The visible messages do not show why. The entry was marked Withdrawn on the same day the author posted a rebased version fixing the whitespace problems and the relcache reference leak that Pavel Stehule had reported, and the thread shows review still ongoing rather than an objection to the design. (AI summary of the thread) Different now: That entry was withdrawn with no recorded reason, so it does not show what this proposal would need to fix.
- [Global temporary tables](https://www.postgresql.org/message-id/flat/73954ab7-44d3-b37b-81a3-69bdcbb446f7@postgrespro.ru) (similar idea; Returned with feedback, 2020-02-01). "As there has been very little activity since then, I've marked it as returned with feedback." ([message](https://www.postgresql.org/message-id/20200201114954.mpmpfmdaxsnnj3ux%40development)) Different now: That attempt closed for inactivity. This proposal uses local buffers and has active review and steady new versions, so the recorded reason does not apply.

## Objections a reviewer is likely to raise

### 1. Idle sessions that used a global temporary table can block wraparound protection

A connection pooler keeps 200 backends open for weeks, and each touched a global temporary table once. Their tempfrozenxid values stop datfrozenxid and clog truncation from advancing. Autovacuum cannot fix this, so the DBA learns about it from wraparound warnings and must find and terminate or DISCARD each backend.

Message #1 and the 0006 commit message state that datfrozenxid advances only if each session runs VACUUM from time to time, and autovacuum is excluded. Local temporary tables have the same weakness, but with a global temporary table any application session that reads a shared definition can pin the horizon. Message #37 item 10 also reports that a database-wide VACUUM opens every global temporary table. Exposing tempfrozenxid in pg_stat_activity (message #28) helps diagnosis but does not provide protection.

**Question:** What bounds the age of tempfrozenxid for a backend that is idle but still connected, and is a backend that never wrote to a global temporary table exempt from it?

<sub>fixable, level 2, WAL / replication / standby; messages [#1](https://www.postgresql.org/message-id/CAEZATCUbTj5MraZsimV8VYd1M3zrhP8soVJRZ%2BWvYyea6moefg%40mail.gmail.com), [#28](https://www.postgresql.org/message-id/CAEZATCUhknD0kAkXVJeehvP10sEAwV8LCCNecrRvbBLm2vavCQ%40mail.gmail.com), [#37](https://www.postgresql.org/message-id/CAN4CZFPEZm_eZair%3D%3DzLSA%3D2Tca%3DWZU3Ee%2BddTeo-UaMkHJcVA%40mail.gmail.com), [#45](https://www.postgresql.org/message-id/CAEZATCUvPKqEvJUn%2BTP21yD_-DX%3DVqaJ_ggKTK1ArEGFgfRisQ%40mail.gmail.com); `replication-ondisk-1`</sub>

### 2. Permanent catalog shape chosen before the two designs are compared

The project releases reloncommit, relpersistence 'g' and two new global temporary catalogs. If Andrew Dunstan's approach, with ON COMMIT stored as a reloption and different wraparound handling, later proves better, the switch requires catalog changes plus pg_upgrade and pg_dump compatibility code for the released format.

Message #4 says neither design is a clear winner on the catalog or on wraparound. In message #25 John Naylor asked for a comparison of the operational and maintenance tradeoffs, and no one has written it. The catalog choice has already changed twice in this thread: pg_temp_class and pg_temp_index were added and then removed (messages #40 and #43). Catalog columns and relpersistence values are hard to take back once released.

**Question:** Where do the two series differ on catalog layout and wraparound handling, and why is this design's choice better for long-term maintenance?

<sub>fixable, level 2, design; messages [#4](https://www.postgresql.org/message-id/251A02BE-A148-4FEF-90EB-5E9559310B8E%40dunslane.net), [#25](https://www.postgresql.org/message-id/CANWCAZbAFmULzmenAG3kLk68d2Vt%2B3k9pBUSrdxYxUDwUoaj0Q%40mail.gmail.com), [#40](https://www.postgresql.org/message-id/CAEZATCUn-uA2tvmHcBxexZkQYdm0AU4VDW8reRzbcGDFsLypAg%40mail.gmail.com), [#43](https://www.postgresql.org/message-id/CAEZATCW9q3B%3D_Nf_qorFXqXbDW5HRMKox_fUaxMGQProdctaWg%40mail.gmail.com); `maintenance-fit-3`</sub>

### 3. Session-local pg_class and pg_index overlay may repeat the earlier maintainability objection

A future patch, or an extension, reads pg_index.indisready or pg_class.relfilenode directly instead of through the GetEffective_* helpers. On a global temporary table that code then acts on another session's base value, for example by skipping an index or trusting stale freeze data. No test or assertion fails until a user hits it.

The earliest global temporary table attempt [P1] was turned away as unmaintainable, with a call for a generic way to split catalog state into local and non-local parts. v11 dropped pg_temp_class and pg_temp_index in favor of an in-memory overlay applied in ScanPgRelation and a set of effective-value helpers (message #41, message #43). Reviewers have already found consumers that bypassed the overlay: a direct indisvalid read in repack.c (message #44) and an infer_arbiter_indexes assertion that is now wrong for global temporary tables (message #46). The thread does not say how such misses would be caught after commit.

Precedent: [Global temporary table](https://www.postgresql.org/message-id/flat/1A1A6EDC-D0EC-47B0-BD21-C2ACBAEA65E4@alibaba-inc.com) (Returned with feedback, 2022-06-30)

**Question:** What mechanism, such as an assertion or a check, makes a direct read of a session-local pg_class or pg_index field fail loudly for a global temporary relation?

<sub>fixable, level 2, prior rejection; messages [#40](https://www.postgresql.org/message-id/CAEZATCUn-uA2tvmHcBxexZkQYdm0AU4VDW8reRzbcGDFsLypAg%40mail.gmail.com), [#41](https://www.postgresql.org/message-id/CABXr29FsL7FF7A8n07MpM_7fs5vou4GuKCjitQaZoVTrDxBbeg%40mail.gmail.com), [#43](https://www.postgresql.org/message-id/CAEZATCW9q3B%3D_Nf_qorFXqXbDW5HRMKox_fUaxMGQProdctaWg%40mail.gmail.com), [#44](https://www.postgresql.org/message-id/CABXr29FA3sMWtYde4RxD-UJR5-SKsFSB12Vz%3DGdbCsNaPYX8TQ%40mail.gmail.com), [#46](https://www.postgresql.org/message-id/CABXr29EikqFkBSU9Q5wX9oVOj%3Dy3vjL5bJTRsuKBFgM3jDGwww%40mail.gmail.com); `prior-attempts-1`</sub>

### 4. DISCARD ALL does not clear global temporary table data

A connection pooler resets each server connection with DISCARD ALL before giving it to the next client. Client A fills a global temporary table and disconnects. Client B gets the same backend and reads A's rows, and nothing reports an error.

Message #37 showed that rows and sequence values survive DISCARD ALL. Message #39 answers with a separate DISCARD GLOBAL TEMP that works independently of DISCARD TEMP, and the brief lists it as new syntax. The material does not say whether DISCARD ALL now includes it. Poolers rely on DISCARD ALL meaning that no session state carries over to the next client.

**Question:** Does DISCARD ALL discard global temporary data and sequence state, and if not, how does a pooler user learn that DISCARD ALL is no longer a full reset?

<sub>fixable, level 2, security / privileges; messages [#37](https://www.postgresql.org/message-id/CAN4CZFPEZm_eZair%3D%3DzLSA%3D2Tca%3DWZU3Ee%2BddTeo-UaMkHJcVA%40mail.gmail.com), [#39](https://www.postgresql.org/message-id/CAEZATCVOsXa1dnuJvqpbQ_VBiOXfgdMUW2rADbRSX6Ze0KnB5g%40mail.gmail.com); `standards-compat-1`</sub>

### 5. pg_class.relfrozenxid of zero breaks existing wraparound monitoring queries

A DBA runs the wraparound query from the maintenance chapter, or a monitoring tool built on it. Each global temporary table has relkind 'r' and relfrozenxid 0, so age() returns its maximum value. The alert fires at once, and the real per-session horizon sits in a set-returning function the tool never calls.

Patch 0006 sets pg_class.relfrozenxid and relminmxid to zero for global temporary relations. The query in maintenance.sgml still filters on relkind IN ('r','m') without change. relpages and reltuples likewise become values that pg_gtr_info() overrides, so tools that read pg_class get base values with no sign that these are not the real ones. Haibo Yan raised the base-versus-effective model in message #15, and the answer so far is documentation, which existing tools will not read.

**Question:** How do the documented wraparound query and common external tools that read pg_class.relfrozenxid avoid reporting every global temporary table at maximum age?

<sub>fixable, level 2, catalog changes; messages [#15](https://www.postgresql.org/message-id/CABXr29F5yVOf6johLW7WY-eXEaJrHq1_f6QFjhUT5nOEaHtfFA%40mail.gmail.com), [#22](https://www.postgresql.org/message-id/CAEZATCWMYVFcvzPuGDwP1T%3D53zUs8%2BE2BGGi3YZYS3nh1thRDQ%40mail.gmail.com), [#28](https://www.postgresql.org/message-id/CAEZATCUhknD0kAkXVJeehvP10sEAwV8LCCNecrRvbBLm2vavCQ%40mail.gmail.com); `standards-compat-2`</sub>

## Questions for the author

1. What bounds the age of tempfrozenxid for a backend that is idle but connected, and is a backend that never wrote to a global temporary table exempt?
2. Where do this series and Andrew Dunstan's differ on catalog layout and wraparound handling, and why is this design's choice better for long-term maintenance?
3. What assertion or check makes a direct read of a session-local pg_class or pg_index field fail loudly for a global temporary relation?
4. Does DISCARD ALL discard global temporary table data and sequence state, and if not, how does a pooler user learn that it is no longer a full reset?
5. What may a hot standby session do with a global temporary table, and which tests show that catalog-wide queries on a standby work?

## Touch points

- **on-disk format / pg_upgrade:** Adds pg_class.reloncommit, relpersistence 'g' and two new catalogs; patch 0002 touches pg_upgrade/info.c.
- **WAL / replication / standby:** Standby behaviour for global temporary tables is not defined; publications and subscriptions reject them.
- **catalog changes:** New catalogs pg_temp_statistic and pg_temp_statistic_ext_data; pg_class.relfrozenxid is zero for global temporary relations.
- **ABI / extension API:** ambuildempty() gains a fork number argument, so every out-of-tree index access method must change.
- **SQL-standard conflict or non-standard syntax:** GLOBAL in CREATE TEMP TABLE gets a meaning; the default ON COMMIT action differs from the standard.
- **planner behaviour change:** Selectivity functions read pg_temp_statistic and in-memory relation statistics for global temporary tables.
- **pg_dump / restore round-trip:** pg_dump must emit GLOBAL TEMP tables and sequences with their ON COMMIT action; patches 0002 and 0004 touch pg_dump.c.
- **psql UX conventions:** \d shows the ON COMMIT action; tab completion and describe.c change.
- **security / privileges:** A session's usage entry can block the owner's DDL, and DISCARD ALL may leave data for the next pooled client.
- **maintenance burden ("who maintains this in 10 years"):** Every future reader of about ten pg_class and pg_index fields must use the session overlay helpers.
- **performance on the hot path:** Message #43 runs the invalidation check at every transaction start and command start, for all sessions.

## Who would commit it, and when

- Owner: silent. It needs a committer who knows the relcache, catalog and VACUUM freeze-horizon code. The material does not mark any participant as a committer.
- Priority: fix design first; audience a sizeable minority; review cost high. Demand is shown by two independent series, earlier attempts [P1][P3][P4] and users citing migration and catalog bloat. The nine patches touch the relcache, VACUUM, the planner and every index access method, and the wraparound and design-comparison questions are open.

---

[roast-full.md](roast-full.md) has 4 more objection(s) that held, 5 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 10.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

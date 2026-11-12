# Design pre-review: reindexdb: Fix schema name lookup in parallel mode

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7477](https://commitfest.postgresql.org/patch/7477/) (Needs review); authors: Koshi Shibagaki (k-zshiba)
- Thread: [\[PATCH\] reindexdb: Fix schema name lookup in parallel mode](https://www.postgresql.org/message-id/flat/TY5PR01MB1844509B94E3179A2A2434082FA9E2%40TY5PR01MB18445.jpnprd01.prod.outlook.com), started 2026-10-13
- Messages read: 6 of 6
- Patch prose read: v2-0001-reindexdb-Fix-schema-name-lookup-in-parallel-mode.patch, v1-0001-Fix-schema-name-lookup-in-parallel-reindexdb.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No surviving objection questions the fix on master: it is a small client-side bug fix that copies the regnamespace lookup vacuumdb already uses. The one surviving objection, maintenance-fit-1, concerns the request to backpatch through 15. Today `-j -S Sales` reindexes a schema created as "Sales"; after the patch that job fails. Only a committer can decide whether a minor release may do that.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read. Not checked: whether the catalog query quotes the argument safely and is schema-qualified, and how the error for a missing schema reads.

## The proposal, restated

**Problem.** With -j, reindexdb compares --schema arguments directly with pg_namespace.nspname, while serial mode sends REINDEX SCHEMA. So in parallel mode unquoted names are not case folded, quoted names do not match, and a missing schema is silently ignored with exit code 0. Users running reindexdb -S with -j get a run that reindexes nothing and reports success.

**Proposed design.** Parallel mode resolves each schema argument through a ::regnamespace cast and filters the table list by namespace OID, as vacuumdb does. A missing schema now fails the catalog query before any REINDEX runs, and existing empty schemas still succeed. The patch extends the TAP tests for case folding, a quoted mixed-case name, and a missing schema given with an empty one.

**User-visible surface.**
- reindexdb -S/--schema with -j: arguments now follow identifier rules (case folding, double quotes).
- reindexdb -S with -j: a missing schema is now an error with a non-zero exit code.
- reindexdb -S with -j: numeric arguments are taken as schema OIDs, and '-' matches nothing.
- The error output for a missing schema includes the catalog query text, as -t does today.

**What the author claims.**
- Parallel mode now matches serial mode for case folding, quoted names and missing schemas.
- The approach matches vacuumdb, which uses regnamespace in both serial and parallel modes.
- Existing empty schemas still succeed.
- The bug exists since -j was added in Postgres 13, and the author asks to backpatch through 15.

**Thread state.** Takeshi Ideriha and Joao Rito support the regnamespace approach, and Rito tested v1 and v2 and calls v2 ready for committer. No committer has replied, and the backpatch question is open; v2 does not apply to 15 to 17.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-10-13; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Backpatching turns silent successes into failures in minor releases

fixable · level 2, rubric item 6 · backpatchability, maintenance burden ("who maintains this in 10 years") · `maintenance-fit-1`

A site runs a nightly `reindexdb -j 4 -S Sales` job on 16.x, where `Sales` is a schema created as `"Sales"`. After a routine minor update the job exits non-zero and its scheduler raises an alert. Nobody changed the job, and the release notes are the only place the cause is explained.

The fix itself is small and copies the vacuumdb approach, so on master the cost is low. The open part is message #5's request to backpatch through 15, which changes exit codes for scripts that work today, even if they reindex nothing. Message #4 agrees these scripts will now fail and asks for a release note. Message #6 says no port to 15-17 is posted or tested, so the backpatch is not yet ready for review.

Thread: [#4 (João Rito, 2026-11-10)](https://www.postgresql.org/message-id/CADMs5OOOyzwMa7LYKZU_YxjbCVbx-5%3DscF8%2BSq7x4kG_s34Mpg%40mail.gmail.com), [#5 ("Koshi Shibagaki (Fujitsu)", 2026-11-10)](https://www.postgresql.org/message-id/TY5PR01MB1844575BDDB2E1B415E75A094FA722%40TY5PR01MB18445.jpnprd01.prod.outlook.com), [#6 (Joao Rito, 2026-11-10)](https://www.postgresql.org/message-id/20261110131410.60486-1-joaovictor.rito%40gmail.com)

**Question:** Would a master-only fix, or a backpatch that only warns about unmatched schemas, be enough, and if not, why must back branches start failing these scripts?

Counter-check: held. Parallel mode compares the argument with nspname today, so `-S Sales` matches schema "Sales" and reindexes it; after the patch that job fails. No committer has answered the backpatch question, and no port to 15-17 is posted.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | **yes** | The regnamespace input accepts numeric OIDs and '-' and does not recognize U& identifiers or SQL comments, unlike REINDEX SCHEMA. |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The author asks to backpatch through 15, which changes exit codes for existing `reindexdb -S ... -j` scripts; v2 does not apply to 15 to 17. |
| maintenance burden ("who maintains this in 10 years") | **yes** | A backpatch needs separate, tested patch versions for 15 to 17, which are not posted. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Why must branches 15 to 18 start failing scripts that pass today, and would a master-only fix or a back-branch warning for unmatched schemas be enough?
2. Can you post and test patch versions that apply to 15, 16 and 17, since v2 does not apply there?
3. What would the release note say to users whose `-j -S` job matched a mixed-case schema without double quotes and will now fail?
4. Does the reindexdb documentation need a change to say that -S arguments follow identifier rules and accept numeric OIDs in parallel mode?
5. Is it acceptable that serial mode keeps REINDEX SCHEMA syntax while parallel mode uses regnamespace syntax, or should both modes use regnamespace as vacuumdb does?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. A committer who knows the src/bin/scripts client tools and the back-branch policy for behaviour changes would have to own it; none has replied in the visible thread.
- **Liability:** On master, a small client-side change with TAP tests. With a backpatch, the owner also signs for exit-code changes in minor releases of 15 to 18 and for any reports from broken scheduled jobs.
- **Author follow-up:** The author replied to review in message #3 and posted v2 with message #5, which suggests continued engagement.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch touches reindexdb.c and one TAP test file, is about 3.5 KB, and reuses the vacuumdb approach. It affects only users of -S with -j, and the remaining decision is the backpatch scope.

## Appendix

### Refuted in the counter-check

- `standards-compat-1` Backpatching turns silent successes into failures in minor releases. Refuted (not specific): Duplicate of maintenance-fit-1. Its first example also does not fail: after the patch `-S Sales` folds to `sales`, which exists, so the job starts to reindex and still exits 0.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. The only listed row, [P5], reverts a change that forbade DROP SCHEMA on temporary namespaces. It shares only the word "schema", and its recorded reason has nothing to do with reindexdb or with how client tools look up names. No thread message points to an earlier discussion of this idea. The open backpatch question belongs to another angle.
- standards-compat: 1 objection(s). The standards-compat case against the fix on master is weak. It reuses vacuumdb's regnamespace scheme and -t's regclass lookup. The thread already settled that OIDs and '-' are accepted (messages #2-#4), so I dropped that point.
- replication-ondisk: 0 objection(s). The patch changes only how the reindexdb client builds its catalog query for -S with -j. It adds no on-disk format, WAL record, catalog column, wire protocol message or extension ABI, so this angle has nothing to object to. The back-branch behaviour change is for another angle.
- security: 0 objection(s). No new privileges or exposure. The cast resolves names only for the connected user, so schema existence stays as visible as pg_namespace already makes it. Message #4 shows the argument is quoted as a string literal before the regnamespace cast, the same scheme as vacuumdb and -t. A code check could still confirm the catalog query is schema-qualified under the scripts' secure search_path.
- maintenance-fit: 1 objection(s). For master, I see no maintenance or fit objection. The change is a small client-side fix that reuses the regnamespace and regclass lookups already in vacuumdb and in reindexdb -t. It adds no hot-path cost and no new API.
- already-exists: 0 objection(s). This is a bug fix in reindexdb parallel mode. It reuses the regnamespace lookup that vacuumdb and reindexdb -t already use, so it adds no new mechanism. No existing feature or extension removes the bug, and the serial-mode workaround gives up -j. [P5] is a revert about DROP SCHEMA on temp namespaces, which is unrelated.
- demand: 0 objection(s). This is a bug fix: with -j, reindexdb -S silently reindexes nothing and exits 0, which messages #1 and #4 reproduce. Anyone who uses -S with -j benefits, and users who do not use them bear no cost. Demand does not need separate proof for a fix to silent wrong behaviour that brings parallel mode in line with serial mode and vacuumdb. The back-branch behaviour change is a different angle. [P5] is not relevant.

### Precedents retrieved

- P1: [Add CREATE SCHEMA ... LIKE support](https://commitfest.postgresql.org/patch/6481/) (cf, Returned with feedback, 2026-04-06) - judged not relevant; reason summarised from its thread
- P2: [Tab completion for CREATE SCHEMA AUTHORIZATION](https://commitfest.postgresql.org/patch/3252/) (cf, Returned with feedback, 2021-12-03) - judged not relevant; reason summarised from its thread
- P3: [Prevent extension creation in temporary schemas](https://commitfest.postgresql.org/patch/1969/) (cf, Returned with feedback, 2019-03-07) - judged not relevant; reason summarised from its thread
- P4: [psql: add schema size information to \dn+](https://commitfest.postgresql.org/patch/2010/) (cf, Withdrawn, 2019-02-25) - judged not relevant; reason summarised from its thread
- P5: [Revert "Forbid DROP SCHEMA on temporary namespaces"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=65192e02441cedd106b6abebe0036fb8cc124fb3) (revert, Reverted, 2020-01-08) - used in the review
- P6: [Various small unrelated doc patches: plpgsql, schemas, permissions, oidvector](https://commitfest.postgresql.org/patch/4581/) (cf, Returned with feedback, 2024-10-07) - judged not relevant; reason summarised from its thread
- P7: [pg_get_constraintdef: Schema qualify foreign tables unless pretty printing is enabled](https://commitfest.postgresql.org/patch/3811/) (cf, Returned with feedback, 2022-11-30) - judged not relevant; reason summarised from its thread
- P8: [revert "warn of SECURITY DEFINER schemas for non-sql_body funcs"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f77becbe5a99c9853b1ae7f22aa15babd01368b7) (revert, Reverted, 2022-09-28) - judged not relevant
- P9: [ALTER TABLE SET LOGGED/UNLOGGED on a partitioned table does nothing silently](https://commitfest.postgresql.org/patch/2857/) (cf, Withdrawn, 2021-04-07) - judged not relevant; reason summarised from its thread
- P10: [Revert "vacuumdb: Add option for analyzing only relations missing stats."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=19e57f4f78e4354d9a21c284868373d28bb0d368) (revert, Reverted, 2025-03-07) - judged not relevant
- P11: [Connection string usage for Core Postgresql client applications](https://commitfest.postgresql.org/patch/2354/) (cf, Withdrawn, 2020-01-21) - judged not relevant; reason summarised from its thread
- P12: [pg_upgrade: Pass -j option down to vacuumdb](https://commitfest.postgresql.org/patch/1943/) (cf, Returned with feedback, 2019-04-09) - judged not relevant
- P13: [Return ERROR on COMMIT instead of silently ROLLBACK](https://commitfest.postgresql.org/patch/2522/) (cf, Returned with feedback, 2021-03-25) - judged not relevant
- P14: [Add Hook Functions for Disk Quota Extension](https://commitfest.postgresql.org/patch/1883/) (cf, Returned with feedback, 2020-07-31) - judged not relevant
- P15: [Prevent multibyte characters from being quoted in format() and quote_ident()](https://commitfest.postgresql.org/patch/491/) (cf, Rejected, 2016-01-28) - judged not relevant
- P16: [improve tab-completion behavior when parse text in single quote](https://commitfest.postgresql.org/patch/3267/) (cf, Returned with feedback, 2021-12-03) - judged not relevant
- P17: [Enforce creation of destination folders for source files in pg_regress](https://commitfest.postgresql.org/patch/100/) (cf, Rejected, 2015-03-01) - judged not relevant
- P18: [Allowing control of upper/lower case folding of unquoted identifiers](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Exotic_Features) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P19: [REINDEX CONCURRENTLY](https://commitfest.postgresql.org/patch/1060/) (cf, Returned with feedback, 2017-04-03) - judged not relevant
- P20: [REINDEX CONCURRENTLY](https://commitfest.postgresql.org/patch/1276/) (cf, Returned with feedback, 2018-02-01) - judged not relevant
- P21: [REINDEX filtering in the backend](https://commitfest.postgresql.org/patch/2206/) (cf, Returned with feedback, 2019-09-06) - judged not relevant
- P22: [REINDEX backend filtering](https://commitfest.postgresql.org/patch/2895/) (cf, Withdrawn, 2021-05-08) - judged not relevant
- P23: [Revert "pg_upgrade: Fix quoting of some arguments in pg_ctl command"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=4e818866e57aa9ec13821fd36a2ff60e00e6dea6) (revert, Reverted, 2020-02-10) - judged not relevant
- P24: [Revert "Fix under-quoted filename pattern in pgbench TAP test."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f4593bd2ff0376fbc56dd1b384c7d51e1a7062f0) (revert, Reverted, 2019-01-21) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 12; wall time: 87 s; cost reported by the CLI: $1.03
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 7 from cache, 1 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 56 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:59:39.636Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

# Design pre-review: COPY TO BLACKHOLE / pg_dump -j -Fb

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6788](https://commitfest.postgresql.org/patch/6788/) (Needs review); authors: Jakub Wartak (jakub.wartak.tomtom)
- Thread: [COPY TO BLACKHOLE / pg_dump -j -Fb](https://www.postgresql.org/message-id/flat/CAKZiRmzorOYZH-CvyN98%2BKz3aGdJW5Kj-MirtM0tmjYO3PtASQ%40mail.gmail.com), started 2026-05-20
- Messages read: 6 of 6
- Patch prose read: v1-0001-Add-COPY-TO-BLACKHOLE.patch, v1-0002-Add-BLACKHOLE-destination-format-in-pg_dump-Fb.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**LIKELY-REJECT** (confidence: medium)

The proposal fails at level 1b on maintenance-fit-1 and already-exists-1. COPY TO '/dev/null' already runs the same check, and the thread gives no reason for a permanent COPY keyword beyond 11-14% less CPU. The author accepts in message #6 that pg_dump is the wrong place and considers a client-only pg_amcheck option, which would be a different proposal. Confidence is medium because v1 has no documentation and the pg_amcheck alternative has no design.

First failing level, from the base up: **Level 1b: desirability**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: failed
- Level 2: architecture and design: not evaluated
- Levels 3-5 (implementation): out of scope

Provisional: The level 2 objections on syntax, privileges, progress reporting and per-row cost were not evaluated because level 1b fails. The code was not read, so the behaviour for file and program targets and the size of the CPU saving are unverified.

## The proposal, restated

**Problem.** Users check for corruption with COPY to '/dev/null' or verify_heapam(), and the author says each finds different kinds of corruption. pg_dump -j works only with -Fd, so a parallel logical check of a large database needs disk space for the whole dump.

**Proposed design.** Patch 0001 adds a BLACKHOLE destination to COPY TO that reads and forms all rows but sends no output, which saves some CPU. Patch 0002 adds a pg_dump archive format, -Fb, that uses COPY TO BLACKHOLE in parallel mode and writes nothing. In message #6 the author considers moving the parallel part to pg_amcheck instead of pg_dump.

**User-visible surface.**
- SQL syntax: COPY <table> TO BLACKHOLE, with BLACKHOLE as a new keyword in kwlist.h.
- pg_dump option: new archive format -Fb (blackhole), allowed with -j.
- COPY progress reporting: progress.h is touched, but the patch prose does not say what changes.

**What the author claims.**
- COPY TO BLACKHOLE is about 1.14x faster than COPY to '/dev/null' on a hot cache and about 1.11x on a cold cache, on pgbench_accounts with 10M rows.
- COPY TO BLACKHOLE on a cold cache was faster than verify_heapam() with check_toast => true.
- COPY tests TOAST harder than amcheck: the attached script shows corrupt pglz data that verify_heapam() misses and COPY reports.
- The added check in the COPY row path is one predictable branch, so it costs close to nothing.
- pg_dump -j -Fb gives a parallel logical check of the whole database with no disk space used.

**Thread state.** Ahmed and Miłosz find both COPY and pg_dump the wrong place for this, Hannu Krosing suggests COPY to '/dev/null' or a hashtext query, and Michael Paquier asks for better corruption tools instead and warns about cost in the COPY path. The author agrees pg_dump is wrong and is choosing between withdrawing and a pg_amcheck option; no new version was posted.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-05-20; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. A COPY destination for corruption checks does not belong in COPY

fatal · level 1b, rubric item 6 · doesn't fit what Postgres is about (extension / external tool / already solved), SQL-standard conflict or non-standard syntax, maintenance burden ("who maintains this in 10 years") · `maintenance-fit-1`

Postgres adds BLACKHOLE as a COPY keyword and destination to gain about 14% over COPY TO '/dev/null'. Users then rely on COPY as a supported corruption checker. If amcheck later gains detoast checks, the syntax still has to stay for compatibility.

Message #2 says a COPY destination that moves no data is counter-intuitive. Message #4 notes that '/dev/null' or a hashtext query already does the same work, and in message #5 Michael Paquier asks for better corruption tools instead. In message #6 the author suggests a client-only pg_amcheck option that would not touch core COPY, which fits existing practice better.

Thread: [#2 (ahmed, 2026-07-29)](https://www.postgresql.org/message-id/CAFTkQVK0uaGV%3DtL2EtjWrCY3Rz7FRHLUegn4EdG4VKRN6Yv%3DYQ%40mail.gmail.com), [#4 (Hannu Krosing, 2026-07-30)](https://www.postgresql.org/message-id/CAMT0RQR5_CQ3NsVvmHePMqF2etQoJSeFrTiMcCvXVkMHT8rh%3Dw%40mail.gmail.com), [#5 (Michael Paquier, 2026-07-31)](https://www.postgresql.org/message-id/amwMzxnv4I9VoSPc%40paquier.xyz), [#6 (Jakub Wartak, 2026-08-07)](https://www.postgresql.org/message-id/CAKZiRmz6Nfwj0BS%3DdizC1%3DjRoGt6Yu990fz8T4f0bxPJmEn_SA%40mail.gmail.com)

**Question:** What does COPY TO BLACKHOLE offer that a pg_amcheck option running COPY TO '/dev/null' or a full-row query in parallel cannot offer?

Counter-check: held. Messages #2, #4 and #5 object to the placement, and message #6 offers a client-only pg_amcheck option that leaves core COPY alone. The material gives no reason beyond 11-14% less CPU for a permanent COPY destination.

### 2. COPY TO '/dev/null' already gives the same check

fixable · level 1b, rubric item 12 · doesn't fit what Postgres is about (extension / external tool / already solved), SQL-standard conflict or non-standard syntax, maintenance burden ("who maintains this in 10 years") · `already-exists-1`

Postgres adds a second COPY TO destination that forms every row and discards it, as COPY TO '/dev/null' already does. Every later change to the COPY TO row path has to keep the BLACKHOLE branch working. The only gain is about 11-14% on a check that users run rarely.

Message #1 shows that COPY to '/dev/null' runs the same check, and message #6's pglz reproducer shows it catches the same TOAST corruption that BLACKHOLE targets. Message #4 makes the same point, and the thread never answers it. Users without pg_write_server_files can run COPY TO STDOUT with output sent to /dev/null on the client. Parallel runs need no core change, because N sessions or a pg_amcheck-style client driver can each run COPY to '/dev/null', and message #6 itself names the client-only option "--copy-to-null".

Thread: [#1 (Jakub Wartak, 2026-05-20)](https://www.postgresql.org/message-id/CAKZiRmzorOYZH-CvyN98%2BKz3aGdJW5Kj-MirtM0tmjYO3PtASQ%40mail.gmail.com), [#4 (Hannu Krosing, 2026-07-30)](https://www.postgresql.org/message-id/CAMT0RQR5_CQ3NsVvmHePMqF2etQoJSeFrTiMcCvXVkMHT8rh%3Dw%40mail.gmail.com), [#6 (Jakub Wartak, 2026-08-07)](https://www.postgresql.org/message-id/CAKZiRmz6Nfwj0BS%3DdizC1%3DjRoGt6Yu990fz8T4f0bxPJmEn_SA%40mail.gmail.com)

**Question:** Which check can BLACKHOLE do that COPY TO '/dev/null' or COPY TO STDOUT from a parallel client cannot, beyond about 12% less CPU?

Counter-check: held. Message #6 shows COPY TO '/dev/null' reporting the same pglz error, and parallel client sessions need no core change. The thread does not answer message #4, and the author names a client-only "--copy-to-null" option.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | **yes** | COPY TO BLACKHOLE sends no row data to the client, and the proposal does not say which COPY messages the client receives. |
| catalog changes | no |  |
| ABI / extension API | **yes** | Patch 0001 changes copy.h and parsenodes.h and touches the test_copy_callbacks module. |
| SQL-standard conflict or non-standard syntax | **yes** | Patch 0001 adds BLACKHOLE to kwlist.h and to the COPY TO grammar. |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | **yes** | Patch 0002 adds an archive format, -Fb, that writes nothing and cannot be restored. |
| psql UX conventions | no |  |
| security / privileges | **yes** | The proposal does not say which privilege COPY TO BLACKHOLE needs, while COPY to '/dev/null' needs pg_write_server_files. |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | A new keyword, a new COPY destination and a new pg_dump archive format must be kept working after commit. |
| performance on the hot path | **yes** | The patch adds a branch to the per-row path of every COPY TO, with no benchmark for normal COPY TO. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | COPY to '/dev/null' from parallel client sessions already gives the same check with no core change. |

## 5. Questions the author must answer before anyone reviews code

1. What does COPY TO BLACKHOLE offer that a pg_amcheck option running COPY TO '/dev/null' or COPY TO STDOUT in parallel sessions cannot, beyond 11-14% less CPU?
2. Where do the hot-cache savings come from, given that COPY to '/dev/null' already discards the output?
3. If the pg_amcheck route is chosen, what is the option name, and does it need any change to core COPY?
4. Which corruption does a COPY-based check find that verify_heapam() with check_toast, index checks and pg_catcheck do not cover?
5. What does the new per-row branch cost normal COPY TO STDOUT and COPY TO file, in text and binary format, on narrow and wide tables?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. It needs someone who knows the COPY TO row path and, for the alternative, pg_amcheck. Michael Paquier raised the COPY path cost in message #5, but nobody has offered to take the patch.
- **Liability:** The owner signs for a new SQL keyword and COPY destination that must stay for compatibility, a per-row branch in every COPY TO, and a pg_dump archive format that archives nothing.
- **Author follow-up:** The author replied within a day in message #3 and offered to write a new version, but no new version was posted and withdrawal is under consideration.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** withdraw and resubmit
- **Audience:** niche
- **Maintenance burden:** medium
- **Review cost:** low
- **Why:** The goal of a parallel logical check with no disk use is valid for large databases, and the two patches are small. Reviewers and the author agree the COPY and pg_dump placement is wrong, so a client-only pg_amcheck option needs its own proposal.

## Appendix

### Held up in the counter-check, not listed above

- `standards-compat-1` (fixable, level 2) New COPY destination keyword where an option or existing target would do. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `standards-compat-2` (fixable, level 2) Visible COPY contract for BLACKHOLE is not defined. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `maintenance-fit-2` (fixable, level 2) No measurement of the new branch for ordinary COPY TO. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).

### Refuted in the counter-check

- `maintenance-fit-3` Check coverage depends on COPY internals that nobody guards. Refuted (no failure scenario): The scenario needs a later optimization that nobody has proposed, and it would have to remove the row forming that the commit message gives as the feature's purpose. The same risk applies to any code a later patch might change.
- `demand-1` Gain over the existing COPY to '/dev/null' workaround is small. Refuted (not specific): Duplicate of already-exists-1. Its extra claim of no demand does not hold for the check itself, because message #5 calls pg_dump and COPY a popular way to check data state.
- `demand-2` COPY TO users pay for a check that few people run. Refuted (not specific): Duplicate of maintenance-fit-2, which states the same unmeasured per-row cost and asks for the same benchmark in more detail.

### Review angles

- prior-attempts: 0 objection(s). No precedent found: the index returned no earlier attempt at a discard-only COPY target or an output-free pg_dump format. Message #6 links earlier threads about pg_catcheck reading only the first row of each relation, but the material does not report a rejection reason there that applies to this proposal.
- standards-compat: 2 objection(s). The pg_dump -Fb archive format is not raised here: message #2 objected to it and the author accepted that point in message #6.
- replication-ondisk: 0 objection(s). The proposal adds no persistent or cross-node state. It changes no on-disk format, WAL record, catalog, replication path or extension-facing struct, and the -Fb format writes no archive for pg_restore to read. The open questions are the client messages for COPY TO BLACKHOLE under DestRemote and what progress.h changes, and the prose does not answer either.
- security: 0 objection(s). No security objection: COPY TO BLACKHOLE reads only what COPY TO STDOUT or a SELECT already lets a user with SELECT privilege read, and it writes nothing to the server file system. It also needs less privilege than COPY to '/dev/null', which needs pg_write_server_files. The prose does not say which privilege check BLACKHOLE uses, how it handles row-level security, or what progress reporting exposes, and these need checking in the code.
- maintenance-fit: 3 objection(s). I do not repeat the pg_dump -Fb objection. The author accepted in message #6 that pg_dump is the wrong place for this.
- already-exists: 1 objection(s). The pg_dump -Fb format is not raised, because message #6 already accepts that pg_dump is the wrong place.
- demand: 2 objection(s). The parallel need is real for large databases, but the author already accepts in message #6 that pg_dump is the wrong place. A pg_amcheck option with no core change has no design yet and was not evaluated.

### Precedents retrieved

- P1: [Verify true root on replicas with amcheck](https://commitfest.postgresql.org/patch/2418/) (cf, Returned with feedback, 2020-04-08) - judged not relevant; reason summarised from its thread
- P2: [Allow pg_verify_checksums to specify a database to scan](https://commitfest.postgresql.org/patch/1782/) (cf, Returned with feedback, 2018-10-02) - judged not relevant; reason summarised from its thread
- P3: [amcheck verification for GiST](https://commitfest.postgresql.org/patch/1800/) (cf, Returned with feedback, 2019-11-28) - judged not relevant; reason summarised from its thread
- P4: [Revert "pg_dump: Lock all relations, not just plain tables".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=d3adaabaf7d555ec8bb1d83c43f72e79f1bf0b7d) (revert, Reverted, 2020-11-06) - judged not relevant
- P5: [pg_dump: optimize dumpFunc()](https://commitfest.postgresql.org/patch/5147/) (cf, Withdrawn, 2024-08-02) - judged not relevant; reason summarised from its thread
- P6: [Check for tuplestorestate nullness before dereferencing](https://commitfest.postgresql.org/patch/5705/) (cf, Withdrawn, 2025-12-08) - judged not relevant; reason summarised from its thread
- P7: [Align verify_heapam.c error message offset with test expectations](https://commitfest.postgresql.org/patch/6423/) (cf, Withdrawn, 2026-05-12) - judged not relevant; reason summarised from its thread
- P8: [Check for TupleTableSlot nullness before dereferencing](https://commitfest.postgresql.org/patch/5662/) (cf, Withdrawn, 2025-12-08) - judged not relevant; reason summarised from its thread
- P9: [Improve amcheck to check UNIQUE constraint in btree index](https://commitfest.postgresql.org/patch/2976/) (cf, Returned with feedback, 2021-04-08) - judged not relevant; reason summarised from its thread
- P10: [Improve pg_dump dumping publication tables](https://commitfest.postgresql.org/patch/2728/) (cf, Withdrawn, 2021-01-15) - judged not relevant
- P11: [Add test module for verifying backtrace functionality](https://commitfest.postgresql.org/patch/4823/) (cf, Withdrawn, 2024-08-24) - judged not relevant
- P12: [pg_dump --where option](https://commitfest.postgresql.org/patch/2600/) (cf, Returned with feedback, 2020-09-30) - judged not relevant
- P13: [pg_dump new feature: exporting functions only](https://commitfest.postgresql.org/patch/3051/) (cf, Returned with feedback, 2022-04-07) - judged not relevant
- P14: [pg_dump -C Grant statement on create database](https://commitfest.postgresql.org/patch/724/) (cf, Returned with feedback, 2016-10-03) - judged not relevant
- P15: [Fix bug of COPY (on_error set_null)](https://commitfest.postgresql.org/patch/6774/) (cf, Withdrawn, 2026-05-19) - judged not relevant
- P16: [Ignore attributes of tables part of extensions in pg_dump](https://commitfest.postgresql.org/patch/164/) (cf, Rejected, 2015-02-23) - judged not relevant
- P17: [Authorization facility for multiple password verifiers + SCRAM-SHA1](https://commitfest.postgresql.org/patch/320/) (cf, Returned with feedback, 2015-12-01) - judged not relevant
- P18: [Improving test coverage of extensions with pg_dump](https://commitfest.postgresql.org/patch/187/) (cf, Returned with feedback, 2015-12-02) - judged not relevant
- P19: [Allow COPY "text" to output a header and add header matching mode to COPY FROM](https://commitfest.postgresql.org/patch/2504/) (cf, Returned with feedback, 2020-09-29) - judged not relevant
- P20: [Code checks for App Devs, using new options for transaction behavior](https://commitfest.postgresql.org/patch/3970/) (cf, Withdrawn, 2023-03-24) - judged not relevant
- P21: [pg_replslotdata - a tool for displaying replication slot information](https://commitfest.postgresql.org/patch/3425/) (cf, Rejected, 2022-01-17) - judged not relevant
- P22: [Revert "Fix corner case with PGP decompression in pgcrypto"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=38f60f174e3279069b5547b5f4015eb4a8492037) (revert, Reverted, 2020-07-22) - judged not relevant
- P23: [Allow COPY's 'text' format to output a header](https://commitfest.postgresql.org/patch/1629/) (cf, Returned with feedback, 2018-10-02) - judged not relevant
- P24: [Discarding DISCARD ALL](https://commitfest.postgresql.org/patch/2919/) (cf, Withdrawn, 2021-06-21) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 15; wall time: 139 s; cost reported by the CLI: $1.50
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 5 from cache, 3 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 89 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:03:49.818Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

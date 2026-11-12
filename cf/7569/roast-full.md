# Design pre-review: Fix trigger memory leak

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7569](https://commitfest.postgresql.org/patch/7569/) (Needs review); authors: Paul Jungwirth (pjungwir)
- Thread: [Fix trigger memory leak](https://www.postgresql.org/message-id/flat/CA%2BrenyVvS8K0YypfwYfRNT8-8OHoDKPPNv%3DpcSC0RdZx%2BuV__w%40mail.gmail.com), started 2026-10-31
- Messages read: 5 of 5
- Patch prose read: v2-0001-Free-the-tuple-fetched-for-BEFORE-INSTEAD-OF-row-.patch, v1-0001-Free-the-tuple-fetched-for-BEFORE-INSTEAD-OF-row-.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium)

No objection survived at levels 1a, 1b or 2. The problem is stated, and message #3 measures a statement-lifespan leak of about 104 MiB for 50k rows through views and partitioned tables. The fix is confined to trigger.c and adds no user-visible surface. Confidence is medium because the safety of the added free depends on code that was not read.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, it could not be checked that the freed tuple is never referenced afterwards, or that the free happens only when the slot made a copy. The four functions named in the commit message were not compared with other trigger paths.

## The proposal, restated

**Problem.** ExecBRInsertTriggers, ExecBRUpdateTriggers, ExecIRInsertTriggers and ExecIRUpdateTriggers leak a copied HeapTuple when the slot is virtual and the trigger returns NEW unchanged (or NULL in ExecIRUpdateTriggers). Users hit it with triggers on views and with BEFORE INSERT triggers on partitioned tables routing to same-layout partitions. Memory grows per row for the whole statement.

**Proposed design.** The patch frees the tuple fetched from the virtual slot when the trigger hands back the same tuple. The change is confined to src/backend/commands/trigger.c and is meant for back-patching to 14.

**What the author claims.**
- The leak exists on master and goes back to v14 at least.
- A reviewer measured ExecutorState at about 104 MiB on master for 50k rows with a 1 kB column, flat with v1.
- The partitioned-table case drops from about 105 MiB to about 544 KiB with the patch, per a reviewer.
- Assigning to NEW in PL/pgSQL, even NEW.id := NEW.id, avoids the leak on master.

**Thread state.** Chao Li and Surya Poondla reviewed and approved; v2 fixed a commit message typo and added the partitioned-table case. No committer has replied, and nothing is open.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-10-31; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

No objection held up strongly enough in the counter-check to list here.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | The patch frees a tuple that a C trigger function in an extension may have received or returned. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The commit message asks for a back-patch through 14. |
| maintenance burden ("who maintains this in 10 years") | no |  |
| performance on the hot path | **yes** | The patch adds a per-row free in the BEFORE and INSTEAD OF row trigger paths. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Can a C trigger function keep or return a pointer into the tuple that the patch now frees, and if so, what guards against use after free?
2. Does the patch apply to every branch back to 14, or do some branches need a different version?
3. Do other trigger paths, such as delete triggers or triggers on foreign tables, leak the same copied tuple?
4. Does the patch add a regression test, and if not, how would a later change that brings the leak back be noticed?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows executor tuple slots and trigger.c tuple ownership would have to own it. No committer has replied in the visible thread; the commit message credits Andres Freund only as reporter.
- **Liability:** The owner signs for the added free being safe in six supported branches, including for C trigger functions in extensions. A mistake there is a use after free in released minor versions.
- **Author follow-up:** The author posted v2 within days, fixing the commit message and adding the partitioned-table case from message #3.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The leak reaches users of INSTEAD OF triggers on views and BEFORE INSERT triggers on partitioned tables, at about 104 MiB per 50k rows in message #3. The patch is about 3 KiB in one file, with two reviewers approving.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. [P1] was rejected because it showed no query-lifespan leak, while message #3 here measures ExecutorState growth of about 104 MiB over one statement, so that reason does not apply. Message #1 says Andres reported the leak while reviewing FOR PORTION OF and does not report any earlier rejected attempt at this fix.
- standards-compat: 0 objection(s). The patch adds no syntax, GUC, catalog object or psql change, and it does not change query results or what pg_dump emits. Back-patching to 14 changes only memory use. [P1] does not apply, because message #3 shows a statement-lifespan leak of about 100 MiB per 50k rows. Whether freeing the tuple is safe for C trigger functions that keep the returned pointer is a safety question for another angle.
- replication-ondisk: 0 objection(s). The patch only frees a backend-local HeapTuple in trigger.c. It touches no on-disk format, WAL, catalog, wire protocol or replication state. A back-patch could break ABI only if it changed exported signatures or structs, and the brief describes a change inside function bodies. Whether ownership of the returned tuple stays correct for callers and extensions cannot be checked without the code.
- security: 0 objection(s). The patch adds no privilege, no new surface and no new exposure. It frees a per-row copy that today grows for the whole statement, so it reduces memory use that an ordinary user can trigger. Whether the freed tuple could still be referenced is a safety question that needs the code to answer, and it is outside the security angle. [P1] does not apply, because message #3 shows a statement-lifespan leak.
- maintenance-fit: 0 objection(s). From the maintenance and fit angle there is nothing to send. This is a back-patchable bugfix confined to trigger.c, with no new API, GUC, catalog or per-row cost for non-users. [P1] does not apply, because its reason required a query-lifespan leak and message #3 shows one of about 104 MiB per statement. Whether other paths need the same free, such as FOR PORTION OF or delete and foreign-table triggers, and whether a per-tuple memory context would be the more durable fix, are review questions. Neither carries a concrete failure scenario against this patch.
- already-exists: 0 objection(s). This is a bug fix for a statement-lifetime leak in ExecutorState, which nothing in Postgres or in an extension can replace. The PL/pgSQL workaround of assigning to NEW is a user-side trick, so it does not meet the need. [P1] does not apply because its reason was per-tuple context reset, and message #3 shows query-lifespan growth.
- demand: 0 objection(s). This is a bug fix for a statement-lifespan memory leak, so the demand test does not apply in the usual way. Message #3 shows about 104 MiB of ExecutorState growth for 50k rows through common paths: INSTEAD OF triggers on views, and BEFORE INSERT triggers on partitioned tables. Users cannot opt out of this cost. [P1] does not fit, because its recorded reason was the lack of a query-lifespan leak, and this thread shows one.

### Precedents retrieved

- P1: [Leaking memory in text_overlay function](https://commitfest.postgresql.org/patch/658/) (cf, Rejected, 2016-08-04) - used in the review; reason summarised from its thread
- P2: [pg_upgrade: fix memory leak in SLRU I/O code](https://commitfest.postgresql.org/patch/6462/) (cf, Withdrawn, 2026-02-05) - judged not relevant; reason summarised from its thread
- P3: [Re: BUG #18741: Detaching a partition referencing a partitioned table fails with a trigger-related error](https://commitfest.postgresql.org/patch/5484/) (cf, Withdrawn, 2025-02-18) - judged not relevant; reason summarised from its thread
- P4: [BUG Segfault triggered by reentrancy in ri_triggers.c](https://commitfest.postgresql.org/patch/6805/) (cf, Withdrawn, 2026-05-29) - judged not relevant; reason summarised from its thread
- P5: [Row Level Security, leakproof-ness, and performance implications](https://commitfest.postgresql.org/patch/2038/) (cf, Rejected, 2019-03-19) - judged not relevant; reason summarised from its thread
- P6: [Revert function to get memory context stats for processes](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=fb844b9f06568aa29303122ac0bb7b4c3cce2542) (revert, Reverted, 2025-05-23) - judged not relevant
- P7: [Change tuple table slot for Unique node to "virtual"](https://commitfest.postgresql.org/patch/4531/) (cf, Withdrawn, 2024-03-28) - judged not relevant; reason summarised from its thread
- P8: [Code comments: why some text-handling functions are leakproof](https://commitfest.postgresql.org/patch/3510/) (cf, Returned with feedback, 2022-03-28) - judged not relevant; reason summarised from its thread
- P9: [BUG Segfault triggered by reentrancy in ri_triggers.c](https://commitfest.postgresql.org/patch/6815/) (cf, Withdrawn, 2026-06-01) - judged not relevant; reason summarised from its thread
- P10: [Triggers on Materialized Views](https://commitfest.postgresql.org/patch/1948/) (cf, Returned with feedback, 2019-04-01) - judged not relevant
- P11: [Allow WHEN in INSTEAD OF triggers](https://commitfest.postgresql.org/patch/2398/) (cf, Withdrawn, 2020-02-26) - judged not relevant
- P12: [Triggers on materialized views](https://commitfest.postgresql.org/patch/1953/) (cf, Withdrawn, 2019-01-04) - judged not relevant
- P13: [Allow INSTEAD OF DELETE triggers to modify the tuple for RETURNING](https://commitfest.postgresql.org/patch/1237/) (cf, Returned with feedback, 2017-09-19) - judged not relevant
- P14: [doc: virtual envs with Pl/Python](https://commitfest.postgresql.org/patch/5308/) (cf, Withdrawn, 2025-02-10) - judged not relevant
- P15: [Revert "Change SHA2 implementation based on OpenSSL to use EVP digest routines"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=fe0a1dc52c7332a65b44db8e8408a5fd1d8fc8fb) (revert, Reverted, 2020-09-29) - judged not relevant
- P16: [foreign key on virtual generated column](https://commitfest.postgresql.org/patch/5782/) (cf, Withdrawn, 2026-09-23) - judged not relevant
- P17: [partial heap only tuples](https://commitfest.postgresql.org/patch/2981/) (cf, Returned with feedback, 2021-11-04) - judged not relevant
- P18: [Reducing memory usage of creating foreign key between partitioned tables](https://commitfest.postgresql.org/patch/2330/) (cf, Rejected, 2020-09-12) - judged not relevant
- P19: [Infinite Autovacuum loop caused by failing virtual generated column expression](https://commitfest.postgresql.org/patch/7100/) (cf, Withdrawn, 2026-08-19) - judged not relevant
- P20: [Revert analyze support for partitioned tables](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6f8127b7390119c21479f5ce495b7d2168930e82) (revert, Reverted, 2021-08-16) - judged not relevant
- P21: [Revert: Allow table AM tuple_insert() method to return the different slot](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=8dd0bb84da7d56a9e41241b26bfbf6b79644d574) (revert, Reverted, 2024-04-11) - judged not relevant
- P22: [Prefetch the next tuple's memory during seqscans](https://commitfest.postgresql.org/patch/3978/) (cf, Withdrawn, 2024-01-22) - judged not relevant
- P23: [Revert "Avoid creation of the free space map for small heap relations."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a23676503b746b7f1588cd2ab0c60411032d32da) (revert, Reverted, 2019-01-28) - judged not relevant
- P24: [Allow INSTEAD OF DELETE triggers to modify the tuple for RETURNING](https://commitfest.postgresql.org/patch/4617/) (cf, Returned with feedback, 2024-10-07) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 13; wall time: 59 s; cost reported by the CLI: $0.67
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 5 from cache, 3 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 41 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T04:11:27.578Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

# Design pre-review: Fix plpgsql's R/W optimization for simple expressions run via SPI

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7476](https://commitfest.postgresql.org/patch/7476/) (Needs review); authors: Muzzammil Sarwar (muzzammil242)
- Thread: [\[PATCH\] Fix plpgsql's R/W optimization for simple expressions run via SPI](https://www.postgresql.org/message-id/flat/CABUcsRRoJg_d%3D-%3DFGFRk2ZqMcDjXG_Yhpn9wuzfJXWa2AvnrpQ%40mail.gmail.com), started 2026-10-18
- Messages read: 1 of 1
- Patch prose read: v1-0001-Fix-plpgsql-s-R-W-optimization-for-simple-express.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: low)

No objection survived at levels 1a, 1b or 2. Message #1 states the bug, gives a reproducer that returns a wrong result on 18 and master, and explains the cause. The fix narrows the read/write optimization to simple expressions and adds no user-visible surface. Confidence is low because only one message is visible and the code was not read.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so the review could not check that the callback is installed only for the simple expression's own tree, or that the "inplace" method is safe on the SPI path. The memory-safety side of the second reference was also not checked.

## The proposal, restated

**Problem.** Since 18, a PL/pgSQL assignment can give a wrong result in builds without assertions, and trips an assertion in builds with them. It happens when the simple expression is busy, so PL/pgSQL runs it through SPI, and SPI has replanned it. The new plan's Params get the read/write "transfer" treatment chosen for a different Param.

**Proposed design.** plpgsql_param_compile() installs the plpgsql_param_eval_var_check() callback only while compiling the simple expression's own tree. Expressions run through SPI (exec_run_select) get read-only references, as non-simple expressions always do. The patch adds regression tests to plpgsql_array.sql.

**What the author claims.**
- Master and 18 without assertions return a wrong result for the attached script; 17 and patched master return the correct one.
- Assertion-enabled builds of master, 19 and 18 fail Assert("param == expr->expr_rw_param") or Assert(context.count > 0).
- The bug needs neither force_generic_plan nor recursion; default plan caching or an error caught in an exception block also triggers it.
- The cost is one copy of the value at SPI levels only, in a path that flattens its result anyway.
- check-world passes without TAP tests, PL/pgSQL tests pass under Valgrind, and the patch cherry-picks cleanly to 19 and 18.

**Thread state.** Only the first message is visible. Nobody has replied, and the CommitFest entry is in Needs review, targeting back-patch to 18.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-10-18; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

No objection held up strongly enough in the counter-check to list here.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The author asks for a back-patch to 18 and 19, and the CommitFest entry targets stable branches. |
| maintenance burden ("who maintains this in 10 years") | **yes** | The patch changes when pl_exec.c installs plpgsql_param_eval_var_check(), a rule that future changes to the read/write optimization must keep. |
| performance on the hot path | **yes** | Simple expressions run through SPI now copy the variable's value, where 18 could transfer it. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Does the "inplace" method, or any other user of expr_rwopt and expr_rw_param, have the same exposure when the expression runs through SPI after a replan?
2. How much slower is an assignment to a large array when the expression stays busy for the rest of a transaction after a caught error?
3. Do the TAP tests pass on master, 19 and 18 with the patch applied?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows PL/pgSQL expression execution in pl_exec.c, the expanded-object read/write rules, and how SPI replans cached plans.
- **Liability:** The owner signs for the correctness of the read/write optimization on 18, 19 and master for the life of those branches, including any slowdown on the SPI path in a minor release.
- **Author follow-up:** No sign either way; the thread has one message, with a reproducer, regression tests and a check on the cherry-pick to 19 and 18.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs champion
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch fixes a wrong result in released 18 that needs a busy simple expression and a replan, so few users hit it, but the result is silently wrong. The patch is 6.4 KB in one source file plus tests, and nobody has replied in the visible thread.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent matches. [P1] is in the same area, since it added the PARAM_EXTERN callback machinery that plpgsql_param_compile() builds on. Its recorded text describes what that commit does, not a reason for rejection that applies to this bug fix. Message #1 points only to the thread that introduced the "transfer" method, which is the origin of the bug and not an earlier attempt at this fix.
- standards-compat: 0 objection(s). The patch changes no SQL syntax, GUC, catalog, pg_dump output, psql behaviour or client-visible protocol. The only behaviour change makes 18 and later return the results 17 returned, so there is nothing to object to from the compatibility angle. Back-patching to 18 adds no user-facing contract.
- replication-ondisk: 0 objection(s). The fix changes only in-memory PL/pgSQL behaviour in pl_exec.c, with regression tests. It touches no on-disk format, WAL, replication, catalog or wire protocol. The commit message mentions no struct or exported-function change, so back-patching to 18 shows no ABI risk; this is provisional without the code.
- security: 0 objection(s). The fix narrows when PL/pgSQL hands a read/write expanded value to an expression; it adds no privilege, role, GUC, view or interface. Expressions run through SPI get read-only references, which is the safer default. A memory-safety angle (a second reference reading a freed or moved value) could not be checked without the code; the reported symptom is a null read.
- maintenance-fit: 0 objection(s). This is a back-patchable bug fix that narrows an 18-era optimization back to the case its own comment allows: simple expressions only. Expressions run through SPI fall back to read-only references, which is how 17 handled them and how non-simple expressions are handled now. The extra copy lands only on the SPI path, which flattens its result anyway. The patch adds no new API, GUC or catalog state, and puts no cost on users who never take that path. [P1] concerns a different, larger redesign of Param execution and does not apply. Without the code, two points remain unchecked: whether the "inplace" method has the same SPI-path exposure, and whether long transactions that follow a caught error copy large arrays often.
- already-exists: 0 objection(s). This is a bug fix for a wrong result in the 18 transfer optimization, inside pl_exec.c. No existing feature, extension or setting covers it, and it adds no new mechanism. Turning the optimization off would mean reverting it, which is the larger change. Only the code can show whether the fix is correct.
- demand: 0 objection(s). This is a wrong-result bug fix in released 18, with a reproducer in message #1 that works under default plan caching, so demand is not in question. The cost is one extra copy on the SPI fallback path only, and it falls on the affected expressions. Precedent [P1] concerns the PARAM_EXTERN callback design and does not bear on demand.

### Precedents retrieved

- P1: [Rearrange execution of PARAM_EXTERN Params for plpgsql's benefit.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6719b238e8f0ea83c39d2b1728c7670cdaa34e06) (revert, Partially reverted, 2017-12-21) - used in the review
- P2: [Revert inappropriate weakening of an Assert in plpgsql.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=16a3ae504e2030583851c8f45cbed14b0c546d6e) (revert, Reverted, 2025-03-21) - judged not relevant
- P3: [Expand applicability of aggregate's sortop optimization](https://commitfest.postgresql.org/patch/4980/) (cf, Returned with feedback, 2025-07-08) - judged not relevant; reason summarised from its thread
- P4: [new plpgsql.extra_errors check - strict_expr_check](https://commitfest.postgresql.org/patch/5044/) (cf, Withdrawn, 2026-03-24) - judged not relevant; reason summarised from its thread
- P5: [When deleting the plpgsql function, release the CachedPlan of the function](https://commitfest.postgresql.org/patch/5974/) (cf, Withdrawn, 2025-08-19) - judged not relevant; reason summarised from its thread
- P6: [When deleting the plpgsql function, release the CachedPlan of the function](https://commitfest.postgresql.org/patch/5975/) (cf, Withdrawn, 2025-12-31) - judged not relevant; reason summarised from its thread
- P7: [plpgsql - possibility to get element or array type for referenced variable type](https://commitfest.postgresql.org/patch/412/) (cf, Rejected, 2016-03-16) - judged not relevant; reason summarised from its thread
- P8: [Expand the use of check_canonical_path() for more GUCs](https://commitfest.postgresql.org/patch/2564/) (cf, Returned with feedback, 2020-07-10) - judged not relevant; reason summarised from its thread
- P9: [Various small unrelated doc patches: plpgsql, schemas, permissions, oidvector](https://commitfest.postgresql.org/patch/4581/) (cf, Returned with feedback, 2024-10-07) - judged not relevant; reason summarised from its thread
- P10: [use regular expressions to simplify less_greater and not_equals](https://commitfest.postgresql.org/patch/3281/) (cf, Rejected, 2021-09-05) - judged not relevant; reason summarised from its thread
- P11: [iteration over record in plpgsql](https://commitfest.postgresql.org/patch/124/) (cf, Returned with feedback, 2015-07-07) - judged not relevant
- P12: [Don't allow to EXECUTE empty queries (fixes Assert violation)](https://commitfest.postgresql.org/patch/5918/) (cf, Rejected, 2025-07-17) - judged not relevant
- P13: [Revert the logic for expanding AND/OR conditions in pred_test() to what it was in 7.4, and add some comments explaining why it has to be this way.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=4e89bae704f79af8b2b6ac3d43ea963c22df1947) (revert, Reverted, 2005-03-01) - judged not relevant
- P14: [possibility to write window function in PL languages (like PLpgSQL)](https://commitfest.postgresql.org/patch/2697/) (cf, Withdrawn, 2021-01-27) - judged not relevant
- P15: [Surround CheckRelation\[Oid\]LockedByMe() with USE_ASSERT_CHECKING](https://commitfest.postgresql.org/patch/5096/) (cf, Withdrawn, 2024-07-01) - judged not relevant
- P16: [Simplify some RI checks to reduce SPI overhead](https://commitfest.postgresql.org/patch/2948/) (cf, Returned with feedback, 2022-04-09) - judged not relevant
- P17: [ALTER SYSTEM READ { ONLY | WRITE }](https://commitfest.postgresql.org/patch/2602/) (cf, Withdrawn, 2022-01-10) - judged not relevant
- P18: [Optimize planner memory consumption for huge arrays](https://commitfest.postgresql.org/patch/4825/) (cf, Returned with feedback, 2024-10-07) - judged not relevant
- P19: [Refactor compile-time assertion checks for C/C++](https://commitfest.postgresql.org/patch/2437/) (cf, Returned with feedback, 2020-03-23) - judged not relevant
- P20: [Fix slotsync worker busy loop causing repeated log messages](https://commitfest.postgresql.org/patch/6546/) (cf, Withdrawn, 2026-05-01) - judged not relevant
- P21: [Revert "Cannot use WL_SOCKET_WRITEABLE without WL_SOCKET_READABLE."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a58db3aa10e62e4228aa409ba006014fa07a8ca2) (revert, Reverted, 2020-12-15) - judged not relevant
- P22: [SLOPE - Planner optimizations on monotonic expressions.](https://commitfest.postgresql.org/patch/6653/) (cf, Withdrawn, 2026-07-20) - judged not relevant
- P23: [Revert "Rename routines for write/read of pgstats file"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=381b3cfe2be02aa114ef7a28c4ec24feaf377c63) (revert, Reverted, 2026-07-14) - judged not relevant
- P24: [Inconsistency in owner assignment between INDEX and STATISTICS](https://commitfest.postgresql.org/patch/6571/) (cf, Withdrawn, 2026-03-11) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 12; wall time: 61 s; cost reported by the CLI: $0.70
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 41 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:59:36.341Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

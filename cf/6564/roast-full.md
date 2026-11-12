# Design pre-review: Reject ADD CONSTRAINT NOT NULL if name mismatches existing domain not-null constraint

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6564](https://commitfest.postgresql.org/patch/6564/) (Needs review); authors: Jian He (stayreal)
- Thread: [Reject ADD CONSTRAINT NOT NULL if name mismatches existing domain not-null constraint](https://www.postgresql.org/message-id/flat/CACJufxEcTx2U8tQTcJ2g490wzd%3DjQgJNDpNb2L%2Bf3gr9J%3DcpNA%40mail.gmail.com), started 2026-03-01
- Messages read: 4 of 4
- Patch prose read: v1-0001-Reject-ADD-CONSTRAINT-NOT-NULL-if-name-mismatches-existing-domain.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

No surviving objection questions the goal on master: v1 extends the table rule from commit 96e2af6 to domains, and message #3 confirms the one-constraint rule. standards-compat-1 is about scope, because the entry targets stable branches and no wrong result from the silent success is shown. standards-compat-2 applies only to the v2 NOTICE variant, which departs from the table behaviour. Both points are fixable and need a committer's decision on ERROR versus NOTICE and on back branches.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so this review cannot confirm that v1 leaves unnamed `add not null` and `set not null` unchanged. Message #4 was judged only from the brief's summary of it.

## The proposal, restated

**Problem.** When a domain already has a not-null constraint, `alter domain ... add constraint <name> not null` with a different name succeeds silently today. The author wants the same check that commit 96e2af6 added for tables, so users are not confused about which constraint object enforces the rule.

**Proposed design.** v1 makes `AlterDomainAddConstraint()` raise an error when the requested name differs from the existing domain not-null constraint name. Unnamed `add not null` and `set not null` stay allowed when repeated. A reviewer's v2 instead keeps the silent success and emits a NOTICE in both `AlterDomainAddConstraint()` and `AlterDomainNotNull()`.

**User-visible surface.**
- v1: new ERROR "cannot create not-null constraint ... for domain ..." with a DETAIL naming the existing constraint, raised by `alter domain ... add constraint <name> not null`.
- v2: new NOTICE "Domain ... is already NOT NULL, skipping" for repeated `add not null`, `add constraint ... not null` and `set not null`.
- v2: new NOTICE "Domain ... is already NULL, skipping" for repeated `drop not null`.

**What the author claims.**
- The change aligns domain not-null handling with not-null constraints on tables after commit 96e2af6.
- v1 reuses the error code and a similar message from commit 96e2af6, according to Chao Li.
- `create domain` needs no change because it already rejects multiple not-null constraints.
- Zizhuan Liu claims an ERROR breaks backward compatibility for applications and transaction blocks that rely on silent success.

**Thread state.** Chao Li and Álvaro Herrera agreed that order-dependent behaviour (unnamed first, then named, fails) is expected. Zizhuan Liu then proposed a NOTICE-only v2; no one has answered it, so ERROR versus NOTICE is still open.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-03-01; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. New ERROR is proposed for stable branches as a bugfix

fixable · level 1b, rubric item 6 · backpatchability, SQL-standard conflict or non-standard syntax · `standards-compat-1`

A migration tool re-runs `alter domain d add constraint d_nn not null` on a domain whose not-null constraint has a different name, often the default one. After a minor upgrade the statement raises an ERROR and the transaction aborts. The operator finds out when a deployment that worked last week fails.

The CommitFest entry is tagged Bugfix and targets stable branches. v1 turns a statement that succeeds today into an ERROR. Message #4 raises this compatibility cost, and no one has answered it. The thread does not show that the current silent success causes wrong results or corruption, which is what would justify a behaviour change in a minor release.

Thread: [#1 (jian he, 2026-03-01)](https://www.postgresql.org/message-id/CACJufxEcTx2U8tQTcJ2g490wzd%3DjQgJNDpNb2L%2Bf3gr9J%3DcpNA%40mail.gmail.com), [#4 (ZizhuanLiu X-MAN, 2026-06-27)](https://www.postgresql.org/message-id/tencent_84E11BD212851AB351001EA90D8D96782205%40qq.com)

**Question:** What wrong result does the current silent success cause that justifies a new ERROR in minor releases, rather than a master-only change?

Counter-check: held. The CommitFest entry targets stable, and v1 turns a statement that succeeds today into an ERROR. Message #4 raises the compatibility cost, and the thread shows no answer and no wrong result from the silent success.

### 2. v2 NOTICE design makes domains diverge from tables

fixable · level 2, rubric item 12 · design, SQL-standard conflict or non-standard syntax, psql UX conventions · `standards-compat-2`

A user runs `alter domain d add constraint other_name not null` on a domain that already has a not-null constraint. Under v2 it succeeds with a NOTICE, but the matching table command errors after 96e2af6. Scripts that repeat `set not null` and `drop not null` also start printing NOTICEs on every run.

The stated goal in message #1 is to align domains with tables. v2 in message #4 keeps the name mismatch that 96e2af6 rejects for tables. It also adds NOTICEs to `set not null` and `drop not null`, and the thread does not show that the table forms emit them. That is a second, larger departure from existing practice, and v2 gives no reason for it beyond preserving silent success.

Thread: [#1 (jian he, 2026-03-01)](https://www.postgresql.org/message-id/CACJufxEcTx2U8tQTcJ2g490wzd%3DjQgJNDpNb2L%2Bf3gr9J%3DcpNA%40mail.gmail.com), [#4 (ZizhuanLiu X-MAN, 2026-06-27)](https://www.postgresql.org/message-id/tencent_84E11BD212851AB351001EA90D8D96782205%40qq.com)

**Question:** If domains should match tables, why should a mismatched name be accepted with a NOTICE, and do repeated table `set not null` commands emit a NOTICE today?

Counter-check: held. Message #1 states alignment with tables as the goal, and v2 accepts the mismatched name that 96e2af6 rejects for tables. Repeated `alter table ... set not null` emits no NOTICE today, so the added NOTICEs are a second departure that v2 does not justify.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | **yes** | see objection standards-compat-1: New ERROR is proposed for stable branches as a bugfix |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | **yes** | see objection standards-compat-2: v2 NOTICE design makes domains diverge from tables |
| security / privileges | no |  |
| backpatchability | **yes** | The CommitFest entry is tagged Bugfix with target version stable, and v1 turns a statement that succeeds today into an ERROR. |
| maintenance burden ("who maintains this in 10 years") | no |  |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. What wrong result does the current silent success cause that justifies a new ERROR in minor releases, compared with a master-only change?
2. Which behaviour is proposed for commit: the v1 ERROR that matches tables after 96e2af6, or the v2 NOTICE that keeps the mismatched name accepted?
3. If v2 is preferred, why should repeated `set not null` and `drop not null` on a domain emit a NOTICE when the table forms do not?
4. Does the `alter domain` documentation page need to state that a named not-null constraint must match the existing constraint name?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Álvaro Herrera answered the order-dependence question in message #3; the owner needs to know the not-null constraint work for tables and domains in typecmds.c.
- **Liability:** A new user-visible ERROR in `alter domain ... add constraint ... not null`; if back-patched, the owner also answers for scripts that break in a minor release.
- **Author follow-up:** No sign either way; the visible thread shows no author reply after message #1, and the v2 proposal is unanswered.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The v1 patch is 4.5 KB, touches only typecmds.c and the domain regression test, and follows an accepted table rule. Few users add a named not-null constraint to a domain that already has one, and the open points are the back-branch scope and ERROR versus NOTICE.

## Appendix

### Refuted in the counter-check

- `maintenance-fit-1` Back-patching a new error changes minor-release behaviour for working scripts. Refuted (not specific): Duplicate of standards-compat-1. It gives the same scenario and asks the author the same question about a new ERROR in minor releases.
- `demand-1` Back-branch target lacks any user report of harm. Refuted (not specific): Duplicate of standards-compat-1. The missing field report is the same gap as the missing wrong result that standards-compat-1 already asks the author to show.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. [P4] was withdrawn only because it was merged into another patch, and the thread records no objection to its content. [P8] concerned alter constraint changing foreign key actions. Its state-dependent syntax concern resembles the order dependence raised in message #2, but message #3 settled that point here, and the same rule already applies to tables. The thread cites no earlier discussion of this idea apart from the tables commit it follows.
- standards-compat: 2 objection(s). The order-dependent failure (unnamed first, then named) was settled in message #3, so it is not raised. pg_dump round-trip is unaffected because a domain can hold only one not-null constraint.
- replication-ondisk: 0 objection(s). Neither v1 nor v2 changes persistent or cross-node state: no new catalog column, WAL record, on-disk format, wire message or exported struct. Both only change whether `alter domain` raises an ERROR or a NOTICE before any catalog write. Back-branch acceptance of a new ERROR is a behaviour question for another angle.
- security: 0 objection(s). The proposal has no privilege or exposure aspect. alter domain still requires domain ownership, and the new ERROR (v1) or NOTICE (v2) shows only a constraint name or the domain name, which the owner can already read in pg_constraint.
- maintenance-fit: 1 objection(s). Apart from the back-branch question, maintenance cost is small. The change is local to typecmds.c, adds nothing to a hot path, follows the existing table rule, and a committer who knows not-null constraints took part in message #3.
- already-exists: 0 objection(s). No existing feature, contrib module or extension does this name check for domain not-null constraints, so this angle has nothing to raise. The proposal adds no new mechanism; it extends the table-side check from commit 96e2af6 to domains, so ERROR versus NOTICE and back-branch scope belong to other angles.
- demand: 1 objection(s). For master, the change is a small consistency fix that follows an accepted table precedent, and its cost falls only on users of the command, so demand is not a strong objection there.

### Precedents retrieved

- P1: [Fix pg_get_constraintdef for NOT NULL constraints on domains](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=09d09d4297b9acbc2848ec35e8bf030d6c1fae18) (revert, Partially reverted, 2024-11-27) - judged not relevant
- P2: [Revert structural changes to not-null constraints](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6f8bb7c1e9610dd7af20cdaf74c4ff6e6d678d44) (revert, Reverted, 2024-05-13) - judged not relevant
- P3: [Revert "Catalog NOT NULL constraints" and fallout](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9ce04b50e120275afbc03ca0b80839dde3da8308) (revert, Reverted, 2023-04-12) - judged not relevant
- P4: [clean up create|alter domain stmt incompatiable constraint error case and add regression test](https://commitfest.postgresql.org/patch/5385/) (cf, Withdrawn, 2024-12-09) - used in the review; reason summarised from its thread
- P5: [Change a constraint's index - ALTER TABLE ... ALTER CONSTRAINT ... USING INDEX ...](https://commitfest.postgresql.org/patch/2635/) (cf, Returned with feedback, 2020-09-30) - judged not relevant; reason summarised from its thread
- P6: [Allow values to be cast to domain types](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Domains) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P7: [Allow functions defined as casts to domains to be called during casting](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Domains) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P8: [Add support for ON UPDATE/DELETE actions on ALTER CONSTRAINT](https://commitfest.postgresql.org/patch/1533/) (cf, Returned with feedback, 2021-04-08) - used in the review; reason summarised from its thread
- P9: [Detect the null type correctly in setop case by peeking the later exprs.](https://commitfest.postgresql.org/patch/2653/) (cf, Withdrawn, 2020-07-23) - judged not relevant; reason summarised from its thread
- P10: [Prevent ALTER TABLE DROP NOT NULL on child tables if parent column has it](https://commitfest.postgresql.org/patch/646/) (cf, Rejected, 2016-06-16) - judged not relevant; reason summarised from its thread
- P11: [Remove redundant NULL check in clause_selectivity_ext()](https://commitfest.postgresql.org/patch/5188/) (cf, Rejected, 2024-09-30) - judged not relevant; reason summarised from its thread
- P12: [Skip collecting decoded changes of already-aborted transactions](https://commitfest.postgresql.org/patch/4436/) (cf, Returned with feedback, 2024-02-01) - judged not relevant; reason summarised from its thread
- P13: [Revert unnecessary check for NULL](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=807ee417e562c355360e891f415e6e8e6e4c40ed) (revert, Reverted, 2025-08-22) - judged not relevant
- P14: [Make domains work better with polymorphic functions](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Domains) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P15: [Revert "Adjust contrib/sepgsql regression test expected outputs."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=21e6bf4ddd045bc718c99ae551f25957ae91cb4d) (revert, Reverted, 2023-04-12) - judged not relevant
- P16: [Warning for Datatype mismatch between subscriber table and publisher table for logical replication](https://commitfest.postgresql.org/patch/5648/) (cf, Returned with feedback, 2025-05-13) - judged not relevant; reason summarised from its thread
- P17: [Remove redundant initializations](https://commitfest.postgresql.org/patch/3229/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P18: [ALTER TABLE: emit NOTICE when ONLY is omitted but no partition recursion occurs](https://commitfest.postgresql.org/patch/6424/) (cf, Withdrawn, 2026-01-22) - judged not relevant
- P19: [notice lock waits](https://commitfest.postgresql.org/patch/702/) (cf, Returned with feedback, 2016-10-03) - judged not relevant
- P20: [Implement NULL-related checks in object address functions to prevent cache lookup errors](https://commitfest.postgresql.org/patch/1204/) (cf, Returned with feedback, 2018-11-29) - judged not relevant
- P21: [Revert "Skip redundant anti-wraparound vacuums"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=dd9ac7d5d80608a640bb82cffb6a805ce84cf112) (revert, Reverted, 2020-03-30) - judged not relevant
- P22: [\[PROPOSAL\] : Disallow use of empty column name in (column_name '') in ALTER or CREATE of foreign table.](https://commitfest.postgresql.org/patch/5184/) (cf, Rejected, 2025-03-26) - judged not relevant
- P23: [Allow to warn or error on ... = NULL expressions](https://commitfest.postgresql.org/patch/1728/) (cf, Returned with feedback, 2018-12-01) - judged not relevant
- P24: [Add ALWAYS DEFERRED option for CONSTRAINTs and CONSTRAINT TRIGGERs](https://commitfest.postgresql.org/patch/1319/) (cf, Returned with feedback, 2019-01-31) - judged not relevant

### Warnings

- synthesis said NEEDS-COMMITTER-EYES but listed a level 1b objection (standards-compat-1); changed to NEEDS-DESIGN-DISCUSSION

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 11; wall time: 84 s; cost reported by the CLI: $0.98
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 8 from cache, 0 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 64 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:54:51.453Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

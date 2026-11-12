# Design pre-review: [PATCH v1] Fix for Bug#19724 - ALTER TYPE ... ALTER ATTRIBUTE triggers internal error for base type of domain with check

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7356](https://commitfest.postgresql.org/patch/7356/) (Needs review); authors: Nitin Motiani (nitinmotiani)
- Thread: [\[PATCH v1\] Fix for Bug#19724 - ALTER TYPE ... ALTER ATTRIBUTE triggers internal error for base type of domain with check](https://www.postgresql.org/message-id/flat/CAH5HC94%2B4teDZvuWkBiAikcK1pM2DH9W6iCNDoGpHxUdPGTNPw%40mail.gmail.com), started 2026-09-28
- Messages read: 11 of 11
- Patch prose read: v4-0003-Don-t-require-ownership-when-rebuilding-constrain.patch, v4-0001-Fix-ALTER-.-TYPE-failure-with-dependent-domain-co.patch, v4-0002-Validate-re-added-domain-constraints-after-ALTER-.patch, v1-0001-Fix-ALTER-TYPE-.-ALTER-ATTRIBUTE-on-types-used-in.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

The goal is a bug fix for an internal error and a possible crash on every supported branch, and reviewers agree on 0001. Two fixable design points remain for stable branches: replication-ondisk-2 (0003 widens the name-based domain lookup race to domains the caller does not own) and standards-compat-1 (0002 touches an exported prototype in typecmds.h). Neither blocks the idea, but a committer must decide what goes to back branches.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, I could not check whether 0002 changes the existing AlterDomainAddConstraint() signature or only adds a declaration. I also could not check how the deferred validation orders its catalog updates or which identity it runs under.

## The proposal, restated

**Problem.** ALTER TYPE ... ALTER ATTRIBUTE ... TYPE and ALTER TABLE ... ALTER COLUMN ... TYPE fail with "could not identify relation associated with constraint" when a domain over a non-composite type (int, float8, int[], a domain over a domain) has a CHECK that references the altered composite type or row type. The bug dates from af20e2d72 (v11), so every supported branch has it.

**Proposed design.** 0001 attaches the domain constraint rebuild to the relation being altered (tab->relid) instead of deriving a relation from the domain's base type. 0002 re-adds the domain constraint without validation and validates it after phase 3 rewrites, so rows are no longer read with the new tuple descriptor over the old heap. 0003 skips ownership checks when rebuilding constraints and restores comments with CreateComments() instead of CommentObject().

**User-visible surface.**
- ALTER TYPE ... ALTER ATTRIBUTE ... TYPE and ALTER TABLE ... ALTER COLUMN ... TYPE succeed where they failed with an internal error.
- The rebuild no longer takes AccessExclusiveLock on another composite type's relation when the domain is over that other type.
- Altering your own type no longer fails with "must be owner of type/relation" because of another user's dependent domain or commented constraint.
- Domain constraint validation errors now come after table rewrites, not during phase 2.
- AlterDomainAddConstraint() changes in typecmds.h, a C API visible to extensions.

**What the author claims.**
- Using tab->relid is safe because the relid only picks which relation gets locked and which work queue entry gets the command.
- The fix, with tests, applies cleanly to REL_14_STABLE through REL_19_STABLE; reviewers support back-patching.
- Early validation can give wrong values, false constraint violations, "type with OID 4294967295 does not exist", or a crash.
- Ownership checks make no sense on re-add because the matching drop is done without permission checks.
- Any user can block a type owner's ALTER, since types grant USAGE to PUBLIC by default.

**Thread state.** Ayush Tiwari, Rahul Yadav, Matheus Alcantara and Zsolt Parragi reviewed; Matheus wrote 0002 and 0003, and v4 is the latest. Open points: whether to squash 0001 and 0002, Ayush's pending review of 0003, and the name-based domain lookup race, which the author suggests fixing later without back-patching.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-09-28; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Name-based re-add can attach a constraint to the wrong domain

fixable · level 2, rubric item 7 · catalog changes, security / privileges, backpatchability · `replication-ondisk-2`

Session A runs ALTER TYPE while session B renames domain d and creates a new domain d over the same type. The rebuilt d_check is stored in pg_constraint on the new domain, and the original domain loses it. Later inserts into columns of the original domain skip the check, and pg_dump keeps the wrong state.

Message #10 shows this race succeeding silently, and message #11 defers the fix to a separate patch that would not be back-patched. The bug is older than this patch, but 0003 removes the ownership check, so after v4 the stale lookup also succeeds when the domain belongs to another user. 0002 already tracks constraint OIDs in the work queue, so resolving the domain by OID may not need much more code.

Thread: [#8 (Zsolt Parragi, 2026-09-28)](https://www.postgresql.org/message-id/CAN4CZFMhvTw7J8kNcU%2BZxxWq2oUg%2B-hiaeVH%3DsdEt1%2BrtOK3Pg%40mail.gmail.com), [#10 ("Matheus Alcantara", 2026-09-29)](https://www.postgresql.org/message-id/DLRUNGYX33TS.2M17VTR2OBXPC%40gmail.com), [#11 (Nitin Motiani, 2026-10-03)](https://www.postgresql.org/message-id/CAH5HC945XpFgKhZK9vQug5-A%3D_-dWRt2_LzOp9unA4JEAA941Q%40mail.gmail.com)

**Question:** Can the re-add use the domain OID taken before the drop, so that 0003 does not ship to stable branches while the name lookup can still pick the wrong domain?

Counter-check: held. Message #10 confirms the constraint can silently move to a new domain of the same name, and that 0003 extends this to domains the caller does not own. Message #11 defers the fix without back-patching, so the concern is open for stable branches.

### 2. 0002 changes an exported function signature in a back-patched fix

fixable · level 2, rubric item 6 · backpatchability, ABI / extension API · `standards-compat-1`

An extension built against an earlier minor release of 14 through 19 calls AlterDomainAddConstraint(). After a minor upgrade, the server passes or expects arguments the extension's compiled call does not supply. The extension then fails at load time, or misbehaves at run time.

0002 touches src/include/commands/typecmds.h and changes AlterDomainAddConstraint(), so the fix depends on a changed prototype. Message #3 supports back-patching to REL_14_STABLE through REL_19_STABLE, and the CommitFest entry targets stable. The thread checks only that v1 applies cleanly. Nobody has said whether 0002 goes to the back branches, or how its signature change would be kept ABI-compatible there.

Thread: [#3 (Rahul Yadav, 2026-09-28)](https://www.postgresql.org/message-id/CAJJjRRf%2Bb21kqB6-fKCBAqrOhBT12K5%2BV-Pa6wCpZi%2BD9xKiCg%40mail.gmail.com), [#5 ("Matheus Alcantara", 2026-09-28)](https://www.postgresql.org/message-id/DLR67P6TO2C8.3SL4J7XR66QBJ%40gmail.com), [#11 (Nitin Motiani, 2026-10-03)](https://www.postgresql.org/message-id/CAH5HC945XpFgKhZK9vQug5-A%3D_-dWRt2_LzOp9unA4JEAA941Q%40mail.gmail.com)

**Question:** For back branches, will 0002 keep the existing AlterDomainAddConstraint() signature, for example with a new wrapper function, so extensions built against earlier minor releases keep working?

Counter-check: held. The thread does not say whether 0002 goes to back branches or how the typecmds.h change stays ABI-compatible there. The material shows only that the header is touched, so the author should confirm whether an existing signature changes or a declaration is only added.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | **yes** | The rebuild drops and re-creates pg_constraint rows for domain CHECK constraints, and finds the domain by name. |
| ABI / extension API | **yes** | 0002 changes AlterDomainAddConstraint() in src/include/commands/typecmds.h, which extensions can call. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | **yes** | 0003 skips the domain ownership check on re-add and restores comments with CreateComments(). |
| backpatchability | **yes** | The CommitFest entry targets stable, and 0002 and 0003 change a C prototype and permission behaviour. |
| maintenance burden ("who maintains this in 10 years") | **yes** | 0002 adds a deferred validation step for domain constraints after phase 3 in ATRewriteTables(). |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Which of 0001, 0002 and 0003 are meant for back-patching to REL_14_STABLE through REL_19_STABLE?
2. Can the re-add use the domain OID taken before the drop, so 0003 does not reach stable branches while the name lookup can pick another user's domain?
3. For back branches, does 0002 keep the existing AlterDomainAddConstraint() signature, for example through a new wrapper function?
4. Does the new validation loop after phase 3 need CommandCounterIncrement() between constraints, and what case shows it?
5. Will the stack depth error seen with a ROW(VALUE, NULL)::t check be handled in this patch set or left as is?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows the ALTER TABLE phase 2 and phase 3 work queue in tablecmds.c and the domain constraint code in typecmds.c; no such committer has spoken in the visible thread.
- **Liability:** The owner signs for a back-patched change to ALTER TABLE rewrite ordering, a relaxed ownership check, and an exported prototype, on six branches for their remaining support life.
- **Author follow-up:** The author and a co-author posted revisions up to v4 within about six weeks and answered review comments, which suggests they stay for fixes.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs champion
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** medium
- **Why:** The bug gives an internal error or a crash on all supported branches, but only for domains whose CHECK references an altered composite or row type. The three patches are small, yet back-patching 0002 and 0003 needs a committer's decision and none is engaged.

## Appendix

### Refuted in the counter-check

- `replication-ondisk-1` Back-patched AlterDomainAddConstraint() signature change breaks extension ABI. Refuted (not specific): Duplicate of standards-compat-1.
- `security-1` 0003 runs another user's domain check code as the ALTER caller. Refuted (already handled): Message #5 notes that rebuilding another user's table CHECK constraint already has no ownership check, and that rebuild revalidates the expression as the ALTER caller. Bob can reach the same result today with a table CHECK, so 0003 follows existing practice and adds no new exposure.
- `security-2` Without the ownership check, the name lookup race can modify other users' domains. Refuted (not specific): Duplicate of replication-ondisk-2.
- `maintenance-fit-1` Back-patching 0002 changes an exported function signature in stable branches. Refuted (not specific): Duplicate of standards-compat-1.
- `maintenance-fit-2` Back-patching 0003 widens a known race whose fix is planned for master only. Refuted (not specific): Duplicate of replication-ondisk-2.

### Review angles

- prior-attempts: 0 objection(s). No precedent found: the search returned no earlier rejected, returned or withdrawn proposal of the same or a similar idea. Message #1 points to the 2017 thread behind af20e2d72, which showed the same float8 case without a test; that is the origin of the bug, not a rejected attempt at this fix.
- standards-compat: 1 objection(s). Beyond the ABI point, this is a bug fix with no new syntax, GUC, catalog object or pg_dump surface. The later error timing in 0002 and the relaxed ownership checks in 0003 move behaviour toward what users would expect, and I found no concrete compatibility failure in either.
- replication-ondisk: 2 objection(s). There is no on-disk format, WAL, replication, wire protocol or catalog schema change. The only issues on this angle are the exported C API in typecmds.h and how persistent constraint rows are re-attached.
- security: 2 objection(s). 0001 and 0002 raise no privilege concern on their own. Both objections concern 0003, mainly in combination with back-patching.
- maintenance-fit: 2 objection(s). The core fix, 0001, has no maintenance or fit objection from this angle. It is a narrow bug fix in existing code, has no hot-path cost, and several reviewers agree on it.
- already-exists: 0 objection(s). This is a bug fix for an internal error in ALTER ... TYPE, and no existing feature, option or extension avoids it. The only workaround is to drop the domain constraint, alter the type and re-add the constraint by hand. That is a workaround for a bug and not an alternative design, so this angle has nothing to raise.
- demand: 0 objection(s). This is a bug fix, so the demand test does not apply. The command fails with an internal XX000 error or crashes, it was reported as bug #19724, and four reviewers reproduced it on every supported branch. Rarity of the setup (message #1) is no reason to keep an internal error or a crash. Whether 0003's ownership change belongs in stable branches is a design question for other angles.

### Precedents retrieved

- P1: [refactor AlterDomainAddConstraint (alter domain add constraint)](https://commitfest.postgresql.org/patch/5455/) (cf, Returned with feedback, 2026-04-02) - judged not relevant; reason summarised from its thread
- P2: [clean up create|alter domain stmt incompatiable constraint error case and add regression test](https://commitfest.postgresql.org/patch/5385/) (cf, Withdrawn, 2024-12-09) - judged not relevant; reason summarised from its thread
- P3: [Fix pg_get_constraintdef for NOT NULL constraints on domains](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=09d09d4297b9acbc2848ec35e8bf030d6c1fae18) (revert, Partially reverted, 2024-11-27) - judged not relevant
- P4: [Change a constraint's index - ALTER TABLE ... ALTER CONSTRAINT ... USING INDEX ...](https://commitfest.postgresql.org/patch/2635/) (cf, Returned with feedback, 2020-09-30) - judged not relevant; reason summarised from its thread
- P5: [Revert "Enable fast default for domains with non-volatile constraints"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a0354e29c41a9fb7491b3c7c23f079b1923c045a) (revert, Reverted, 2026-06-08) - judged not relevant
- P6: [Allow values to be cast to domain types](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Domains) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P7: [Add support for ON UPDATE/DELETE actions on ALTER CONSTRAINT](https://commitfest.postgresql.org/patch/1533/) (cf, Returned with feedback, 2021-04-08) - judged not relevant; reason summarised from its thread
- P8: [Relax transactional restrictions on ALTER TYPE ... ADD VALUE (redux).](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=212fab9926b2f0f04b0187568e7124b70e8deee5) (revert, Partially reverted, 2018-10-08) - judged not relevant
- P9: [Fix permission tests for views/tables proven empty by constraint exclusion.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=50c137487c96e629e0e5372bb3d1b5f1a2f71a88) (revert, Partially reverted, 2013-05-01) - judged not relevant
- P10: [Restructure ALTER TABLE notes to clarify table rewrites and verification scans](https://commitfest.postgresql.org/patch/3604/) (cf, Returned with feedback, 2022-08-02) - judged not relevant; reason summarised from its thread
- P11: [Make domains work better with polymorphic functions](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Domains) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P12: [Allow functions defined as casts to domains to be called during casting](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Domains) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P13: [plpgsql - possibility to get element or array type for referenced variable type](https://commitfest.postgresql.org/patch/412/) (cf, Rejected, 2016-03-16) - judged not relevant; reason summarised from its thread
- P14: [CREATEROLE and role ownership hierarchies](https://commitfest.postgresql.org/patch/3506/) (cf, Returned with feedback, 2022-04-12) - judged not relevant; reason summarised from its thread
- P15: [Reduce lock level for ALTER TABLE ... ADD CHECK](https://commitfest.postgresql.org/patch/3088/) (cf, Withdrawn, 2021-11-21) - judged not relevant; reason summarised from its thread
- P16: [Revert "Fix bug in checking of IDENTIFY_SYSTEM result."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=c4762886539b06da682045e19ba73dcee3789638) (revert, Reverted, 2014-08-19) - judged not relevant
- P17: [pg_get_domain_ddl: DDL reconstruction function for CREATE DOMAIN statement](https://commitfest.postgresql.org/patch/6983/) (cf, Withdrawn, 2026-09-07) - judged not relevant
- P18: [Add pg_ownerships and pg_privileges system views](https://commitfest.postgresql.org/patch/5033/) (cf, Withdrawn, 2025-01-04) - judged not relevant
- P19: [ALTER TABLE - check for column existence before starting operations](https://commitfest.postgresql.org/patch/1621/) (cf, Returned with feedback, 2018-10-02) - judged not relevant
- P20: [revert: Generalize relation analyze in table AM interface](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6377e12a5a5278446bb0d215439b4825ef8996d1) (revert, Reverted, 2024-04-16) - judged not relevant
- P21: [Revert "Use Foreign Key relationships to infer multi-column join selectivity".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=77ba610805e7ef9ba9c9a593ea8b1ca8f98f8bcb) (revert, Reverted, 2016-06-07) - judged not relevant
- P22: [Fix for REFRESH MATERIALIZED VIEW ownership error message](https://commitfest.postgresql.org/patch/1766/) (cf, Rejected, 2018-08-19) - judged not relevant
- P23: [Rewrite undirected edge patterns in GRAPH_TABLE using UNION ALL](https://commitfest.postgresql.org/patch/7034/) (cf, Returned with feedback, 2026-09-10) - judged not relevant
- P24: [Revert badly-broken patch to dump comments on composite-type columns.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=18661f282bae5599abb68f53bc8b451b2d67bd75) (revert, Reverted, 2004-08-06) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 12; wall time: 143 s; cost reported by the CLI: $1.76
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 8 from cache, 0 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 83 text field(s) checked, 1 finding(s), 1 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:44:13.905Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

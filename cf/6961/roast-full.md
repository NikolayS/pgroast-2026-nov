# Design pre-review: REVOKE's CASCADE protection doesn't work with inherited table owners

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6961](https://commitfest.postgresql.org/patch/6961/) (Needs review); authors: Jacob Champion (jchampion)
- Thread: [REVOKE's CASCADE protection doesn't work with INHERITed table owners](https://www.postgresql.org/message-id/flat/CAOYmi%2B%3DKTLd%2BXsEP%3DTDiZ48iVf-CEc7JrZd5uhWPYWKEfOgyyQ%40mail.gmail.com), started 2026-06-24
- Messages read: 7 of 7
- Patch prose read: v1-0001-Prevent-broken-grant-chains-when-indirect-grant-o.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

Nothing that survived questions the fix itself: the problem is reproducible, and the old behaviour leaves ACL entries that pg_dump cannot restore. The one surviving objection, prior-attempts-1, is about back-patching to 14. A REVOKE that succeeds today would fail in a minor release, and the thread neither weighs that nor gives release-note text. That is a stable-branch policy decision for a committer who knows the privilege code.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read. This review cannot confirm that aclmask_direct() gives the intended result for every grantor case, including SUPERUSER and PUBLIC, or that no legitimate REVOKE now fails.

## The proposal, restated

**Problem.** recursive_revoke() lets REVOKE without CASCADE succeed when the grantor holds its grant option or table ownership only through role membership (INHERIT) or SUPERUSER. The dependent grant then stays in the ACL with no valid grant option behind it. pg_dump output for such an ACL cannot recreate it, so restore fails or silently drops the grant.

**Proposed design.** recursive_revoke() switches from aclmask() to aclmask_direct(), so only directly held grant options and real ownership count when it checks whether the grantee still holds the option. REVOKE then raises "dependent privileges exist" unless CASCADE is given. The fix is planned for back-patching to 14.

**User-visible surface.**
- REVOKE ... RESTRICT now fails with "dependent privileges exist" in cases where it used to succeed: a grantor holding options through INHERIT, SUPERUSER or PUBLIC.
- REVOKE ... CASCADE now also removes dependent grants in those cases.

**What the author claims.**
- The current behaviour is unintended and leaves orphaned ACL entries that dump/restore cannot recreate.
- The fix is safe to back-patch through 14.
- New regression tests cover ownership inherited through INHERIT, grant options inherited through INHERIT, and grant options through PUBLIC.
- A reviewer reports that the patch applies on 14 through 18 and master, with 3-way merge on 14 and 15, and that the regression suite passes on all of them.

**Thread state.** Ayush Tiwari and Kirill Reshke gave LGTM, and Paul Kim added a superuser test as v2-0002, but no committer has replied. The comment wording is still open, and so is a related check_circularity() fix for self-grants, which could break restore of existing clusters and needs its own dump/restore change.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-06-24; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Back-patched REVOKE behavior change can break existing scripts

fixable · level 2, rubric item 6 · backpatchability, security / privileges · `prior-attempts-1`

An admin's deployment script runs `revoke all on admin_table from bob`, where bob's grant option comes through role membership or SUPERUSER. After a minor update to 14.x through 18.x, that statement fails with "dependent privileges exist". The migration aborts, and the admin finds out only on the first deploy after the update.

Message #2 proposes back-patching "all the way down", and the commit message says Backpatch-through: 14. The thread does not discuss whether a REVOKE that used to succeed should start failing in a minor release, and no release-note text is given. [P1] is a privileges change back-patched to 14 that was reverted because it "broke existing behavior". Here the old behavior leaves ACLs that pg_dump cannot restore, which is a stronger case for the fix, but a committer would likely still weigh master-only against back-patching.

**Precedents.**
- [Revert b2a459edf "Fix GRANTED BY support in REVOKE ROLE statements"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e68570e388f08c2e36ce7d2a9564941b89db6549) (Reverted, 2021-12-30); thread: [#1](https://www.postgresql.org/message-id/flat/17346-f72b28bd1a341060@postgresql.org), [#2](https://www.postgresql.org/message-id/flat/B7F6699A-A984-4943-B9BF-CEB84C003527@yesql.se), [#3](https://www.postgresql.org/message-id/flat/f2feac44-b4c5-f38f-3699-2851d6a76dc9@2ndquadrant.com)

Thread: [#2 (Jacob Champion, 2026-06-26)](https://www.postgresql.org/message-id/CAOYmi%2Bn_hw%3DSC5V1i3BmqfZPfPBRaUSJc%2BBeOXEKDwRue%2BWYrg%40mail.gmail.com), [#7 (Paul Kim, 2026-09-30)](https://www.postgresql.org/message-id/179074273441.6180.2515281464552050662%40mail.gmail.com)

**Question:** Why should existing REVOKE RESTRICT scripts fail in minor releases rather than only on master, and what will the release notes tell affected users?

Counter-check: held. The commit message sets Backpatch-through: 14, and the thread never weighs a REVOKE that starts failing in minor releases. P1 reverted a spec-compliance change, so it must not be used as support here.

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
| pg_dump / restore round-trip | **yes** | The fix stops REVOKE from leaving ACL entries that pg_dump output cannot recreate on restore. |
| psql UX conventions | no |  |
| security / privileges | **yes** | recursive_revoke() in acl.c changes which grant options count when REVOKE checks for dependent privileges. |
| backpatchability | **yes** | The commit message says Backpatch-through: 14, and REVOKE without CASCADE starts failing in cases that succeed today. |
| maintenance burden ("who maintains this in 10 years") | no |  |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Why should the stricter REVOKE reach stable branches 14 through 18 and not only master, given that scripts that succeed today will fail after a minor update?
2. What release-note text will tell users that REVOKE without CASCADE can now fail, and that REVOKE ... CASCADE now removes dependent grants it used to leave?
3. Do other grant-option checks that call aclmask() have the same flaw as recursive_revoke()?
4. How can an administrator find ACL entries that this bug already orphaned in an existing cluster before running pg_dump or pg_upgrade?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. The patch needs a committer who knows the ACL and grant-option code in acl.c and the back-patch rules for behaviour changes. None has replied in the visible thread.
- **Liability:** The committer signs for a REVOKE behaviour change in five stable branches. Any script that breaks after a minor update is reported against that commit, and the back-patch may have to be reverted.
- **Author follow-up:** The author posted the patch two days after the report and said he would look at the related check_circularity() case. Later messages were not shown.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs champion
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch changes one call in acl.c and adds regression tests. It has two LGTMs but no committer reply after three CommitFests, and only a committer can settle the back-patch question.

## Appendix

### Refuted in the counter-check

- `standards-compat-1` Back-patched REVOKE behaviour change can break scripts after a minor update. Refuted (not specific): Duplicate of prior-attempts-1. Its use of P1 also does not fit: this patch stops ACL entries that pg_dump cannot restore, and P1 was a spec-compliance change.
- `standards-compat-2` Back-patch leaves other orphaned-grant paths and existing orphans unhandled. Refuted (no failure scenario): The scenario happens today without the patch, and the patch does not cause it or make it worse. Message #6 treats the role-membership path as a separate fix ("we need both"), so it does not count against this one.
- `replication-ondisk-1` Back-patched REVOKE starts failing in minor releases with no upgrade note. Refuted (not specific): Duplicate of prior-attempts-1 for the minor-release error and the release note. The part about existing orphaned entries describes a state the patch neither creates nor worsens.
- `maintenance-fit-1` Back-patched REVOKE starts failing in minor releases. Refuted (not specific): Duplicate of prior-attempts-1. Its claim that such role setups are common has no support in the material.

### Review angles

- prior-attempts: 1 objection(s). [P2] covers REVOKE ROLE grantor semantics, records no reason, and is a different idea, so it is skipped. The related check_circularity() fix in messages #2-#6 and Kirill Reshke's earlier patch are outside this patch.
- standards-compat: 2 objection(s). The change brings grant-option chains closer to direct privilege descriptors, so there is no SQL-standard or syntax objection. The objections are about back-patching and dump round-trip, not the fix on master.
- replication-ondisk: 1 objection(s). The patch changes no on-disk format, WAL record, catalog layout, wire protocol or extension ABI. ACL updates replicate through the normal catalog WAL. The check_circularity() change, which could break restore, is out of this patch, as message #5 agreed.
- security: 0 objection(s). From the security angle the patch narrows privileges. It closes a path where a grant stays in the ACL with no valid grant option behind it, and it adds no new privilege, default or exposure. The back-patch behaviour change and the pre-existing orphaned ACLs belong to other angles. The membership-revoke case in message #6 and the check_circularity() self-grant case are separate gaps that this patch does not make worse.
- maintenance-fit: 1 objection(s). The fit and maintenance cost are small: one call changes in acl.c, REVOKE is not a hot path, and this is a core privilege bug. No committer has replied yet, which is a committability concern only.
- already-exists: 0 objection(s). This is a bug fix inside recursive_revoke(), so no existing feature, extension or tool can stand in for it. A user could use REVOKE ... CASCADE by hand, but only after spotting the broken grant chain. The fix reuses the existing aclmask_direct() and adds no second mechanism. Neither precedent fits this angle: [P1] and [P2] are about REVOKE ROLE and GRANTED BY, and say nothing about grant-option chain checks.
- demand: 0 objection(s). This is a bug fix for an orphaned ACL that pg_dump cannot restore. Message #1 says the case was minimized from a real setup, and messages #5 to #7 show independent reproductions on 18.4 and master, so the demand angle does not apply. The minor-release behaviour change is a design and back-patch question for other angles.

### Precedents retrieved

- P1: [Revert b2a459edf "Fix GRANTED BY support in REVOKE ROLE statements"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e68570e388f08c2e36ce7d2a9564941b89db6549) (revert, Reverted, 2021-12-30) - used in the review
- P2: [Implement the SQL-standard mechanism whereby REVOKE ROLE revokes only the privilege granted by the invoking role, and not those granted by other roles](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Administration) (do_not_want, Not worth doing, date unknown) - used in the review
- P3: [Revert "Avoid race condition between "GRANT role" and "DROP ROLE"".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=8abbbbae610cd62faad4f9ec18e7cfa908593796) (revert, Reverted, 2025-09-16) - judged not relevant
- P4: [Allow multiple recursive self-references](https://commitfest.postgresql.org/patch/3046/) (cf, Returned with feedback, 2022-08-02) - judged not relevant; reason summarised from its thread
- P5: [Allow logical replication via inheritance root table](https://commitfest.postgresql.org/patch/4225/) (cf, Returned with feedback, 2023-08-31) - judged not relevant; reason summarised from its thread
- P6: [Delegating superuser tasks to new security roles](https://commitfest.postgresql.org/patch/3223/) (cf, Withdrawn, 2021-11-15) - judged not relevant; reason summarised from its thread
- P7: [Revert non-text output formats for pg_dumpall](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=7ca548f23a60887541493b2da47f2b2720c35ee3) (revert, Reverted, 2026-06-18) - judged not relevant
- P8: [table inheritance versus column compression and storage settings](https://commitfest.postgresql.org/patch/4813/) (cf, Returned with feedback, 2024-03-21) - judged not relevant; reason summarised from its thread
- P9: [Revert structural changes to not-null constraints](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6f8bb7c1e9610dd7af20cdaf74c4ff6e6d678d44) (revert, Reverted, 2024-05-13) - judged not relevant
- P10: [Revert "Improve compression and storage support with inheritance"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=74563f6b90216180fc13649725179fc119dddeb5) (revert, Reverted, 2024-02-20) - judged not relevant
- P11: [pg_dump -C Grant statement on create database](https://commitfest.postgresql.org/patch/724/) (cf, Returned with feedback, 2016-10-03) - judged not relevant; reason summarised from its thread
- P12: [Revert: Remove useless self-joins](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=d1d286d83c0eed695910cb20d970ea9bea2e5001) (revert, Reverted, 2024-05-06) - judged not relevant
- P13: [Dump-restore loosing 'attnotnull' bit for DEFERRABLE PRIMARY KEY column(s)](https://commitfest.postgresql.org/patch/4729/) (cf, Rejected, 2023-12-28) - judged not relevant; reason summarised from its thread
- P14: [Allows database-specific role memberships](https://commitfest.postgresql.org/patch/3374/) (cf, Returned with feedback, 2022-10-12) - judged not relevant; reason summarised from its thread
- P15: [Revert Non text modes for pg_dumpall, and pg_restore support](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ce9a6244b5b4ce1df71611512a757353803404a5) (revert, Reverted, 2025-07-30) - judged not relevant
- P16: [Fix pg_restore --no-globals SQL output for pg_dumpall archives](https://commitfest.postgresql.org/patch/6899/) (cf, Withdrawn, 2026-06-20) - judged not relevant; reason summarised from its thread
- P17: [Document efficient self-joins / UPDATE LIMIT techniques.](https://commitfest.postgresql.org/patch/4539/) (cf, Returned with feedback, 2024-02-13) - judged not relevant
- P18: [ALTER TABLE: emit NOTICE when ONLY is omitted but no partition recursion occurs](https://commitfest.postgresql.org/patch/6424/) (cf, Withdrawn, 2026-01-22) - judged not relevant
- P19: [Remove ctid from self-join examples in UPDATE and DELETE docs](https://commitfest.postgresql.org/patch/6246/) (cf, Withdrawn, 2026-01-16) - judged not relevant
- P20: [DELETE CASCADE](https://commitfest.postgresql.org/patch/3195/) (cf, Returned with feedback, 2022-01-31) - judged not relevant
- P21: [Revert "postgres_fdw: Inherit the local transaction's access/deferrable modes."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=7d4667c6203d0174903a53de9d1730fbec26576b) (revert, Reverted, 2025-06-08) - judged not relevant
- P22: [Add option --drop-cascade for pg_dump/restore](https://commitfest.postgresql.org/patch/3065/) (cf, Returned with feedback, 2021-12-01) - judged not relevant
- P23: [pg_rewind with cascade standby doesn't work well](https://commitfest.postgresql.org/patch/4559/) (cf, Returned with feedback, 2024-10-07) - judged not relevant
- P24: [Add document is_superuser](https://commitfest.postgresql.org/patch/3889/) (cf, Returned with feedback, 2023-12-02) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 14; wall time: 112 s; cost reported by the CLI: $1.51
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 68 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:10:38.429Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

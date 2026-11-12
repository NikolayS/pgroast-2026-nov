# Design pre-review: pg_dump: skip pg_init_privs entries for non-existent roles

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6915](https://commitfest.postgresql.org/patch/6915/) (Needs review); authors: Hüseyin Demir (demir)
- Thread: [BUG #19483: pg_upgrade fails with orphan records in pg_init_priv catalog table](https://www.postgresql.org/message-id/flat/19483-80de42dc4e62cfd6%40postgresql.org), started 2026-05-18
- Messages read: 35 of 35
- Patch prose read: v1-0001-pg_dump-skip-dangling-initprivs.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No objection survived at levels 1a or 1b: the patch fixes a reproducible pg_upgrade failure and follows the pg_dump direction suggested in message #4. Two fixable design points remain at level 2: standards-compat-1 (dropping an aclitem whose grantor alone is dangling causes lasting dump drift) and maintenance-fit-1 (no server version gate on the filter). Neither blocks the idea, but a committer who knows pg_dump's ACL code has to accept the trade-off and the backpatch range.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, I could not check that the filter covers every query that reads pg_init_privs, nor how often the per-table column ACL query runs. The behaviour against v15 and v16 source servers is untested in the material.

## The proposal, restated

**Problem.** On servers before v17, DROP ROLE can leave pg_init_privs ACL entries that point to the dropped role by numeric OID. pg_dump emits these as GRANT or SET SESSION AUTHORIZATION "<oid>", so pg_upgrade from such clusters fails during restore after --check passed. The reporter hit this upgrading v14 to v18 with extensions installed by a role later dropped.

**Proposed design.** pg_dump filters each initprivs array on the server and drops any aclitem whose grantor or grantee OID is missing from pg_roles, but keeps PUBLIC (grantee 0). It applies a SAFE_INITPRIVS macro in getAdditionalACLs(), which runs once per dump, and in the per-table column ACL query. An object whose items are all dangling gets no initial privileges.

**User-visible surface.**
- pg_dump output: dangling-role aclitems from pg_init_privs no longer appear in GRANT, REVOKE or SET SESSION AUTHORIZATION.
- After upgrade, affected extension members lose their pg_init_privs row, so later dumps re-emit the extension's own REVOKE and GRANT lines.
- Separate proposal, HEAD only: aclitemout (putid()) would quote all-digit role names so they differ from dangling OIDs.

**What the author claims.**
- A failed pg_upgrade becomes a restore that works, with two extra harmless lines in later dumps.
- Overhead is about 1 to 2 ms on the one-time pg_init_privs fetch and does not grow with object count.
- Laurenz Albe saw no noticeable effect in a pg_upgrade --link of 20000 empty tables.
- pg_dump still works as a non-superuser because the check uses pg_roles instead of pg_authid.
- The fix can be backpatched to v15; pg_upgrade uses the new cluster's pg_dump, so the v18 copy fixes the v14 to v18 case.

**Thread state.** Laurenz Albe and Rui Zhao reviewed v9 and Rui reports no open items; the CF entry has gone back to Needs review after Waiting on Author. Tom Lane's objections about per-object query cost, and his putid() HEAD-only idea, got no visible final answer from him on v6 and later.

## 2. Prior attempts

### pg_upgrade fails with non-standard ACL (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/2215/](https://commitfest.postgresql.org/patch/2215/)
- **Thread:** [https://www.postgresql.org/message-id/flat/f85991ad-bbd4-ad57-fde4-e12f0661dbf0@postgrespro.ru](https://www.postgresql.org/message-id/flat/f85991ad-bbd4-ad57-fde4-e12f0661dbf0@postgrespro.ru)
- **Date:** decided 2021-11-26; started 2019-07-18
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Noah Misch (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** The author stopped responding after Noah Misch's March 2021 review, so the entry sat in Waiting on Author until Daniel Gustafsson returned it in November 2021. That review showed that the generated script still let the final pg_upgrade fail on a REVOKE ... FROM PUBLIC, and that the check compared obj_ident without obj_type. Noah Misch also had doubts about the approach, since pg_upgrade code that predicts restore failures cannot match a real restore.
- **Reason (quote):** "I'm not generally enthusiastic about adding pg_upgrade code to predict whether the dump will fail to restore, because such code will never be as good as just trying the restore." ([message](https://www.postgresql.org/message-id/20210127112135.GB2481576%40rfd.leadboat.com))
- **What is different this time:** P3 had pg_upgrade predict restore failures from ACLs and ask the user to run a cleanup script. This proposal dropped the pg_upgrade check from message #1 and makes pg_dump skip the dangling aclitems, so the restore itself succeeds.


## 3. Objections a reviewer is likely to raise

### 1. Dropping whole aclitems leaves upgraded extensions with permanent dump drift

fixable · level 2, rubric item 12 · pg_dump / restore round-trip, backpatchability · `standards-compat-1`

A DBA upgrades a v15 cluster whose extension owner was dropped. The new cluster has no pg_init_privs row for those members, so every later pg_dump emits REVOKE and GRANT lines after CREATE EXTENSION. A restore against a newer extension version where a member was renamed then fails, and schema-diff tools report privilege changes nobody made.

The filter drops an aclitem when only its grantor is dangling, so a valid PUBLIC grant such as =r/16384 also leaves pg_init_privs (messages #33, #35). The patch now documents this drift, but documenting it does not remove it. A committer would likely ask why the patch drops these items instead of mapping the dangling role to the object owner. As far as I know that mapping is closer to what REASSIGN OWNED does on v17 and later, and it would match a fresh install.

**Precedents.**
- [pg_upgrade fails with non-standard ACL](https://commitfest.postgresql.org/patch/2215/) (Returned with feedback, 2021-11-26); thread: [link](https://www.postgresql.org/message-id/flat/f85991ad-bbd4-ad57-fde4-e12f0661dbf0@postgrespro.ru)

Thread: [#4 (Tom Lane, 2026-06-07)](https://www.postgresql.org/message-id/104812.1780847519%40sss.pgh.pa.us), [#19 (Tom Lane, 2026-06-24)](https://www.postgresql.org/message-id/1030965.1782313078%40sss.pgh.pa.us), [#33 (Rui Zhao, 2026-08-10)](https://www.postgresql.org/message-id/CAHWVJhGQ5qGHnxYqbDEkj1WdXosbR0xk_ZdaY5tK7y0HHhWvhg%40mail.gmail.com), [#34 (Hüseyin Demir, 2026-09-04)](https://www.postgresql.org/message-id/CAB5wL7ZXt0h1F4pQfKERUxcHhkKUMYEMm%3Dy9%2BJe7eWTLjrLTPg%40mail.gmail.com), [#35 (Rui Zhao, 2026-09-09)](https://www.postgresql.org/message-id/CAHWVJhGamC1wN5LB6b5XAKwQBxqx1wtM-GaYq2CJXZVNa4MF7w%40mail.gmail.com)

**Question:** Why drop items with a dangling grantor or grantee instead of substituting the object owner, so upgraded extension members dump the same as a fresh install?

Counter-check: held. Messages #33 to #35 confirm the drift, and message #19 suggests keeping a grant whose grantor is dangling, so the design question stands. P3 concerns a pg_upgrade check and must not be used as support; the renamed-member restore failure is speculative.

### 2. Filter runs on every dump, including from v17+ servers where it cannot match

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years"), performance on the hot path, backpatchability · `maintenance-fit-1`

Someone running HEAD pg_dump against a v19 server with many tables pays for the SAFE_INITPRIVS subquery in the per-table column ACL query on every table, although dangling rows cannot exist there. Years later, a developer changing the pg_init_privs queries cannot tell whether the macro is still needed. Nothing fails if the filter is dropped or missing from a new query site.

Message #4 and message #32 show dangling rows only come from pre-v17 servers, yet v6 to v9 apply the filter without a remoteVersion check. Message #23 objected to the per-table subquery, and Tom Lane, who raised the cost concern in message #14, gives no view on v6 or later. The only measurement, in message #26, used empty tables with no pg_init_privs rows and runs that varied between 19 and 25 minutes, so it does not exercise the new path. A version gate would limit both the cost and the scope of the code kept in HEAD.

Thread: [#4 (Tom Lane, 2026-06-07)](https://www.postgresql.org/message-id/104812.1780847519%40sss.pgh.pa.us), [#14 (Tom Lane, 2026-06-22)](https://www.postgresql.org/message-id/592839.1782153054%40sss.pgh.pa.us), [#23 (Laurenz Albe, 2026-06-26)](https://www.postgresql.org/message-id/daee495807e183ea6f48c3c0844239753d7f9f6d.camel%40cybertec.at), [#26 (Laurenz Albe, 2026-07-02)](https://www.postgresql.org/message-id/ec8f6c8008cf88577b14489b68fef971f16960ab.camel%40aon.at), [#32 (solai v, 2026-08-07)](https://www.postgresql.org/message-id/CAF0whuft4HjO1NhMuir-fJ-mcAWbX5m4ZG7FuARcLuxJzgK2_Q%40mail.gmail.com)

**Question:** Why not apply SAFE_INITPRIVS only when the source server is older than v17, so HEAD dumps of modern servers skip the per-table subquery?

Counter-check: held. No visible patch version gates the filter on server version, although message #4 says v17 and later cannot have dangling rows. The cost part is likely overstated: as far as I know, pg_dump since v15 runs the column ACL query only for tables that have column ACLs.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | The fix changes what pg_upgrade restores; affected extension members lose their pg_init_privs row in the new cluster. |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | **yes** | pg_dump changes how it reads pg_init_privs and checks role OIDs against pg_roles; no catalog definition changes. |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | **yes** | pg_dump drops dangling aclitems, so later dumps of upgraded extension members re-emit REVOKE and GRANT lines. |
| psql UX conventions | no |  |
| security / privileges | **yes** | The patch decides which initial privileges survive a dump; it filters by OID against pg_roles to avoid a name-matching risk (messages #17, #18). |
| backpatchability | **yes** | The author proposes a backpatch to v15, but the CF target is 19 and the range is not agreed. |
| maintenance burden ("who maintains this in 10 years") | **yes** | The SAFE_INITPRIVS macro stays in two pg_dump query sites with no server version gate. |
| performance on the hot path | **yes** | The filter subquery is added to the per-table column ACL query in dumpTable(). |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Why does the filter drop an aclitem whose grantor alone is dangling, such as =r/16449, instead of keeping the grant with the object owner as grantor?
2. Why is SAFE_INITPRIVS not applied only when the source server is older than v17, given that message #4 says later servers cannot have dangling rows?
3. What is the pg_dump-only time with and without the patch on a database with many tables that have column ACLs and pg_init_privs rows?
4. Which branches should get the fix, and does a test run pg_dump against a v15 or v16 source server that has dangling rows?
5. Does the pg_dump fix depend on the HEAD-only putid() quoting change, and has that patch been posted?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Needs a committer who knows pg_dump's ACL and pg_init_privs handling. Tom Lane raised the cost concern and the putid() idea, but shows no view on v6 to v9.
- **Liability:** The owner signs for changed pg_dump output in back branches, for the loss of pg_init_privs rows on upgraded clusters, and for the filter staying in HEAD while pre-v17 source servers are supported.
- **Author follow-up:** The author posted nine versions over about four months and returned the entry to Needs review after each rebase and review round.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs champion
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch touches one pg_dump source file plus a test and fixes a hard pg_upgrade failure for clusters with extensions installed by later-dropped roles. Two reviewers report no open items, so the entry waits for a committer in this area to settle the two design points and the backpatch range.

## Appendix

### Refuted in the counter-check

- `demand-1` Every dump pays a per-table subquery for a rare broken case. Refuted (not specific): Duplicate of maintenance-fit-1 on the per-table cost. On desirability, the patch fixes a failed pg_upgrade and follows the pg_dump direction Tom Lane suggested in message #4; P10 was an optimization patch and does not apply.

### Review angles

- prior-attempts: 0 objection(s). The one close precedent, [P3], argued against the pg_upgrade pre-check that message #1 first proposed, and message #4 moved the work to pg_dump, so that reason does not apply to v9. Message #3 links an earlier thread on the pg_upgrade-check patch, and message #4 cites the role-GRANT fixes as the model, but the material does not record a rejection reason that would still apply.
- standards-compat: 1 objection(s). From the standards and compatibility angle, the patch adds no syntax, GUC or psql change. The silent filtering matches what Tom Lane cites for role grants in message #4. The putid() quoting change would alter aclitem text for clients, but it is a separate HEAD-only proposal and not judged here.
- replication-ondisk: 0 objection(s). The proposal changes only the SQL that pg_dump sends; it adds no WAL record, catalog column, on-disk format or protocol message. The one lasting effect, extension members losing their pg_init_privs row after upgrade, came up in message #33, was confirmed by a real upgrade in message #35 and is stated in the v9 commit message, so I do not raise it again.
- security: 0 objection(s). No security objection remains for v9. The only privilege risk in the thread was a CREATEROLE user creating a role named after a dangling OID to receive its grants (message #17), and v6 removed it by filtering on OID against pg_roles (message #18). pg_roles is readable by any role, so non-superuser pg_dump gains no new visibility, and the filter only removes items for roles that no longer exist.
- maintenance-fit: 1 objection(s). The fix itself fits core. It is a pg_dump-side repair of a real upgrade failure, a different approach from the pg_upgrade pre-check in [P3]. The putid() HEAD-only patch is separate and not judged here.
- already-exists: 0 objection(s). The only existing way is the workaround in message #1: delete pg_init_privs rows directly with allow_system_table_mods. Tom Lane (message #4) calls manual correction tedious and error-prone, and catalog surgery is not a documented idiom, so it does not meet the need. No contrib module or extension can repair pg_dump's own ACL output during pg_upgrade. The proposal follows the pg_dumpall precedent for dangling role grants, so it adds no second mechanism.
- demand: 1 objection(s). Demand is narrow but real. Failed upgrades from v15 and v16 clusters with extensions installed by later-dropped roles are a recorded bug, and Tom Lane proposed the pg_dump direction himself in message #4. The only demand-side concern is that all dump users pay for a fix that a few users need.

### Precedents retrieved

- P1: [Revert "Avoid race condition between "GRANT role" and "DROP ROLE"".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=8abbbbae610cd62faad4f9ec18e7cfa908593796) (revert, Reverted, 2025-09-16) - judged not relevant
- P2: [Implement the SQL-standard mechanism whereby REVOKE ROLE revokes only the privilege granted by the invoking role, and not those granted by other roles](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Administration) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P3: [pg_upgrade fails with non-standard ACL](https://commitfest.postgresql.org/patch/2215/) (cf, Returned with feedback, 2021-11-26) - used in the review; reason summarised from its thread
- P4: [pg_upgrade: Support for upgrading to checksums enabled](https://commitfest.postgresql.org/patch/5206/) (cf, Returned with feedback, 2025-04-03) - judged not relevant; reason summarised from its thread
- P5: [Revert MAINTAIN privilege and pg_maintain predefined role.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=151c22deee66a3390ca9a1c3675e29de54ae73fc) (revert, Reverted, 2023-07-07) - judged not relevant
- P6: [pg_dump -C Grant statement on create database](https://commitfest.postgresql.org/patch/724/) (cf, Returned with feedback, 2016-10-03) - judged not relevant; reason summarised from its thread
- P7: [DROP PRIVILEGES OWNED](https://commitfest.postgresql.org/patch/39/) (cf, Rejected, 2015-01-16) - judged not relevant; reason summarised from its thread
- P8: [Revert "Add basic TAP test setup for pg_upgrade"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=58ffe141eb37c3f027acd25c1fc6b36513bf9380) (revert, Reverted, 2017-09-22) - judged not relevant
- P9: [USAGE privilege on PUBLICATION](https://commitfest.postgresql.org/patch/3641/) (cf, Withdrawn, 2024-01-15) - judged not relevant
- P10: [pg_dump: optimize dumpFunc()](https://commitfest.postgresql.org/patch/5147/) (cf, Withdrawn, 2024-08-02) - used in the review; reason summarised from its thread
- P11: [pg_upgrade documentation improvement](https://commitfest.postgresql.org/patch/583/) (cf, Returned with feedback, 2016-10-03) - judged not relevant; reason summarised from its thread
- P12: [pg_upgrade: warn about roles with md5 passwords](https://commitfest.postgresql.org/patch/5792/) (cf, Withdrawn, 2025-06-04) - judged not relevant; reason summarised from its thread
- P13: [orphaned function](https://commitfest.postgresql.org/patch/2850/) (cf, Returned with feedback, 2021-04-08) - judged not relevant; reason summarised from its thread
- P14: [Add pg_ownerships and pg_privileges system views](https://commitfest.postgresql.org/patch/5033/) (cf, Withdrawn, 2025-01-04) - judged not relevant
- P15: [pg_upgrade: avoid failing if tablespace dirs exist and are empty](https://commitfest.postgresql.org/patch/2789/) (cf, Rejected, 2020-10-27) - judged not relevant
- P16: [Use role name "system_user" instead of "user" for unsafe_tests](https://commitfest.postgresql.org/patch/4271/) (cf, Withdrawn, 2023-04-13) - judged not relevant
- P17: [use has_privs_of_role() for pg_hba.conf](https://commitfest.postgresql.org/patch/3609/) (cf, Rejected, 2022-10-08) - judged not relevant
- P18: [Avoid orphaned dependencies](https://commitfest.postgresql.org/patch/3106/) (cf, Returned with feedback, 2022-08-02) - judged not relevant
- P19: [Orphaned users in PG16 and above can only be managed by Superusers](https://commitfest.postgresql.org/patch/5608/) (cf, Withdrawn, 2026-03-05) - judged not relevant
- P20: [Improve pg_dump dumping publication tables](https://commitfest.postgresql.org/patch/2728/) (cf, Withdrawn, 2021-01-15) - judged not relevant
- P21: [Add pg_privileges, pg_ownerships, and pg_get_acl() \[DUP\]](https://commitfest.postgresql.org/patch/5042/) (cf, Withdrawn, 2024-06-16) - judged not relevant
- P22: [Forbid use of LF and CR characters in database and role names](https://commitfest.postgresql.org/patch/711/) (cf, Rejected, 2017-03-29) - judged not relevant
- P23: [Revert b2a459edf "Fix GRANTED BY support in REVOKE ROLE statements"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e68570e388f08c2e36ce7d2a9564941b89db6549) (revert, Reverted, 2021-12-30) - judged not relevant
- P24: [pg_upgrade data type check connection overhead reduction](https://commitfest.postgresql.org/patch/4200/) (cf, Returned with feedback, 2024-02-01) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 15; wall time: 137 s; cost reported by the CLI: $3.27
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 4 from cache, 4 summarised now, 0 failed; quotes: 4 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 63 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:08:00.892Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

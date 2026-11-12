# Design pre-review: Allow pg_read_all_stats to read replication origin status

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7183](https://commitfest.postgresql.org/patch/7183/) (Needs review); authors: Virender Singla (vksingla)
- Thread: [Allow pg_read_all_stats to read replication origin status](https://www.postgresql.org/message-id/flat/CAM6Zo8ywTSsi7JHEYLTbyvJ-ukH2Mr%3DZ611Qf4rQLhX12NyFqQ%40mail.gmail.com), started 2026-08-08
- Messages read: 8 of 8
- Patch prose read: v2-0001-Grant-pg-read-all-stats-access-to-replication-origin.patch, 0001-Grant-pg-read-all-stats-access-to-replication-origin.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No objection survived against the goal or the demand. The one surviving objection, standards-compat-1, is fixable: pg_replication_origin_progress() would still deny the same LSN to the same role. The remaining choice between pg_read_all_stats and PUBLIC is a privilege-model decision, and no committer has spoken in the thread.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, and only 3 of 8 messages were shown. The exact grant statements and what the privileges.sql tests cover could not be checked.

## The proposal, restated

**Problem.** Non-superusers cannot read pg_replication_origin_status, and no predefined role (pg_read_all_stats, pg_read_all_data, pg_monitor) covers it. Monitoring users who need replication origin progress must today get an explicit grant or use a superuser.

**Proposed design.** system_views.sql grants select on the pg_replication_origin_status view and execute on its backing function pg_show_replication_origin_status() to pg_read_all_stats. Members of pg_monitor gain the same access through pg_read_all_stats. The patch adds regression tests in privileges.sql and changes no documentation.

**User-visible surface.**
- Members of pg_read_all_stats (and so pg_monitor) can select from pg_replication_origin_status.
- Members of pg_read_all_stats can execute pg_show_replication_origin_status().

**What the author claims.**
- The grant follows the existing practice for monitoring views in system_views.sql.
- pg_read_all_stats already covers point-in-time state views such as pg_shmem_allocations, pg_backend_memory_contexts and pg_aios, so it is not limited to cumulative statistics.
- An earlier thread removed the hardcoded superuser check but stalled before granting the view to pg_read_all_stats.
- Replication views use mixed access models, so a pg_read_all_stats grant is more conservative than a public grant.

**Thread state.** Hayato Kuroda asked whether the view is statistics at all and suggested granting it to PUBLIC instead; the author argued for pg_read_all_stats. Shubhra Jain tested v2, supports the grant and leaves PUBLIC to a committer; the author's 2026-11-03 question to Kuroda has no reply.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-08-08; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Same origin progress stays superuser-only through pg_replication_origin_progress()

fixable · level 2, rubric item 12 · design · `standards-compat-1`

A monitoring tool running as a pg_monitor member reads remote_lsn for origin pg_16401 from pg_replication_origin_status. It then calls pg_replication_origin_progress('pg_16401', false) for the same value and gets "permission denied". The operator finds the gap only when the tool's second query fails in production.

After this patch, pg_replication_origin_status and pg_replication_origin_progress() expose the same per-origin LSN under different privilege rules, and the patch says nothing about the function. The brief lists this as open, and no message in the thread discusses it. A committer would likely ask for one rule for the read-only origin functions. Grants on write functions such as pg_replication_origin_advance() should stay as they are.

Thread: [#1 (Virender Singla, 2026-08-08)](https://www.postgresql.org/message-id/CAM6Zo8ywTSsi7JHEYLTbyvJ-ukH2Mr%3DZ611Qf4rQLhX12NyFqQ%40mail.gmail.com), [#7 (Shubhra Jain, 2026-10-06)](https://www.postgresql.org/message-id/CAOh5eDUS77Q7FGG5tW9m5r9O0Z5ngzwthgQVFj7Jv8e5Tf2NOA%40mail.gmail.com)

**Question:** Should pg_replication_origin_progress() also be granted to pg_read_all_stats, and if not, why should the same LSN need different privileges?

Counter-check: held. Execute on pg_replication_origin_progress() is revoked from public by default, and the patch grants only the view and its backing function. The thread does not discuss the function, so the same LSN would fall under two privilege rules; the question is fair and a new patch version can answer it.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | **yes** | system_views.sql changes the default ACLs of pg_replication_origin_status and pg_show_replication_origin_status(). |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | **yes** | Members of pg_read_all_stats and pg_monitor gain read access to origin names and LSNs that are superuser-only by default today. |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | no |  |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Should pg_replication_origin_progress() also be granted to pg_read_all_stats, and if not, why should the same LSN need different privileges?
2. Why is pg_read_all_stats the right holder of this access instead of PUBLIC, beyond the mixed precedent among replication views?
3. Does pg_replication_origin_session_progress() need a matching grant, or should it stay as it is?
4. Should the pg_read_all_stats description in user-manag.sgml change to mention pg_replication_origin_status?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. A committer who knows the predefined-role privilege model and logical replication origins would have to choose between pg_read_all_stats and PUBLIC.
- **Liability:** The owner signs for a default-privilege decision on one view and one function. Widening to PUBLIC later is easy, but narrowing it would break monitoring setups.
- **Author follow-up:** The author rebased the patch in message #2 and was still answering reviewers on 2026-11-03.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs champion
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch adds two grants and tests in three files, and it helps monitoring users who track replication origins. It waits on a committer's choice between pg_read_all_stats and PUBLIC.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent found in the list. The thread reports an earlier discussion (messages #1, #4, #5) that removed the superuser check, then stalled before the grant; message #5 says it also proposed a PUBLIC grant. The thread records no rejection reason that would apply here.
- standards-compat: 1 objection(s). No standards or compatibility objection to the main idea. pg_read_all_stats matches how pg_stat_replication gates LSNs, pg_dump keeps user grants through pg_init_privs, and the role choice versus PUBLIC can still be widened later. The user-manag.sgml wording is a documentation detail.
- replication-ondisk: 0 objection(s). The patch only adds grants in system_views.sql, so it creates no new on-disk format, WAL record, wire message or extension API, and the catalog change is limited to ACLs on one view and one function. Standbys replay the catalog as usual, and pg_upgrade picks up the new default ACLs from initdb on the new cluster, so this angle shows no failure scenario.
- security: 0 objection(s). The grant is read-only and goes to pg_read_all_stats, an opt-in predefined role, so default access stays superuser-only. The view shows origin names, which pg_replication_origin already exposes publicly, and LSNs, which pg_replication_slots and pg_stat_subscription already show to PUBLIC. Message #7 confirms that a monitoring role gains no table data, and the functions that modify origins stay restricted.
- maintenance-fit: 0 objection(s). From the maintenance and fit angle there is nothing to object to. The patch adds two GRANT lines to system_views.sql in the same form as the existing pg_aios and pg_shmem_allocations grants, so it adds no hot-path cost, no new mechanism and no external dependency. Ten such grants per release would be acceptable, and moving the view to PUBLIC later would only widen access. The open choice between pg_read_all_stats and PUBLIC is a privilege-model question, and no committer has yet said which they prefer.
- already-exists: 0 objection(s). An admin can already grant select on pg_replication_origin_status and execute on pg_show_replication_origin_status() to a monitoring role, but predefined roles exist to remove those per-object grants, so that is no argument against this patch. pg_stat_subscription does not cover origins used outside built-in subscriptions, and the patch extends pg_read_all_stats instead of adding a new mechanism.
- demand: 0 objection(s). The patch adds two grants with almost no cost to non-users, and the request has come back over several years, as message #1 and messages #4 to #5 show. A plain grant works as a workaround, but asking for more demand evidence for a change this small would not hold up; the open question is which role gets the access, which another angle covers.

### Precedents retrieved

- P1: [Allow using replication origins in SQL level parallel sessions](https://commitfest.postgresql.org/patch/6035/) (cf, Withdrawn, 2025-09-16) - judged not relevant; reason summarised from its thread
- P2: [Add more TAP coverage for archive status with crash recovery of standbys](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ebf6de8692766177a36e7f5fb7545a52a0d5d881) (revert, Partially reverted, 2020-04-27) - judged not relevant
- P3: [Revert MAINTAIN privilege and pg_maintain predefined role.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=151c22deee66a3390ca9a1c3675e29de54ae73fc) (revert, Reverted, 2023-07-07) - judged not relevant
- P4: [Add document is_superuser](https://commitfest.postgresql.org/patch/3889/) (cf, Returned with feedback, 2023-12-02) - judged not relevant; reason summarised from its thread
- P5: [PGDOCS - inconsistent linkends to "monitoring" views](https://commitfest.postgresql.org/patch/4602/) (cf, Rejected, 2023-11-30) - judged not relevant; reason summarised from its thread
- P6: [Delegating superuser tasks to new security roles](https://commitfest.postgresql.org/patch/3223/) (cf, Withdrawn, 2021-11-15) - judged not relevant; reason summarised from its thread
- P7: [Improve monitoring of shared memory allocations](https://commitfest.postgresql.org/patch/5620/) (cf, Rejected, 2026-04-07) - judged not relevant; reason summarised from its thread
- P8: [Orphaned users in PG16 and above can only be managed by Superusers](https://commitfest.postgresql.org/patch/5608/) (cf, Withdrawn, 2026-03-05) - judged not relevant; reason summarised from its thread
- P9: [Revert b2a459edf "Fix GRANTED BY support in REVOKE ROLE statements"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e68570e388f08c2e36ce7d2a9564941b89db6549) (revert, Reverted, 2021-12-30) - judged not relevant
- P10: [Recognizing superuser in pg_hba.conf](https://commitfest.postgresql.org/patch/2397/) (cf, Returned with feedback, 2020-07-03) - judged not relevant; reason summarised from its thread
- P11: [Waits monitoring](https://commitfest.postgresql.org/patch/297/) (cf, Returned with feedback, 2015-12-24) - judged not relevant; reason summarised from its thread
- P12: [CLUSTER command progress monitor](https://commitfest.postgresql.org/patch/1268/) (cf, Returned with feedback, 2017-10-02) - judged not relevant
- P13: [Revert "Skip redundant anti-wraparound vacuums"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=dd9ac7d5d80608a640bb82cffb6a805ce84cf112) (revert, Reverted, 2020-03-30) - judged not relevant
- P14: [Allow placeholders in ALTER ROLE w/o superuser](https://commitfest.postgresql.org/patch/3749/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P15: [pg_dump -C Grant statement on create database](https://commitfest.postgresql.org/patch/724/) (cf, Returned with feedback, 2016-10-03) - judged not relevant
- P16: [Show subtransaction overflow status](https://commitfest.postgresql.org/patch/3444/) (cf, Rejected, 2022-03-22) - judged not relevant
- P17: [Revert "docs: change "default role" wording to "predefined role""](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=c185a57753e64fe3ffc996ea18da8672b302a5a4) (revert, Reverted, 2020-02-09) - judged not relevant
- P18: [\[WIP\] Show a human-readable n_distinct in pg_stats view](https://commitfest.postgresql.org/patch/2061/) (cf, Withdrawn, 2019-08-28) - judged not relevant
- P19: [Implement the SQL-standard mechanism whereby REVOKE ROLE revokes only the privilege granted by the invoking role, and not those granted by other roles](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Administration) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P20: [skip replication slot snapshot/map file removal during end-of-recovery checkpoint](https://commitfest.postgresql.org/patch/3491/) (cf, Withdrawn, 2022-01-10) - judged not relevant
- P21: [USAGE privilege on PUBLICATION](https://commitfest.postgresql.org/patch/3641/) (cf, Withdrawn, 2024-01-15) - judged not relevant
- P22: [Revert "Avoid race condition between "GRANT role" and "DROP ROLE"".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=8abbbbae610cd62faad4f9ec18e7cfa908593796) (revert, Reverted, 2025-09-16) - judged not relevant
- P23: [Allow granting of permission to REFRESH MATERIALIZED VIEW, CLUSTER, and REINDEX](https://commitfest.postgresql.org/patch/1605/) (cf, Returned with feedback, 2018-11-30) - judged not relevant
- P24: [Add %r substitution for psql prompts to show recovery status](https://commitfest.postgresql.org/patch/1402/) (cf, Returned with feedback, 2018-01-22) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 13; wall time: 77 s; cost reported by the CLI: $1.11
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 48 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:26:19.802Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

# Design pre-review: Preserve replication origin OIDs in pg_upgrade

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6743](https://commitfest.postgresql.org/patch/6743/) (Needs review); authors: Ajin Cherian (ajinc)
- Thread: [\[PATCH\] Preserve replication origin OIDs in pg_upgrade](https://www.postgresql.org/message-id/flat/CAFPTHDa5aP8ixTZBygGAe48E1N%3DDvuPzcXikw7bNKixLMN3pVg%40mail.gmail.com), started 2026-04-28
- Messages read: 90 of 90
- Patch prose read: v22-0001-Preserve-subscription-OIDs-during-pg_upgrade.patch, v22-0002-Preserve-replication-origin-IDs-during-pg_upgrad.patch, v2-0001-Preserve-replication-origin-OIDs-in-pg_upgrade.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

No surviving objection is fatal, and a committer who knows the area took part in the design. Three fixable points remain. demand-1: the main benefit depends on a separate pg_commit_ts migration whose status the thread does not give. replication-ondisk-2: upgrades from clusters before 17 are now refused in cases that pass on HEAD. replication-ondisk-1: the check that justifies those refusals can be bypassed while apply workers run. A committer has to decide the pre-17 behaviour change.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, this review could not check WAL logging of restored origin progress, locking in binary_upgrade_create_replication_origin(), or the binary-upgrade guard on the new functions. Only 3 of 90 messages were read directly.

## The proposal, restated

**Problem.** Commit timestamp records store the numeric roident, but pg_upgrade lets CreateSubscription() take the lowest free roident in the new cluster, so origins can get new or swapped IDs. If pg_commit_ts were carried over, this would cause spurious update_origin_differs conflicts. Origins not owned by a subscription, such as ones made by external tools, are lost today.

**Proposed design.** Patch 0001 preserves pg_subscription.oid, so origin names like pg_<suboid> stay the same. Patch 0002 has pg_dumpall dump all replication origins as global objects in binary-upgrade mode and recreates each one with its old roident and, if tracked, its remote_lsn. CreateSubscription() skips origin creation in binary-upgrade mode, and the subscription state checks now run for all old versions.

**User-visible surface.**
- New support function binary_upgrade_set_next_pg_subscription_oid().
- New support function binary_upgrade_create_replication_origin(oid, name, lsn); binary_upgrade_replorigin_advance() is removed.
- pg_dumpall --binary-upgrade emits replication origins, also with --globals-only.
- pg_upgrade rejects a new cluster that has any replication origins, and checks max_active_replication_origins against tracked origins instead of subscriptions.
- pg_upgrade from clusters before 17 now rejects subscriptions with a missing origin or relations not in 'i' or 'r' state.

**What the author claims.**
- This is a prerequisite for preserving pg_commit_ts across pg_upgrade.
- User-created origins are now kept with their IDs and remote_lsn.
- For old clusters before 17, enabled subscriptions resume from the migrated remote_lsn, and duplicates are no worse than on HEAD.
- Rejecting origin-less subscriptions from old clusters is harmless.

**Thread state.** Many reviewers, mostly from Fujitsu, reviewed the patch through v22, and Amit Kapila steered the design toward preserving IDs for all source versions. The latest change applies the subscription state checks to clusters before 17; Shveta agreed, and no committer has said the patch is ready.

## 2. Prior attempts

### Patch for migration of the pg_commit_ts directory (related)

- **Link:** [https://commitfest.postgresql.org/patch/5710/](https://commitfest.postgresql.org/patch/5710/)
- **Thread:** [https://www.postgresql.org/message-id/flat/182311743703924@mail.yandex.ru](https://www.postgresql.org/message-id/flat/182311743703924@mail.yandex.ru)
- **Date:** decided 2025-10-15; started 2025-04-03
- **Outcome:** Withdrawn (cf)
- **Who objected:** not recorded in the corpus
- **Reason (quote):** quote unavailable
- **What is different this time:** This proposal does not migrate pg_commit_ts. It preserves the roident values that such a migration needs, and it also keeps origins that are not owned by subscriptions.


## 3. Objections a reviewer is likely to raise

### 1. Main benefit depends on a pg_commit_ts migration not yet accepted

fixable · level 1b, rubric item 2 · demand, on-disk format / pg_upgrade, maintenance burden ("who maintains this in 10 years") · `demand-1`

A committer pushes both patches, but the pg_commit_ts migration does not land in the same release. Upgrades from clusters before 17 that worked before are now rejected for missing origins or relations not in 'i' or 'r' state. The roident preservation that justified this change protects no data in that release.

Messages #1 and #26, and the 0002 commit message, give one motivation: keeping roident stable so a copied pg_commit_ts stays valid. Amit Kapila notes in message #26 that the roident value only matters from PG19 onward. Message #2 says subscription OID preservation was declined before for lack of a strong motivation, and this patch borrows its motivation from a separate patch whose status the thread does not give. The other benefit, keeping origins made by external tools, has no user report in the thread.

Thread: [#1 (Ajin Cherian, 2026-04-28)](https://www.postgresql.org/message-id/CAFPTHDa5aP8ixTZBygGAe48E1N%3DDvuPzcXikw7bNKixLMN3pVg%40mail.gmail.com), [#2 ("Hayato Kuroda (Fujitsu)", 2026-04-29)](https://www.postgresql.org/message-id/OS9PR01MB12149E188221BED862E4BE9E0F5342%40OS9PR01MB12149.jpnprd01.prod.outlook.com), [#26 (Amit Kapila, 2026-06-15)](https://www.postgresql.org/message-id/CAA4eK1KzGSANd%3Dg1S8xpcyiwh3M-c%2BUaa4%2B447v-%2BN8chX7K3g%40mail.gmail.com), [#90 (Ajin Cherian, 2026-11-03)](https://www.postgresql.org/message-id/CAFPTHDYCdy%2BGXTOO8ApS9oXOFWXsh8uxbB4j%2BP_3wNp3A7DBZA%40mail.gmail.com)

**Question:** Is the pg_commit_ts migration patch agreed for the same release, or which users of external replication tools have reported losing origins in pg_upgrade?

Counter-check: held. The 0002 commit message calls the patch a prerequisite for pg_commit_ts migration, and the thread does not give that patch's status. Amit Kapila's engagement lowers the concern, but the thread shows no user report for the independent gain of keeping user-created origins.

### 2. New pg_upgrade refusals for pre-17 clusters where filtering would do

fixable · level 2, rubric item 12 · on-disk format / pg_upgrade, catalog changes · `replication-ondisk-2`

An operator upgrades a PG14 subscriber whose table sync is stuck in 'd' state, or one whose origin was dropped by hand. HEAD upgrades it and restores the subscription disabled. With v22 the check refuses the upgrade, and the operator must fix the old cluster first.

These refusals exist only to keep tablesync origins and origin-less subscriptions from causing trouble after migration (messages #88, #90). For pre-17 sources, pg_subscription_rel is not migrated and subscriptions come back disabled, so origins named pg_<suboid>_<relid> carry no state the new cluster can use. Skipping those origins at dump time would keep the current pre-17 behaviour. Message #90 calls the refusal harmless, but no committer has weighed it, and the patch has no pre-17 upgrade test.

Thread: [#83 (Nisha Moond, 2026-10-07)](https://www.postgresql.org/message-id/CABdArM7iWpoY%3DZM3Bsbbgwc3N5RbP6asS%2BZYyRzZtfC3tik4mA%40mail.gmail.com), [#88 ("Hayato Kuroda (Fujitsu)", 2026-10-30)](https://www.postgresql.org/message-id/OS7PR01MB1831748150E960602D81EE47CF56D2%40OS7PR01MB18317.jpnprd01.prod.outlook.com), [#89 (shveta malik, 2026-11-03)](https://www.postgresql.org/message-id/CAJpy0uD3iXD7Qf%2BBkBosByOW3D_nQ-%2B89J%2BQ0jhwoCmsRL6uRA%40mail.gmail.com), [#90 (Ajin Cherian, 2026-11-03)](https://www.postgresql.org/message-id/CAFPTHDYCdy%2BGXTOO8ApS9oXOFWXsh8uxbB4j%2BP_3wNp3A7DBZA%40mail.gmail.com)

**Question:** Why reject pre-17 clusters in these states instead of leaving out tablesync origins and recreating a missing subscription origin, as HEAD does today?

Counter-check: held. The v22 commit message confirms that upgrades from clusters before 17 are now refused in cases that pass on HEAD. Amit Kapila's argument in message #26 covered restrictions on the new cluster, and the thread does not discuss skipping tablesync origins at dump time.

### 3. Tablesync origin leak check is racy for old clusters before 17

fixable · level 2, rubric item 14 · on-disk format / pg_upgrade, WAL / replication / standby, catalog changes · `replication-ondisk-1`

A user upgrades a PG16 subscriber that has a table in 'i' state, which the check accepts. While pg_upgrade runs, the old apply worker launches a tablesync worker, which creates origin pg_<suboid>_<relid> before pg_dumpall reads pg_replication_origin. That origin is migrated, and later DROP SUBSCRIPTION leaves it behind.

Message #85 states that apply workers can run during pg_upgrade when the source is PG16 or older. Message #88 adds the state check because migrated tablesync origins cannot be dropped once pg_subscription_rel is not carried over. The v22 commit message allows relations in 'i' state. A check taken before the dump does not bound what the dump later sees while workers keep running.

Thread: [#85 ("Hayato Kuroda (Fujitsu)", 2026-10-15)](https://www.postgresql.org/message-id/OS7PR01MB18317CFD00457D42BDBE4B841F59C2%40OS7PR01MB18317.jpnprd01.prod.outlook.com), [#86 (Ajin Cherian, 2026-10-16)](https://www.postgresql.org/message-id/CAFPTHDai3JOZ%3Du%2Bw0MPhZ4kxrBiLxh%3DH%3D8ZW9ZdqeT6NVpja_w%40mail.gmail.com), [#88 ("Hayato Kuroda (Fujitsu)", 2026-10-30)](https://www.postgresql.org/message-id/OS7PR01MB1831748150E960602D81EE47CF56D2%40OS7PR01MB18317.jpnprd01.prod.outlook.com), [#90 (Ajin Cherian, 2026-11-03)](https://www.postgresql.org/message-id/CAFPTHDYCdy%2BGXTOO8ApS9oXOFWXsh8uxbB4j%2BP_3wNp3A7DBZA%40mail.gmail.com)

**Question:** For sources before 17, what stops a tablesync origin from being created between check_old_cluster_subscription_state() and pg_dumpall, or would it be safer not to dump pg_<suboid>_<relid> origins from such sources?

Counter-check: held. Message #85 says apply workers can run during pg_upgrade when the source is before 17, and the check accepts relations in 'i' state. The thread does not say what stops a tablesync origin from appearing between the check and the dump.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | pg_upgrade now preserves pg_subscription.oid and roident, and adds new checks on old and new clusters. |
| WAL / replication / standby | **yes** | Replication origins and their remote_lsn are migrated, which changes where pre-17 subscriptions resume. |
| wire protocol | no |  |
| catalog changes | **yes** | pg_proc.dat gains two binary-upgrade support functions and loses binary_upgrade_replorigin_advance(). |
| ABI / extension API | **yes** | The patch changes src/include/replication/origin.h and removes an existing support function. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | **yes** | pg_dumpall --binary-upgrade emits all replication origins as global objects, and pg_dump emits the subscription OID. |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | Upgrades from released versions before 17 are now rejected in states that HEAD accepts. |
| maintenance burden ("who maintains this in 10 years") | **yes** | Origin migration relies on the pg_<suboid> and pg_<suboid>_<relid> naming conventions and on pg_upgrade checks. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Is the pg_commit_ts migration patch agreed for the same release, and if it does not land, which users need preserved roident values?
2. Why reject pre-17 clusters with relations not in 'i' or 'r' state instead of leaving pg_<suboid>_<relid> origins out of the dump for such sources?
3. Why reject a pre-17 subscription with a missing origin instead of creating a new origin for it, as HEAD does today?
4. For sources before 17, what stops an apply worker from creating a tablesync origin between check_old_cluster_subscription_state() and pg_dumpall?
5. Which external replication tools or users have reported that pg_upgrade loses their replication origins?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Amit Kapila steered the design in messages #26 and #51, but has not said the patch is ready. The owner needs to know logical replication origins and pg_upgrade of subscribers.
- **Liability:** The owner signs for pg_upgrade of subscribers from every supported source version, including the new pre-17 refusals. The owner also signs for the roident guarantee that a later pg_commit_ts migration would depend on.
- **Author follow-up:** The authors posted 22 versions in about six months and kept the patch rebased across three CommitFests.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** The patch helps subscribers that use pg_upgrade and need stable origin IDs, and users of external tools that create origins. It runs only in binary-upgrade mode, spans pg_dump, pg_upgrade and origin.c, and has an engaged committer and active authors.

## Appendix

### Refuted in the counter-check

- `prior-attempts-1` Main motivation rests on a pg_commit_ts migration that is not yet accepted. Refuted (not specific): Duplicate of demand-1, which states the dependency on the pg_commit_ts migration without a precedent. [P11] records no reason for its withdrawal, so it cannot support the point.
- `standards-compat-1` Upgrades from pre-17 clusters now fail where they used to succeed. Refuted (not specific): Duplicate of replication-ondisk-2, which covers both new refusals. Message #88 calls only the origin-less rejection harmless, and the material does not give the affiliation of every reviewer who agreed.
- `replication-ondisk-3` Main invariant only matters if pg_commit_ts migration is accepted. Refuted (not specific): Duplicate of demand-1. The restrictions it calls permanent are pg_upgrade checks and binary-upgrade support functions, which a later release can relax or remove.
- `maintenance-fit-1` Origin migration depends on naming conventions that future workers can break silently. Refuted (no failure scenario): The scenario needs a hypothetical future worker whose cleanup state is not migrated. The tablesync case arose only because sources before 17 do not migrate pg_subscription_rel; with subscription OIDs preserved, name-based links stay valid for newer sources.
- `maintenance-fit-2` Permanent roident invariant for a pg_commit_ts migration not yet accepted. Refuted (not specific): Duplicate of demand-1. The claim that the guarantees are hard to take back is weak, because the binary-upgrade support functions are internal and are not a user-facing interface.
- `maintenance-fit-3` Upgrades from clusters before 17 now fail where they succeeded. Refuted (not specific): Duplicate of replication-ondisk-2. The statement that only Fujitsu reviewers agreed is not supported by the material for every reviewer.

### Review angles

- prior-attempts: 1 objection(s). The thread also reports, in message #2, an earlier attempt to preserve subscription OIDs that stalled for lack of motivation; it is not in the precedent list. [P19] concerns a buildfarm-flaky test for relfilenode preservation, and its reason does not apply here.
- standards-compat: 1 objection(s). Other contract changes were settled in the thread. The empty-new-cluster rule was discussed in message #26, the resume position in message #86, and the max_active_replication_origins check in messages #51-#71.
- replication-ondisk: 3 objection(s). Objections found; see list. Not checked without code: WAL logging of restored origin progress, and catalog locking in the new support function.
- security: 0 objection(s). The proposal has no real privilege or exposure aspect. The new binary_upgrade_* functions run only in binary-upgrade mode, pg_dumpall output already requires superuser, and replication origins are superuser-managed objects today. The quoting of origin names in the dump was raised in message #9 and fixed in message #10. Whether the new support functions keep the usual binary-upgrade guard cannot be checked without the code.
- maintenance-fit: 3 objection(s). No hot-path cost was found: all changes run in pg_upgrade or binary-upgrade mode. Amit Kapila, a committer in this area, engaged in the design (messages #26 and #51), which lowers the item 16 concern.
- already-exists: 0 objection(s). Postgres has no existing way to choose a roident or a subscription OID, and pg_replication_origin_create() always takes the lowest free ID, so this cannot be done outside pg_upgrade today. Recreating user origins by hand after the upgrade restores names and remote_lsn but not their IDs, and no extension or hook covers binary-upgrade restore.
- demand: 1 objection(s). Only one demand concern: the value depends on a separate pg_commit_ts patch. The case for user-created origins has no field reports, but it is a plausible correctness gain at low cost.

### Precedents retrieved

- P1: [pg_upgrade: Support for upgrading to checksums enabled](https://commitfest.postgresql.org/patch/5206/) (cf, Returned with feedback, 2025-04-03) - judged not relevant; reason summarised from its thread
- P2: [pg_upgrade documentation improvement](https://commitfest.postgresql.org/patch/583/) (cf, Returned with feedback, 2016-10-03) - judged not relevant; reason summarised from its thread
- P3: [add timing information to pg_upgrade](https://commitfest.postgresql.org/patch/4464/) (cf, Withdrawn, 2023-12-02) - judged not relevant; reason summarised from its thread
- P4: [Allow using replication origins in SQL level parallel sessions](https://commitfest.postgresql.org/patch/6035/) (cf, Withdrawn, 2025-09-16) - judged not relevant; reason summarised from its thread
- P5: [pg_upgrade version and path checking](https://commitfest.postgresql.org/patch/2095/) (cf, Returned with feedback, 2019-09-30) - judged not relevant; reason summarised from its thread
- P6: [pg_upgrade: remove C comment](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=84a6f63e32dbefe3dc76cbe628fab6cbfc26141e) (revert, Partially reverted, 2018-01-05) - judged not relevant
- P7: [Preserve versions of initdb-created collations in pg_upgrade](https://commitfest.postgresql.org/patch/2328/) (cf, Returned with feedback, 2020-01-24) - judged not relevant; reason summarised from its thread
- P8: [pg_upgrade: Pass -j option down to vacuumdb](https://commitfest.postgresql.org/patch/1943/) (cf, Returned with feedback, 2019-04-09) - judged not relevant; reason summarised from its thread
- P9: [Improve documentation for pg_upgrade, standbys and rsync](https://commitfest.postgresql.org/patch/3125/) (cf, Withdrawn, 2022-04-08) - judged not relevant; reason summarised from its thread
- P10: [Rewriting the test of pg_upgrade as a TAP test](https://commitfest.postgresql.org/patch/1101/) (cf, Returned with feedback, 2017-09-29) - judged not relevant
- P11: [Patch for migration of the pg_commit_ts directory](https://commitfest.postgresql.org/patch/5710/) (cf, Withdrawn, 2025-10-15) - used in the review
- P12: [speed up a logical replication setup (pg_subscriber)](https://commitfest.postgresql.org/patch/3556/) (cf, Returned with feedback, 2022-08-02) - judged not relevant
- P13: [Do not start bgworker processes during binary upgrade](https://commitfest.postgresql.org/patch/3299/) (cf, Returned with feedback, 2022-03-14) - judged not relevant
- P14: [Pg14, pg_dumpall and "password_encryption=true"](https://commitfest.postgresql.org/patch/2939/) (cf, Withdrawn, 2021-03-04) - judged not relevant
- P15: [speed up a logical replication setup (pg_subscriber)](https://commitfest.postgresql.org/patch/4636/) (cf, Withdrawn, 2023-11-01) - judged not relevant
- P16: [New strategies for freezing, advancing relfrozenxid early](https://commitfest.postgresql.org/patch/3843/) (cf, Withdrawn, 2023-01-26) - judged not relevant
- P17: [preserve timestamps when installing headers](https://commitfest.postgresql.org/patch/3363/) (cf, Returned with feedback, 2022-02-07) - judged not relevant
- P18: [Time-delayed logical replication subscriber](https://commitfest.postgresql.org/patch/3581/) (cf, Returned with feedback, 2023-03-10) - judged not relevant
- P19: [Revert recent changes to 002_pg_upgrade.pl.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=87e22f675fd81ba1d96b0b9a34bbf26d5ec532d5) (revert, Reverted, 2022-08-04) - used in the review
- P20: [Revert Non text modes for pg_dumpall, and pg_restore support](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ce9a6244b5b4ce1df71611512a757353803404a5) (revert, Reverted, 2025-07-30) - judged not relevant
- P21: [pg_dumpall --exclude-database option](https://commitfest.postgresql.org/patch/1743/) (cf, Returned with feedback, 2018-10-02) - judged not relevant
- P22: [Warning for Datatype mismatch between subscriber table and publisher table for logical replication](https://commitfest.postgresql.org/patch/5648/) (cf, Returned with feedback, 2025-05-13) - judged not relevant
- P23: [Teach autovacuum.c to launch workers to advance table age without attendant antiwraparound cancellation behavior](https://commitfest.postgresql.org/patch/4027/) (cf, Withdrawn, 2023-01-26) - judged not relevant
- P24: [Add function pg_advance_replication_slot](https://commitfest.postgresql.org/patch/1128/) (cf, Returned with feedback, 2017-11-29) - judged not relevant

### Warnings

- synthesis said NEEDS-COMMITTER-EYES but listed a level 1b objection (demand-1); changed to NEEDS-DESIGN-DISCUSSION

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 13; wall time: 165 s; cost reported by the CLI: $6.26
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 93 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:01:43.609Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

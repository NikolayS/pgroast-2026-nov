# Design pre-review: Check output_plugin_libraries in pg_createsubscriber

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7271](https://commitfest.postgresql.org/patch/7271/) (Needs review); authors: Hayato Kuroda (ha-kun)
- Thread: [pg_createsubscriber does not check output_plugin_libraries](https://www.postgresql.org/message-id/flat/OS9PR01MB12149618ED793133C85A61A34F5B52%40OS9PR01MB12149.jpnprd01.prod.outlook.com), started 2026-09-04
- Messages read: 46 of 46
- Patch prose read: v2-0001-doc-clarify-the-location-where-output-plugin-must.patch, v1-0001-Reproduce-missing-output_plugin_libraries-issue.patch, v1-0002-pg_createsubscriber-ensure-output_plugin_librarie.patch, v1-0003-free-allocated-memory-for-max_slot_wal_keep_size.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No objection survived at levels 1a or 1b, and the goal has support from two committers in the thread. The one surviving objection, standards-compat-1, is a fixable design point: the frontend parse of output_plugin_libraries can stop the tool on a value the server accepts today. A committer who knows this code can settle that point, together with the backpatch question.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The v7 code patch prose is not in the material, so the final error text, the skip conditions and the parse-failure branch could not be checked. Whether the frontend and the server match a quoted or path-qualified pgoutput entry the same way is also unchecked.

## The proposal, restated

**Problem.** pg_createsubscriber always creates slots with pgoutput but does not check the publisher's output_plugin_libraries setting, added by commit 226e49cb. If a DBA removes pgoutput from that list, --dry-run succeeds and the real run fails at slot creation. The affected users run pg_createsubscriber against a publisher with a narrowed plugin list.

**Proposed design.** The patch adds a check on the publisher that reads output_plugin_libraries from pg_settings once per database given with --database, because database and role settings can change the effective value. If pgoutput is missing, the tool fails early with an error and a hint. If the setting cannot be read (missing on an older minor, or no privilege), the check is skipped silently.

**User-visible surface.**
- New pg_createsubscriber error: publisher does not allow the "pgoutput" output plugin in database, with a hint to add it to output_plugin_libraries.
- New prerequisite in the pg_createsubscriber docs: the effective output_plugin_libraries for each source database connection must include pgoutput.
- --dry-run now fails when pgoutput is not allowed in any target database.
- Separate master-only doc patch adds a section "Installing Logical Decoding Output Plugins" to logicaldecoding.sgml.

**What the author claims.**
- --dry-run passing while the real run fails is surprising to users.
- The gap is an oversight in 226e49cb and could be backpatched to v17 and v18.
- A per-database check is needed because ALTER DATABASE SET can change the value.
- A separate patch frees the memory pg_strdup'd for max_slot_wal_keep_size.

**Thread state.** Fujii Masao, Jacob Champion and João Rito favour master only and João Rito tested v7 and supports Ready for Committer; the author still questions the backpatch decision. The CF entry shows only the doc patch, so cfbot does not test the v7 code patch.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-09-04; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Client-side list parsing can reject settings the server accepts

fixable · level 2, rubric item 12 · design, maintenance burden ("who maintains this in 10 years") · `standards-compat-1`

A DBA sets output_plugin_libraries = 'pgoutput, test_decoding, ' on the publisher. The server accepts this value and pgoutput slots work. With the patch, pg_createsubscriber stops with a fatal parse error in --dry-run and in the real run, so a conversion that worked before now fails.

The patch re-parses a server GUC in the frontend with SplitGUCList, so the tool's view of the setting can differ from the server's. Message #29 shows that the trailing-comma value reaches the parse-failure branch. Message #31 calls the server behaviour a bug and plans a fix, but the material does not say whether that fix lands in the same release. The tool also turns a parse failure into a hard error, though the run would succeed today.

Thread: [#20 (Jacob Champion, 2026-09-14)](https://www.postgresql.org/message-id/CAOYmi%2BnFptx-1xTxMqatHgRuLAc3JKZo--FzXZvKcrpCjBc%3DBQ%40mail.gmail.com), [#29 (Fujii Masao, 2026-09-16)](https://www.postgresql.org/message-id/CAHGQGwG81iHN46mY_19Bne5RuKfAjH4dOnQ7-8i%3D7KYJk_0rAg%40mail.gmail.com), [#30 ("Hayato Kuroda (Fujitsu)", 2026-09-16)](https://www.postgresql.org/message-id/TYRPR01MB128459BA258A78D03C5365B2EF5B92%40TYRPR01MB12845.jpnprd01.prod.outlook.com), [#31 (Jacob Champion, 2026-09-16)](https://www.postgresql.org/message-id/CAOYmi%2Bkrun4a8ACxcP4MtvPU6SMrowwENSvh7yzc5mhEJ85k_g%40mail.gmail.com)

**Question:** Will the server-side fix that rejects empty list elements land before or with this patch, and if not, why should a parse failure stop the tool instead of skipping the check?

Counter-check: held. Message #29 confirms that a trailing-comma value set on the server reaches the pg_fatal branch, and v7 keeps that branch. Message #31 plans a server fix but gives no schedule, so the regression for such a setting is unanswered; the severity stays fixable.

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
| security / privileges | **yes** | The tool reads a security allow list through pg_settings and skips the check silently when the role cannot see it. |
| backpatchability | **yes** | The CF entry is tagged Backport and the author proposes v17 and v18, while three reviewers favour master only. |
| maintenance burden ("who maintains this in 10 years") | **yes** | pg_createsubscriber.c parses the list with SplitGUCList, so frontend and server parsing must stay compatible. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Will the server fix that rejects values like 'pgoutput, test_decoding, ' land with this patch, and if not, why should a parse failure stop the tool instead of skipping the check?
2. Is the patch now master only, or does the author still ask for a backpatch to v17 and v18?
3. Should the tool report a skipped check, for example at debug level with -v, when it cannot read output_plugin_libraries?
4. Does the frontend match pgoutput the same way the server does when the list entry is quoted or carries a path?
5. Can the CF entry carry the v7 code patch so that cfbot tests it, since the entry now shows only the doc patch?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Fujii Masao and Jacob Champion take part in the thread; the owner needs to know pg_createsubscriber and the output_plugin_libraries setting from commit 226e49cb.
- **Liability:** The owner signs for a frontend copy of the server's list parsing in pg_createsubscriber.c, which must follow any later server change to that setting. The liability is small and stays inside one tool.
- **Author follow-up:** The author answered review comments the same day in message #3 and revised the patch up to v7.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** Only users who run pg_createsubscriber against a publisher with pgoutput removed from the list gain, but the patch is small and touches one tool. Two committers are engaged and a reviewer tested v7.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent found: the precedent list has no earlier attempt at this idea. The thread cites no earlier discussion of it. Message #12 links a commit that let pg_createsubscriber reuse existing publications, and that commit is context only. No earlier rejection reason applies here.
- standards-compat: 1 objection(s). Other compatibility points are settled in the thread. The patch is now master only, unreadable settings skip the check, and only users who removed pgoutput see new --dry-run failures, which their real run would hit anyway.
- replication-ondisk: 0 objection(s). The proposal changes no persistent or cross-node state: no on-disk format, WAL record, catalog, wire protocol message or extension ABI. pg_createsubscriber only reads output_plugin_libraries from pg_settings over its existing publisher connections, and the guc_parameters.dat edit is a comment.
- security: 0 objection(s). The check adds no privilege or exposure. It reads output_plugin_libraries through pg_settings, which returns a row only to roles that can already see the setting (superuser or pg_read_all_settings), and it prints the value only to the user who ran that query. Server permission rules and the pg_hba path are unchanged.
- maintenance-fit: 0 objection(s). From the maintenance angle the cost is small and stays inside pg_createsubscriber. Users who never run the tool pay nothing, and the only runtime cost is one extra connection and query per --database. The client parses the setting the same way pg_upgrade/check.c already does, and a guc_parameters.dat comment records that dependency, so it follows existing practice. Two committers who know the area, Fujii Masao and Jacob Champion, are engaged (messages #17, #29, #31). Jacob Champion's reply in message #13 leaves open what --dry-run should check, but Fujii Masao answers it in message #17 with a workable rule: add the checks that are simple to perform. If the server later rejects values such as 'test_decoding, ' (message #31), the client parse-error branch could lose its purpose, but it would not fail and would not mislead users. Without the code, I could not check whether a value that names pgoutput with a path or with quoting is matched the same way on the client and on the server.
- already-exists: 0 objection(s). No existing feature, option or extension checks output_plugin_libraries for pg_createsubscriber before slot creation, and the check extends the existing check_publisher() path rather than adding a second mechanism. A manual SHOW in each source database is possible, but a committer would not treat that as a real alternative for a --dry-run check that two committers in the thread already support.
- demand: 0 objection(s). There are no field reports, and the failure needs a DBA to remove pgoutput from its default list (message #11), so the user group is small. The cost is small too and falls only on pg_createsubscriber users: one extra connection and query per database. Two committers support the check for master (messages #17, #29, #31), and a 'show demand' objection would not hold up against that.

### Precedents retrieved

- P1: [pglogical_output - a general purpose logical decoding output plugin](https://commitfest.postgresql.org/patch/418/) (cf, Returned with feedback, 2016-02-03) - judged not relevant; reason summarised from its thread
- P2: [pg_createsubscriber - more logging to say if there are no pubs to drop](https://commitfest.postgresql.org/patch/6130/) (cf, Withdrawn, 2025-10-15) - judged not relevant; reason summarised from its thread
- P3: [Revert "pg_createsubscriber: Add log message when no publications exist to drop."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=45a7faf130176ae7c152413967c7c90068476a94) (revert, Reverted, 2025-10-15) - judged not relevant
- P4: [Revert "Add GUC checks for ssl_min_protocol_version and ssl_max_protocol_version"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=414c2fd1e1c0ccd1af1cad046bef1264e3f4347f) (revert, Reverted, 2020-02-06) - judged not relevant
- P5: [Revert "Add TAP test to automate the equivalent of check_guc"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=52377bb81a2c0427d357bba71e7149de02d016ce) (revert, Reverted, 2022-02-12) - judged not relevant
- P6: [This adds a new GUC parameter that automatically rolls back prepared transactions that have remained unresolved beyond the configured timeout. The cleanup is performed by the checkpointer process.](https://commitfest.postgresql.org/patch/6611/) (cf, Rejected, 2026-07-01) - judged not relevant; reason summarised from its thread
- P7: [Reuse Workers and Replication Slots during Logical Replication](https://commitfest.postgresql.org/patch/3784/) (cf, Returned with feedback, 2024-01-10) - judged not relevant; reason summarised from its thread
- P8: [Trigger failsafe when VACUUM is run in single user mode](https://commitfest.postgresql.org/patch/3467/) (cf, Returned with feedback, 2022-04-07) - judged not relevant; reason summarised from its thread
- P9: [Allow logical decoding to output generated columns](https://commitfest.postgresql.org/patch/5068/) (cf, Returned with feedback, 2024-12-10) - judged not relevant; reason summarised from its thread
- P10: [Fix bug when calling strncmp in check_authmethod_valid](https://commitfest.postgresql.org/patch/4960/) (cf, Withdrawn, 2024-04-30) - judged not relevant; reason summarised from its thread
- P11: [Revert non-text output formats for pg_dumpall](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=7ca548f23a60887541493b2da47f2b2720c35ee3) (revert, Reverted, 2026-06-18) - judged not relevant
- P12: [allow building trusted languages without the untrusted versions](https://commitfest.postgresql.org/patch/3647/) (cf, Returned with feedback, 2022-08-01) - judged not relevant; reason summarised from its thread
- P13: [history of vacuum / checkpoint runs](https://commitfest.postgresql.org/patch/5462/) (cf, Withdrawn, 2025-11-14) - judged not relevant
- P14: [Synchronize logical replication slots from primary to standby](https://commitfest.postgresql.org/patch/3393/) (cf, Returned with feedback, 2022-02-18) - judged not relevant
- P15: [document an example to set multiple libraries in shared_preload_libraries](https://commitfest.postgresql.org/patch/3437/) (cf, Withdrawn, 2022-01-10) - judged not relevant
- P16: [Use indexes on the subscriber when REPLICA IDENTITY is full on the publisher](https://commitfest.postgresql.org/patch/3926/) (cf, Returned with feedback, 2022-10-20) - judged not relevant
- P17: [skip replication slot snapshot/map file removal during end-of-recovery checkpoint](https://commitfest.postgresql.org/patch/3491/) (cf, Withdrawn, 2022-01-10) - judged not relevant
- P18: [Warning for Datatype mismatch between subscriber table and publisher table for logical replication](https://commitfest.postgresql.org/patch/5648/) (cf, Returned with feedback, 2025-05-13) - judged not relevant
- P19: [warn if GUC set to an invalid shared library](https://commitfest.postgresql.org/patch/3482/) (cf, Returned with feedback, 2024-12-11) - judged not relevant
- P20: [Revert "Blind attempt to fix _configthreadlocale() failures on MinGW."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a62d90f2e5cb55d0c8998b63120d6db3c55df866) (revert, Reverted, 2024-11-27) - judged not relevant
- P21: [Revert "Add key management system" (978f869b99) & later commits](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=3187ef7c46c5b884267a88f2d6119c9a05f1bbba) (revert, Reverted, 2020-12-28) - judged not relevant
- P22: [Track invalidation_reason in pg_replication_slots.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6ae701b4378db2284c77314560e95a93d0ba9484) (revert, Partially reverted, 2024-03-22) - judged not relevant
- P23: [Output affected rows in EXPLAIN](https://commitfest.postgresql.org/patch/4551/) (cf, Rejected, 2023-11-22) - judged not relevant
- P24: [always use runtime checks for CRC-32C instructions](https://commitfest.postgresql.org/patch/4634/) (cf, Withdrawn, 2023-10-31) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 13; wall time: 105 s; cost reported by the CLI: $2.49
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 51 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:34:47.260Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

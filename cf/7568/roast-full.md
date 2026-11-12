# Design pre-review: pg_upgrade doesn't copy ICU rules on template0

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7568](https://commitfest.postgresql.org/patch/7568/) (Needs review); authors: Bohyun Lee (leebohyun)
- Thread: [pg_upgrade doesn't copy ICU rules on template0](https://www.postgresql.org/message-id/flat/8c65df31-345e-429c-98fa-867f08d703b6%40iki.fi), started 2026-10-28
- Messages read: 7 of 7
- Patch prose read: v1-pg_upgrade-template0-icu-rules.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No objection survived at levels 1a or 1b: the bug is stated, a user reported it, and the fix completes what pg_upgrade already does for template0's other locale fields. The one surviving objection, maintenance-fit-1, is a fixable level 2 point about patch shape: v1 puts the daticurules fix and a rewrite of shared query and logging helpers in one commit that is requested for backpatch to 16. A committer has to decide what goes to stable branches.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, this review could not check that the source-version checks still handle old clusters before 16, or that the StringInfo change keeps the helpers' behaviour for other callers. The CREATE DATABASE patch in message #7 was not read.

## The proposal, restated

**Problem.** Since 16, pg_upgrade copies the old template0's encoding and locale fields into the new template0, but it skips daticurules. A cluster initialized with --icu-rules loses those rules on template0 after an upgrade. This affects users who set ICU rules at initdb time.

**Proposed design.** set_locale_and_encoding() adds daticurules to its existing UPDATE of template0, with the value escaped by PQescapeLiteral(). The query and logging buffers in pg_upgrade switch to StringInfo so they grow for long rules. The patch also removes the branches on target cluster version, since pg_upgrade only supports its own major version as target.

**User-visible surface.**
- pg_upgrade now carries template0's daticurules into the new cluster.
- pg_upgrade error and log messages no longer truncate long queries or long rules.
- Separate patch in message #7: CREATE DATABASE with a non-ICU provider no longer inherits ICU rules from the template.

**What the author claims.**
- The omission was an oversight in commit 30a53b792959, which added ICU rules.
- Using executeQueryOrDie() keeps the full query with inlined values in error reports.
- The target-version branches in set_locale_and_encoding() are unnecessary.
- A reporter tested v1 on master and found pg_upgrade works.
- The fix needs a backport to 16; the CREATE DATABASE fix needs a backport to 17.

**Thread state.** Tom Lane first questioned how template0 can change, and Shihao explained that pg_upgrade itself updates it. Heikki agreed the fix is right but asked why template0 keeps locale fields at all, and found a CREATE DATABASE bug that Shihao addressed in a second patch.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-10-28; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Back-branch fix bundles unrelated pg_upgrade refactoring

fixable · level 2, rubric item 9 · backpatchability, maintenance burden ("who maintains this in 10 years") · `maintenance-fit-1`

A committer backpatches v1 to 16 as one unit. It brings the StringInfo rewrite of the query and logging buffers in util.c and server.c, the errno change and the removal of the target-version branches. A regression in pg_log() or executeQueryOrDie() then reaches every pg_upgrade run on stable branches, not only clusters that use ICU rules.

The bug needs one more column in an existing UPDATE, but v1 also changes the shared logging and query helpers that every pg_upgrade step uses. The commit message lists all of it as one change, and message #5 asks for a backport to 16 without saying which parts go back. Heikki in message #1 suggested growing the buffers as a possible fix, but he expected ICU rules to stay short. Keeping the general refactoring on master only would limit what stable branches take on.

Thread: [#1 (Heikki Linnakangas, 2026-10-28)](https://www.postgresql.org/message-id/8c65df31-345e-429c-98fa-867f08d703b6%40iki.fi), [#4 (Bohyun Lee, 2026-10-29)](https://www.postgresql.org/message-id/CAMPh8Mpz%3DVNv1kxDL%2BY3T4Ec_poGaFo3ZvVcEHq7-aHOQGt2kw%40mail.gmail.com), [#5 (shihao zhong, 2026-10-30)](https://www.postgresql.org/message-id/CAGRkXqRE2%3DXsW9xzP4%3D3LEk9bWTwXWcTRoFuOAF935oTpfAGSw%40mail.gmail.com)

**Question:** Can the series be split into a minimal daticurules fix for 16 and later, plus the buffer and branch cleanup for master only, or does the fix depend on the buffer change?

Counter-check: held. The v1 commit message puts the daticurules fix, the StringInfo buffer change, the errno change and the branch removal in one commit, and message #5 asks for a backport to 16. The thread does not say which parts go to stable branches.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | The patch changes what pg_upgrade writes into the new cluster's template0 row; no on-disk format changes. |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | **yes** | pg_upgrade now updates the existing pg_database.daticurules column for template0; no catalog schema change. |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The fix is requested for 16 and later, and v1 bundles a StringInfo rewrite of shared helpers in util.c and server.c. |
| maintenance burden ("who maintains this in 10 years") | **yes** | v1 changes the query and logging helpers that every pg_upgrade step uses, and removes the target-version branches. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Can the series be split into a minimal daticurules fix for 16 and later, plus the buffer and branch cleanup for master only?
2. Does the daticurules fix depend on the StringInfo buffer change, and if so, at what rule length does the existing query buffer truncate?
3. Is there a test that upgrades a cluster initialized with multiline or long --icu-rules and checks template0's daticurules afterwards?
4. Should the CREATE DATABASE fix from message #7 go into this CommitFest entry or into a separate one with its own backpatch range?
5. Does the open question about why template0 keeps locale fields at all block this fix, or can the fix go in first?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Heikki Linnakangas started the thread, is the registered reviewer and agreed the fix is right; the owner needs to know pg_upgrade and the ICU locale fields in pg_database.
- **Liability:** The owner signs for pg_upgrade behaviour on every stable branch from 16. If the StringInfo rewrite of the query and logging helpers is backpatched, that covers every pg_upgrade run, including clusters without ICU rules.
- **Author follow-up:** A v1 patch followed the review in message #1, and a second patch in message #7 addressed the CREATE DATABASE bug that Heikki found.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** Only clusters initialized with --icu-rules are affected, but for them an upgrade silently drops template0's rules. The patch is about 10 KiB in five pg_upgrade files, a committer who knows the area is reviewing, and the open point is how to split it for backpatch.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. [P3] was a 2016 documentation patch about matching initdb flags, and its reason, getting the cluster locale from pg_controldata, does not apply to copying daticurules. Message #1 says the issue was split out of the pg_upgrade --initdb thread as an independent fix, and no earlier rejection of this fix is reported.
- standards-compat: 0 objection(s). No user-facing contract changes beyond making pg_upgrade match what initdb already set. After upgrade, template0 again matches the old cluster, so `create database ... template template0` behaves as before. There is no new syntax, GUC, catalog column or psql change, and pg_dump never dumps template0, so the dump round-trip is unaffected. The separate CREATE DATABASE fix in message #7 only removes an error that blocks valid commands, so existing scripts do not change behaviour. A committer may want the StringInfo refactor kept out of the back-branch commit, but that is a question of patch shape, not of compatibility. This angle cannot check whether the source-version checks still handle sources older than 16.
- replication-ondisk: 0 objection(s). Nothing in this proposal changes on-disk format, WAL, the replication protocol, the wire protocol, the catalog schema or any extension ABI. The fix writes an existing column, daticurules, into the new cluster's template0 during pg_upgrade, the same way 16 already writes the encoding and locale columns. The buffer change stays inside the frontend pg_upgrade binary. Source-version checks stay in place for clusters older than 16. One point could go in the release notes: clusters already upgraded under the bug have empty rules on template0, and this fix will not restore them.
- security: 0 objection(s). No privilege or exposure aspect found. pg_upgrade runs as the cluster superuser, the rules come from the old cluster's template0, which only initdb sets, and PQescapeLiteral() quotes the value. Neither the larger buffers nor the CREATE DATABASE fix gives a non-superuser any new action or information.
- maintenance-fit: 1 objection(s). The idea itself fits core. It fixes an omission in existing pg_upgrade code, adds nothing to hot paths, and a committer who knows the area (Heikki) is engaged as reviewer. The only concern from this angle is how the patch is split for backpatching.
- already-exists: 0 objection(s). Nothing in Postgres or a known extension already copies template0's daticurules during pg_upgrade. Running initdb with the same --icu-rules works around it today, but since 16 pg_upgrade copies template0's locale fields so that matching initdb options are not needed, and this patch completes that same mechanism instead of adding a second one. [P3] covers documentation for matching initdb flags and does not apply.
- demand: 0 objection(s). This is a bug fix for an existing feature, so the demand test does not apply. pg_upgrade already copies template0's encoding and locale fields, and only daticurules is skipped; a user report started the thread, and Heikki in messages #1 and #6 agrees it should be fixed. The affected group, clusters initialized with --icu-rules, is small, but the cost stays inside pg_upgrade, with no hot path, UI or catalog change.

### Precedents retrieved

- P1: [pg_upgrade: Support for upgrading to checksums enabled](https://commitfest.postgresql.org/patch/5206/) (cf, Returned with feedback, 2025-04-03) - judged not relevant; reason summarised from its thread
- P2: [add timing information to pg_upgrade](https://commitfest.postgresql.org/patch/4464/) (cf, Withdrawn, 2023-12-02) - judged not relevant; reason summarised from its thread
- P3: [pg_upgrade documentation improvement](https://commitfest.postgresql.org/patch/583/) (cf, Returned with feedback, 2016-10-03) - used in the review; reason summarised from its thread
- P4: [pg_upgrade version and path checking](https://commitfest.postgresql.org/patch/2095/) (cf, Returned with feedback, 2019-09-30) - judged not relevant; reason summarised from its thread
- P5: [Backport perl tests for pg_upgrade](https://commitfest.postgresql.org/patch/3997/) (cf, Withdrawn, 2022-12-19) - judged not relevant; reason summarised from its thread
- P6: [pg_upgrade allows itself to be run twice (initdb)](https://commitfest.postgresql.org/patch/3872/) (cf, Returned with feedback, 2022-12-01) - judged not relevant; reason summarised from its thread
- P7: [pg_upgrade: Pass -j option down to vacuumdb](https://commitfest.postgresql.org/patch/1943/) (cf, Returned with feedback, 2019-04-09) - judged not relevant; reason summarised from its thread
- P8: [Fix order of checking ICU options in initdb and create database](https://commitfest.postgresql.org/patch/3976/) (cf, Returned with feedback, 2023-07-06) - judged not relevant; reason summarised from its thread
- P9: [Rewriting the test of pg_upgrade as a TAP test](https://commitfest.postgresql.org/patch/1101/) (cf, Returned with feedback, 2017-09-29) - judged not relevant
- P10: [ICU as default collation provider](https://commitfest.postgresql.org/patch/1543/) (cf, Returned with feedback, 2019-02-04) - judged not relevant
- P11: [pg_upgrade: warn about roles with md5 passwords](https://commitfest.postgresql.org/patch/5792/) (cf, Withdrawn, 2025-06-04) - judged not relevant
- P12: [initdb: change default --locale-provider back to libc.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2535c74b1a6190cc42e13f6b6b55d94bff4b7dd6) (revert, Partially reverted, 2023-06-21) - judged not relevant
- P13: [Fix database creation during installchecks for ICU cluster](https://commitfest.postgresql.org/patch/3975/) (cf, Returned with feedback, 2023-01-31) - judged not relevant
- P14: [Default to ICU during initdb](https://commitfest.postgresql.org/patch/4169/) (cf, Returned with feedback, 2023-07-06) - judged not relevant
- P15: [Set default transactions to read-only at servers start in pg_upgrade](https://commitfest.postgresql.org/patch/2970/) (cf, Withdrawn, 2021-12-01) - judged not relevant
- P16: [Revert "Add basic TAP test setup for pg_upgrade"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=58ffe141eb37c3f027acd25c1fc6b36513bf9380) (revert, Reverted, 2017-09-22) - judged not relevant
- P17: [Rewriting the test of pg_upgrade as a TAP test - take three](https://commitfest.postgresql.org/patch/3120/) (cf, Withdrawn, 2021-11-18) - judged not relevant
- P18: [CREATE DATABASE ... STRICT_UNICODE](https://commitfest.postgresql.org/patch/4876/) (cf, Withdrawn, 2024-12-11) - judged not relevant
- P19: [Revert "ICU: do not convert locale 'C' to 'en-US-u-va-posix'."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=455f948b0d03a556533a7e4a1a8abf45f0eb202e) (revert, Reverted, 2023-05-09) - judged not relevant
- P20: [pg_dump -C Grant statement on create database](https://commitfest.postgresql.org/patch/724/) (cf, Returned with feedback, 2016-10-03) - judged not relevant
- P21: [Revert "Fix replay of create database records on standby"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=bf902c13930c268388644100663f2998868b6e85) (revert, Reverted, 2022-03-29) - judged not relevant
- P22: [implement CAST(expr AS type FORMAT 'template')](https://commitfest.postgresql.org/patch/5957/) (cf, Withdrawn, 2026-09-09) - judged not relevant
- P23: [Allow meson to find ICU in non-standard location](https://commitfest.postgresql.org/patch/4203/) (cf, Withdrawn, 2023-03-08) - judged not relevant
- P24: [Speed up ICU case conversions](https://commitfest.postgresql.org/patch/5474/) (cf, Returned with feedback, 2025-03-30) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 11; wall time: 68 s; cost reported by the CLI: $0.79
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 8 from cache, 0 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 51 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T04:11:22.491Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

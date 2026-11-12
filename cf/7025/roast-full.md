# Design pre-review: pg_upgrade: add --initdb option to create the new cluster automatically

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7025](https://commitfest.postgresql.org/patch/7025/) (Needs review); authors: Bohyun Lee (leebohyun)
- Thread: [\[PATCH\] pg_upgrade: add --initdb option to create the new cluster automatically](https://www.postgresql.org/message-id/flat/CAMPh8Mr8FbMnPct2Zom88UkpkddT48mZoY%2Bm4rRxG9PdW6Fo%2BA%40mail.gmail.com), started 2026-07-10
- Messages read: 28 of 28
- Patch prose read: v10-pg_upgrade-initdb.patch, v1-pg_upgrade-initdb.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

No level fails outright: reviewers took the goal seriously and worked on the design through v10. Two things still need agreement on the list before code review is useful. Daniel's in-tree versus wrapper doubt has no answer, and the thread does not show which failures remain once the docs-only 0001 is applied (maintenance-fit-2, already-exists-1). Heikki has not replied on the v9 cleanup design, which leaves --check able to keep a cluster on some failures (standards-compat-2) and adds a state-file contract between initdb and pg_upgrade (maintenance-fit-1).

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, two points could not be checked: whether --initdb-options reaches initdb without a shell, and whether the state-file-driven removal accepts only paths that pg_upgrade recorded. Only 3 of 28 messages were read directly.

## The proposal, restated

**Problem.** Before pg_upgrade, users must run initdb by hand with settings that match the old cluster, mainly WAL segment size and data checksums. A mismatch shows up late, as a check_control_data() failure. The thread targets operators who run pg_upgrade directly, without a wrapper such as Debian's pg_upgradecluster.

**Proposed design.** With --initdb, pg_upgrade reads WAL segment size and checksum setting from the old pg_control, briefly starts the old server to read template0 encoding, locale and ICU rules, and runs initdb in the target directory. Logs go to <new datadir>/pg_upgrade_output.d from the start, which a hidden initdb option allows. With --check, the new cluster is created, checked, then removed.

**User-visible surface.**
- New pg_upgrade option --initdb.
- New pg_upgrade option --initdb-options, repeatable, with its own quoting rules, passed to initdb after the generated arguments.
- Changed --check behaviour with --initdb: it creates a temporary cluster and removes it, along with separate WAL files.
- New hidden initdb option --for-pg-upgrade=<state-file> that accepts a target holding only pg_upgrade_output.d.
- Separate docs-only patch saying which initdb settings must match the old cluster when initdb is run by hand.

**What the author claims.**
- Users get WAL segment size, checksums and locale wrong by hand and only find out at check_control_data().
- pg_upgrade already reads the old control data, so doing this in-tree avoids having a wrapper rediscover it.
- A parity script shows manual initdb plus pg_upgrade gives the same state as --initdb with the same options.
- Failed checks clean up the new cluster, so the same command can be retried.
- Reviewer Greg Sabino Mullane called it a much needed feature he would use a lot.

**Thread state.** Reviewers Greg Sabino Mullane, Daniel Gustafsson, Hüseyin Demir, Heikki Linnakangas and Shihao Zhong drove the design through v10; Shihao gave an LGTM on v10. Heikki has not replied to the v9 design, and Daniel's doubt about in-tree versus wrapper was never settled; the CF entry needs a rebase.

## 2. Prior attempts

### pg_upgrade documentation improvement (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/583/](https://commitfest.postgresql.org/patch/583/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CACuBw0hEbZXUW+-HbU3+KXjN7=t=g7Xp56-MhObtP6VfH3K0qQ@mail.gmail.com](https://www.postgresql.org/message-id/flat/CACuBw0hEbZXUW+-HbU3+KXjN7=t=g7Xp56-MhObtP6VfH3K0qQ@mail.gmail.com)
- **Date:** decided 2016-10-03; started 2016-03-20
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Peter Eisentraut (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Peter Eisentraut said the cluster locale is better taken from pg_controldata and doubted that pg_upgrade needs the encoding to match. The author asked whether to change the patch to use pg_controldata only, but no revised patch followed. Michael Paquier marked the entry returned with feedback because the thread had died.
- **Reason (quote):** "A better way to get the *cluster* locale is from pg_controldata." ([message](https://www.postgresql.org/message-id/570E3BA5.8060809%40gmx.net))
- **What is different this time:** P4 was a docs-only patch that told users to match encoding and locale from template0. Here the docs-only 0001 names only WAL segment size and checksums and points to pg_controldata, and the main patch automates the step.


## 3. Objections a reviewer is likely to raise

### 1. In-tree versus wrapper question is still open, and the scope has grown

fixable · level 1b, rubric item 4 · doesn't fit what Postgres is about (extension / external tool / already solved), maintenance burden ("who maintains this in 10 years") · `maintenance-fit-2`

Core takes on initdb orchestration, quoting of --initdb-options, temporary cluster creation and removal, and WAL directory emptying. Packagers who already ship pg_upgradecluster-style wrappers gain nothing. Every pg_upgrade user carries the extra failure modes in a tool whose failures happen during upgrade windows.

Daniel doubted that this complexity belongs in core (message #5) and asked for docs first (message #7). The author answered that pg_upgrade already reads pg_control (messages #6, #9), but Daniel did not reply on that point. Since then the patch grew from about 18 KiB to 73 KiB and added deletion and permission-restore logic. The docs-only 0001 fixes the stated problem, a late check_control_data() failure, with no new code.

**Precedents.**
- [pg_upgrade documentation improvement](https://commitfest.postgresql.org/patch/583/) (Returned with feedback, 2016-10-03); thread: [link](https://www.postgresql.org/message-id/flat/CACuBw0hEbZXUW+-HbU3+KXjN7=t=g7Xp56-MhObtP6VfH3K0qQ@mail.gmail.com)

Thread: [#5 (Daniel Gustafsson, 2026-07-14)](https://www.postgresql.org/message-id/24958906-2301-478B-925F-7977E3785069%40yesql.se), [#6 (Bohyun Lee, 2026-07-14)](https://www.postgresql.org/message-id/CAMPh8MqkGwhu6UC40rWyZ9tb18LgV4VthZFUcbh%2BAGaxy8dyUw%40mail.gmail.com), [#7 (Daniel Gustafsson, 2026-07-16)](https://www.postgresql.org/message-id/5C80002C-27DA-4861-B20D-23DA28817CF5%40yesql.se), [#9 (Bohyun Lee, 2026-07-17)](https://www.postgresql.org/message-id/CAMPh8MoVY5cUM%2BBLqBQkbL8ttHCMiVZ6Z9f3AL%2BK%2BTXSh%3DUP%2BA%40mail.gmail.com), [#11 (Daniel Gustafsson, 2026-08-12)](https://www.postgresql.org/message-id/23C10B41-AA5E-4F3F-9727-CA94F9B84817%40yesql.se), [#15 (Bohyun Lee, 2026-08-18)](https://www.postgresql.org/message-id/CAMPh8MrH9wvP7p_7RuyjdVDn6KtLGevZbz4_NfYY%2BGffwBwecw%40mail.gmail.com)

**Question:** After 0001 documents the required settings, which users still get them wrong, and why can a wrapper not read pg_controldata as well?

Counter-check: held. Daniel's in-tree versus wrapper doubt got only "Maybe" in message #7, and the patch has since grown. P4 must not be used as support, and the runtime cost falls on --initdb users and maintainers, not on every pg_upgrade user.

### 2. Manual initdb plus pg_upgrade --check already meets the need

fixable · level 1b, rubric item 3 · desirability, doesn't fit what Postgres is about (extension / external tool / already solved), maintenance burden ("who maintains this in 10 years") · `already-exists-1`

An operator reads the WAL segment size and checksum state with pg_controldata and runs initdb by hand. If a value is wrong, pg_upgrade --check reports it within seconds, before any data is copied, and the operator reruns initdb. The project still carries a hidden initdb option, a state file and a second cleanup path to save that one retry.

Message #15 says only WAL segment size and checksums must match, since pg_upgrade copies encoding and locale. The docs-only 0001 states this and points to pg_controldata, and Daniel (message #7) and Shihao (message #17) want it applied on its own. The author's answer (messages #6, #9) is that pg_upgrade already reads pg_control. That shows the step is less convenient today, but it can still be done. The material has one reviewer's wish to use it (message #2) and no field reports measured against the improved docs.

Thread: [#2 (Greg Sabino Mullane, 2026-07-10)](https://www.postgresql.org/message-id/CAKAnmmL6atdmODMBC6dHPT3PK1Lf4jS5xMHrYL_eg0q_FNBDpQ%40mail.gmail.com), [#6 (Bohyun Lee, 2026-07-14)](https://www.postgresql.org/message-id/CAMPh8MqkGwhu6UC40rWyZ9tb18LgV4VthZFUcbh%2BAGaxy8dyUw%40mail.gmail.com), [#7 (Daniel Gustafsson, 2026-07-16)](https://www.postgresql.org/message-id/5C80002C-27DA-4861-B20D-23DA28817CF5%40yesql.se), [#9 (Bohyun Lee, 2026-07-17)](https://www.postgresql.org/message-id/CAMPh8MoVY5cUM%2BBLqBQkbL8ttHCMiVZ6Z9f3AL%2BK%2BTXSh%3DUP%2BA%40mail.gmail.com), [#15 (Bohyun Lee, 2026-08-18)](https://www.postgresql.org/message-id/CAMPh8MrH9wvP7p_7RuyjdVDn6KtLGevZbz4_NfYY%2BGffwBwecw%40mail.gmail.com), [#17 (shihao zhong, 2026-09-08)](https://www.postgresql.org/message-id/CAGRkXqT4xX%2B-zfBWjQiS%2B7pY0THyswoAw08CDS3NFvotrCsa8g%40mail.gmail.com)

**Question:** Once 0001 is applied, which concrete failures remain that pg_controldata plus pg_upgrade --check cannot catch before any data is copied?

Counter-check: held. Manual initdb with pg_controldata and --check can do the job, and message #15 narrows the required match to two settings. The thread shows one reviewer's wish and an older question, which does not show the need is common.

### 3. --check with --initdb writes to disk and empties a WAL directory

fixable · level 2, rubric item 10 · design, psql UX conventions · `standards-compat-2`

A deployment script adds --initdb and then runs pg_upgrade --check in a pre-flight step, assuming it is read-only, as the current docs say. The check runs initdb, writes WAL into the directory named by --waldir, and then removes it. If shutdown or the state file fails, the docs say the cluster is retained, so the later real run fails on a non-empty target.

The v10 patch rewrites the --check description from "don't change any data" to "check clusters without performing the upgrade". With --initdb, the docs list several partial-cleanup outcomes that need manual recovery. Heikki asked that --check leave the target unmodified (message #24). The design meets that only on the success path, so --check now has failure modes that change files on disk.

Thread: [#20 (Bohyun Lee, 2026-10-07)](https://www.postgresql.org/message-id/CAMPh8MojTuNXGhL4tKyiKe2B4yNri8M61CAChEeHu%3DKciikHGA%40mail.gmail.com), [#21 (Heikki Linnakangas, 2026-10-12)](https://www.postgresql.org/message-id/e011d91c-99c4-490d-80ae-dcdba16f5955%40iki.fi), [#24 (Heikki Linnakangas, 2026-10-14)](https://www.postgresql.org/message-id/434308b3-13a6-4da6-a554-51f0756c78d5%40iki.fi), [#25 (Bohyun Lee, 2026-10-21)](https://www.postgresql.org/message-id/CAMPh8MrPMBnSYGmmqdp_pNesmupV5LS2pkBT-Zup34Ehs5d9kQ%40mail.gmail.com)

**Question:** Can --check --initdb keep its guarantee in every failure case, for example by initializing into a throwaway location, so that --check never leaves files that block the real run?

Counter-check: held. The docs say cleanup also runs when checks fail, so "only on the success path" overstates it. But the cluster is retained when the state file cannot be read or shutdown is not confirmed, and Heikki has not replied to this design.

### 4. Hidden initdb state-file protocol ties initdb internals to pg_upgrade cleanup

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years"), ABI / extension API · `maintenance-fit-1`

A later release adds an initdb option that creates storage outside the data directory, in the way --waldir does. Nobody updates the --for-pg-upgrade state file, so pg_upgrade --check --initdb leaves that storage behind. A retry then fails, or later cleanup removes the wrong thing, and operators find out in the field.

In v9 (message #25), initdb records the external WAL path, whether the directory existed before, its permissions and the nearest existing parent. pg_upgrade reads that record to empty WAL directories and restore permissions. This makes a private contract between two binaries, and v10 also touches src/common/rmtree.c and src/port/pgcheckdir.c. Heikki suggested the hidden option only to stop initdb rejecting pg_upgrade_output.d (message #24), and he has not replied on the cleanup design (message #25).

Thread: [#21 (Heikki Linnakangas, 2026-10-12)](https://www.postgresql.org/message-id/e011d91c-99c4-490d-80ae-dcdba16f5955%40iki.fi), [#24 (Heikki Linnakangas, 2026-10-14)](https://www.postgresql.org/message-id/434308b3-13a6-4da6-a554-51f0756c78d5%40iki.fi), [#25 (Bohyun Lee, 2026-10-21)](https://www.postgresql.org/message-id/CAMPh8MrPMBnSYGmmqdp_pNesmupV5LS2pkBT-Zup34Ehs5d9kQ%40mail.gmail.com), [#27 (Bohyun Lee, 2026-10-27)](https://www.postgresql.org/message-id/CAMPh8MrYxbKqaZzFHE9U7oVa6qrDwjm8XAG08S2%3DKACj_OBUQQ%40mail.gmail.com)

**Question:** What makes a future initdb option that creates external storage fail a test when it is missing from the state file and its cleanup?

Counter-check: held. Message #25 confirms the state file carries the WAL path, prior existence, permissions and parent for pg_upgrade's cleanup. Heikki proposed only the hidden option, and nothing in the thread makes a future external-storage option fail a test.

### 5. --initdb-options adds a third argument-quoting grammar to pg_upgrade

fixable · level 2, rubric item 12 · design, maintenance burden ("who maintains this in 10 years") · `standards-compat-1`

An operator copies a working initdb command line into --initdb-options, with a quoted locale name or a Windows path such as --waldir=C:\pg wal. pg_upgrade splits it under its own rules for backslashes, quotes and newlines, which differ from both the shell and the -o/-O handling. The operator finds the mismatch only when initdb fails or builds the wrong cluster.

The v10 docs define a new tokenizer for --initdb-options: its own quote, backslash and newline rules. The existing -o/-O strings are passed through to pg_ctl under a different convention, and the thread already showed how fragile -O splitting is (message #8). Each new release would have to keep this grammar stable. A repeatable option that carries one initdb argument per use needs no grammar at all.

Thread: [#8 (Greg Sabino Mullane, 2026-07-16)](https://www.postgresql.org/message-id/CAKAnmmKb7o9iWCXEt0uY%3DpZwo50OZD%3DE%3DHm%2Bf18DZM%2Bh7%2BU-dw%40mail.gmail.com), [#20 (Bohyun Lee, 2026-10-07)](https://www.postgresql.org/message-id/CAMPh8MojTuNXGhL4tKyiKe2B4yNri8M61CAChEeHu%3DKciikHGA%40mail.gmail.com), [#23 (Bohyun Lee, 2026-10-14)](https://www.postgresql.org/message-id/CAMPh8MpLR8sax%3DfG6ShzkE_LF9JFL7MR8eyX3WrNguYp57Xpvw%40mail.gmail.com)

**Question:** Why not take one initdb argument per --initdb-options occurrence, so pg_upgrade never has to define or maintain its own quoting rules?

Counter-check: held. The v10 docs define pg_upgrade's own splitting and quoting rules for --initdb-options, and the option is already repeatable. The thread does not say why one initdb argument per use was not chosen.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | pg_upgrade now creates and removes the new cluster and its separate WAL directory; no on-disk format changes. |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | The patch changes src/include/port.h, src/port/pgcheckdir.c and src/common/rmtree.c, which other frontend and backend code uses. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | **yes** | No psql change, but pg_upgrade gains two options, a quoting grammar, and a --check that writes to disk. |
| security / privileges | **yes** | pg_upgrade removes directories and restores permissions from a state file, and passes user-supplied arguments to initdb. |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | A hidden initdb option, a state file read by pg_upgrade, cleanup paths and a custom tokenizer need care in every release. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | Message #5 asks whether this belongs in wrappers such as pg_upgradecluster, and the thread has not settled it. |

## 5. Questions the author must answer before anyone reviews code

1. Once the docs-only 0001 is applied, which concrete failures remain that pg_controldata plus pg_upgrade --check cannot catch before any data is copied?
2. What can --initdb do that a wrapper reading pg_controldata cannot, given that message #15 narrows the required match to WAL segment size and checksums?
3. Can --check --initdb initialize into a throwaway location, so that a failed state-file read or unconfirmed shutdown never leaves files that block the real run?
4. What makes a future initdb option that creates storage outside the data directory fail a test when the state file and its cleanup do not cover it?
5. Why does --initdb-options not take exactly one initdb argument per use, so pg_upgrade needs no quoting rules of its own?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Someone who knows both pg_upgrade and initdb would have to own it. Heikki Linnakangas shaped the hidden initdb option in message #24 but has not replied on the v9 and v10 cleanup design.
- **Liability:** The owner signs for directory removal and permission restore during upgrades, the state-file contract between initdb and pg_upgrade, and the --initdb-options grammar, for as long as pg_upgrade supports these options.
- **Author follow-up:** The author produced ten versions in about four months in response to review. The CommitFest entry has needed a rebase since 2026-10-14.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** unclear
- **Maintenance burden:** medium
- **Review cost:** high
- **Why:** The thread names the target users, operators who run pg_upgrade without a wrapper, but shows one reviewer's wish as demand. The v10 patch is 73 KiB across pg_upgrade, initdb, src/common and src/port, and its cleanup design and in-tree placement still lack agreement.

## Appendix

### Refuted in the counter-check

- `standards-compat-3` Template0 ICU rules may differ between manual initdb and --initdb. Refuted (no failure scenario): The scenario rests on an assumption that the copy runs only with --initdb, which the material does not support. Message #27 says "the upgrade now copies template0.daticurules", and how a manual initdb handles ICU rules is existing behaviour.
- `maintenance-fit-3` --initdb-options adds a second quoting grammar inside pg_upgrade. Refuted (not specific): Duplicate of standards-compat-1.
- `already-exists-2` Packager wrappers already create the new cluster and own this policy. Refuted (not specific): Duplicate of maintenance-fit-2. Its own scenario, that a wrapper and --initdb choose different settings, is not a failure of this proposal, and Heikki removed group-access inheritance to match manual initdb.
- `demand-1` Remaining problem after the docs fix may not justify the added surface. Refuted (not specific): Duplicate of already-exists-1 for the docs-plus-manual point, and of maintenance-fit-2 for the wrapper point.

### Review angles

- prior-attempts: 0 objection(s). No listed precedent records a reason that still applies here: P4's reason is answered by 0001, P1 concerns rewriting checksums during copy, and P9 has no recorded reason. Message #1 cites an earlier discussion with Jeff Davis about pg_upgrade running initdb itself, but the thread does not report how it ended.
- standards-compat: 3 objection(s). The hidden initdb --for-pg-upgrade option follows existing practice for hidden binary-upgrade switches, so it was not raised. The docs-only 0001 has no compatibility impact.
- replication-ondisk: 0 objection(s). The proposal changes no on-disk format, WAL record, catalog definition, wire protocol message or extension API; it runs the existing initdb before the existing pg_upgrade flow. The new persistent state (the initdb state file, the target and separate WAL directory cleanup) is paired with removal, and initdb's empty-directory rule limits what that removal can touch. The daticurules copy writes to the new template0 as existing locale handling does, which could not be confirmed without the code.
- security: 0 objection(s). pg_upgrade and initdb run as the OS user that owns the clusters, and --initdb crosses no privilege boundary. The shell-injection risk in message #11 was fixed in message #15 with appendShellString(). Message #21 dropped group-access inheritance in favour of initdb's stricter default. No SQL-level privilege, RLS, or exposure surface is added. Two points depend on the code and could not be checked: whether --initdb-options reaches initdb without a shell, and whether the state-file-driven rmtree() accepts only paths that pg_upgrade itself recorded.
- maintenance-fit: 3 objection(s). Not applicable; three objections listed.
- already-exists: 2 objection(s). Not applicable; two objections are raised.
- demand: 1 objection(s). The problem is real and several reviewers engaged, but the demand evidence is one enthusiastic reviewer. The objection is 'show demand beyond the docs fix', not a rejection.

### Precedents retrieved

- P1: [pg_upgrade: Support for upgrading to checksums enabled](https://commitfest.postgresql.org/patch/5206/) (cf, Returned with feedback, 2025-04-03) - used in the review; reason summarised from its thread
- P2: [add timing information to pg_upgrade](https://commitfest.postgresql.org/patch/4464/) (cf, Withdrawn, 2023-12-02) - judged not relevant; reason summarised from its thread
- P3: [pg_upgrade version and path checking](https://commitfest.postgresql.org/patch/2095/) (cf, Returned with feedback, 2019-09-30) - judged not relevant; reason summarised from its thread
- P4: [pg_upgrade documentation improvement](https://commitfest.postgresql.org/patch/583/) (cf, Returned with feedback, 2016-10-03) - used in the review; reason summarised from its thread
- P5: [Preserve versions of initdb-created collations in pg_upgrade](https://commitfest.postgresql.org/patch/2328/) (cf, Returned with feedback, 2020-01-24) - judged not relevant; reason summarised from its thread
- P6: [Backport perl tests for pg_upgrade](https://commitfest.postgresql.org/patch/3997/) (cf, Withdrawn, 2022-12-19) - judged not relevant; reason summarised from its thread
- P7: [pg_upgrade: Pass -j option down to vacuumdb](https://commitfest.postgresql.org/patch/1943/) (cf, Returned with feedback, 2019-04-09) - judged not relevant; reason summarised from its thread
- P8: [Rewriting the test of pg_upgrade as a TAP test](https://commitfest.postgresql.org/patch/1101/) (cf, Returned with feedback, 2017-09-29) - judged not relevant; reason summarised from its thread
- P9: [pg_upgrade allows itself to be run twice (initdb)](https://commitfest.postgresql.org/patch/3872/) (cf, Returned with feedback, 2022-12-01) - used in the review
- P10: [pg_upgrade: remove C comment](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=84a6f63e32dbefe3dc76cbe628fab6cbfc26141e) (revert, Partially reverted, 2018-01-05) - judged not relevant
- P11: [Revert "Add basic TAP test setup for pg_upgrade"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=58ffe141eb37c3f027acd25c1fc6b36513bf9380) (revert, Reverted, 2017-09-22) - judged not relevant
- P12: [Add initdb option to initialize cluster with non-standard xid/mxid/mxoff.](https://commitfest.postgresql.org/patch/3634/) (cf, Rejected, 2023-03-21) - judged not relevant
- P13: [Revert "Enable fast default for domains with non-volatile constraints"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a0354e29c41a9fb7491b3c7c23f079b1923c045a) (revert, Reverted, 2026-06-08) - judged not relevant
- P14: [Revert Non text modes for pg_dumpall, and pg_restore support](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ce9a6244b5b4ce1df71611512a757353803404a5) (revert, Reverted, 2025-07-30) - judged not relevant
- P15: [pg_upgrade data type check connection overhead reduction](https://commitfest.postgresql.org/patch/4200/) (cf, Returned with feedback, 2024-02-01) - judged not relevant
- P16: [pg_upgrade check for Unicode update](https://commitfest.postgresql.org/patch/5563/) (cf, Withdrawn, 2025-03-18) - judged not relevant
- P17: [Add pg_upgrade check for invalid role references in pg_init_privs](https://commitfest.postgresql.org/patch/6856/) (cf, Withdrawn, 2026-06-17) - judged not relevant
- P18: [empty](https://commitfest.postgresql.org/patch/5037/) (cf, Rejected, 2024-06-21) - judged not relevant
- P19: [Set host(no)ssl settings from initdb](https://commitfest.postgresql.org/patch/2378/) (cf, Returned with feedback, 2020-09-07) - judged not relevant
- P20: [Let people set host(no)ssl settings from initdb](https://commitfest.postgresql.org/patch/2916/) (cf, Rejected, 2021-03-09) - judged not relevant
- P21: [improve large object handling in pg_upgrade](https://commitfest.postgresql.org/patch/3841/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P22: [Revert "initdb: Change authentication defaults"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=7961886580a594e519ca7ed1811b464206738be5) (revert, Reverted, 2019-07-22) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 11; wall time: 151 s; cost reported by the CLI: $3.33
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 8 from cache, 0 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 3 of 22 kept
- Tone check: 95 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:15:56.202Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

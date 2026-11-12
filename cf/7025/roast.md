# Design pre-review: pg_upgrade: add --initdb option to create the new cluster automatically

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7025](https://commitfest.postgresql.org/patch/7025/); [thread](https://www.postgresql.org/message-id/flat/CAMPh8Mr8FbMnPct2Zom88UkpkddT48mZoY%2Bm4rRxG9PdW6Fo%2BA%40mail.gmail.com), 28 of 28 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). No level fails outright: reviewers took the goal seriously and worked on the design through v10. Two things still need agreement on the list before code review is useful. Daniel's in-tree versus wrapper doubt has no answer, and the thread does not show which failures remain once the docs-only 0001 is applied (maintenance-fit-2, already-exists-1). Heikki has not replied on the v9 cleanup design, which leaves --check able to keep a cluster on some failures (standards-compat-2) and adds a state-file contract between initdb and pg_upgrade (maintenance-fit-1).

Not checked: Without the code, two points could not be checked: whether --initdb-options reaches initdb without a shell, and whether the state-file-driven removal accepts only paths that pg_upgrade recorded. Only 3 of 28 messages were read directly.

## Prior attempts

- [pg_upgrade documentation improvement](https://www.postgresql.org/message-id/flat/CACuBw0hEbZXUW+-HbU3+KXjN7=t=g7Xp56-MhObtP6VfH3K0qQ@mail.gmail.com) (similar idea; Returned with feedback, 2016-10-03). "A better way to get the *cluster* locale is from pg_controldata." ([message](https://www.postgresql.org/message-id/570E3BA5.8060809%40gmx.net)) Different now: P4 was a docs-only patch that told users to match encoding and locale from template0. Here the docs-only 0001 names only WAL segment size and checksums and points to pg_controldata, and the main patch automates the step.

## Objections a reviewer is likely to raise

### 1. In-tree versus wrapper question is still open, and the scope has grown

Core takes on initdb orchestration, quoting of --initdb-options, temporary cluster creation and removal, and WAL directory emptying. Packagers who already ship pg_upgradecluster-style wrappers gain nothing. Every pg_upgrade user carries the extra failure modes in a tool whose failures happen during upgrade windows.

Daniel doubted that this complexity belongs in core (message #5) and asked for docs first (message #7). The author answered that pg_upgrade already reads pg_control (messages #6, #9), but Daniel did not reply on that point. Since then the patch grew from about 18 KiB to 73 KiB and added deletion and permission-restore logic. The docs-only 0001 fixes the stated problem, a late check_control_data() failure, with no new code.

Precedent: [pg_upgrade documentation improvement](https://www.postgresql.org/message-id/flat/CACuBw0hEbZXUW+-HbU3+KXjN7=t=g7Xp56-MhObtP6VfH3K0qQ@mail.gmail.com) (Returned with feedback, 2016-10-03)

**Question:** After 0001 documents the required settings, which users still get them wrong, and why can a wrapper not read pg_controldata as well?

<sub>fixable, level 1b, doesn't fit what Postgres is about (extension / external tool / already solved); messages [#5](https://www.postgresql.org/message-id/24958906-2301-478B-925F-7977E3785069%40yesql.se), [#6](https://www.postgresql.org/message-id/CAMPh8MqkGwhu6UC40rWyZ9tb18LgV4VthZFUcbh%2BAGaxy8dyUw%40mail.gmail.com), [#7](https://www.postgresql.org/message-id/5C80002C-27DA-4861-B20D-23DA28817CF5%40yesql.se), [#9](https://www.postgresql.org/message-id/CAMPh8MoVY5cUM%2BBLqBQkbL8ttHCMiVZ6Z9f3AL%2BK%2BTXSh%3DUP%2BA%40mail.gmail.com), [#11](https://www.postgresql.org/message-id/23C10B41-AA5E-4F3F-9727-CA94F9B84817%40yesql.se), [#15](https://www.postgresql.org/message-id/CAMPh8MrH9wvP7p_7RuyjdVDn6KtLGevZbz4_NfYY%2BGffwBwecw%40mail.gmail.com); `maintenance-fit-2`</sub>

### 2. Manual initdb plus pg_upgrade --check already meets the need

An operator reads the WAL segment size and checksum state with pg_controldata and runs initdb by hand. If a value is wrong, pg_upgrade --check reports it within seconds, before any data is copied, and the operator reruns initdb. The project still carries a hidden initdb option, a state file and a second cleanup path to save that one retry.

Message #15 says only WAL segment size and checksums must match, since pg_upgrade copies encoding and locale. The docs-only 0001 states this and points to pg_controldata, and Daniel (message #7) and Shihao (message #17) want it applied on its own. The author's answer (messages #6, #9) is that pg_upgrade already reads pg_control. That shows the step is less convenient today, but it can still be done. The material has one reviewer's wish to use it (message #2) and no field reports measured against the improved docs.

**Question:** Once 0001 is applied, which concrete failures remain that pg_controldata plus pg_upgrade --check cannot catch before any data is copied?

<sub>fixable, level 1b, desirability; messages [#2](https://www.postgresql.org/message-id/CAKAnmmL6atdmODMBC6dHPT3PK1Lf4jS5xMHrYL_eg0q_FNBDpQ%40mail.gmail.com), [#6](https://www.postgresql.org/message-id/CAMPh8MqkGwhu6UC40rWyZ9tb18LgV4VthZFUcbh%2BAGaxy8dyUw%40mail.gmail.com), [#7](https://www.postgresql.org/message-id/5C80002C-27DA-4861-B20D-23DA28817CF5%40yesql.se), [#9](https://www.postgresql.org/message-id/CAMPh8MoVY5cUM%2BBLqBQkbL8ttHCMiVZ6Z9f3AL%2BK%2BTXSh%3DUP%2BA%40mail.gmail.com), [#15](https://www.postgresql.org/message-id/CAMPh8MrH9wvP7p_7RuyjdVDn6KtLGevZbz4_NfYY%2BGffwBwecw%40mail.gmail.com), [#17](https://www.postgresql.org/message-id/CAGRkXqT4xX%2B-zfBWjQiS%2B7pY0THyswoAw08CDS3NFvotrCsa8g%40mail.gmail.com); `already-exists-1`</sub>

### 3. --check with --initdb writes to disk and empties a WAL directory

A deployment script adds --initdb and then runs pg_upgrade --check in a pre-flight step, assuming it is read-only, as the current docs say. The check runs initdb, writes WAL into the directory named by --waldir, and then removes it. If shutdown or the state file fails, the docs say the cluster is retained, so the later real run fails on a non-empty target.

The v10 patch rewrites the --check description from "don't change any data" to "check clusters without performing the upgrade". With --initdb, the docs list several partial-cleanup outcomes that need manual recovery. Heikki asked that --check leave the target unmodified (message #24). The design meets that only on the success path, so --check now has failure modes that change files on disk.

**Question:** Can --check --initdb keep its guarantee in every failure case, for example by initializing into a throwaway location, so that --check never leaves files that block the real run?

<sub>fixable, level 2, design; messages [#20](https://www.postgresql.org/message-id/CAMPh8MojTuNXGhL4tKyiKe2B4yNri8M61CAChEeHu%3DKciikHGA%40mail.gmail.com), [#21](https://www.postgresql.org/message-id/e011d91c-99c4-490d-80ae-dcdba16f5955%40iki.fi), [#24](https://www.postgresql.org/message-id/434308b3-13a6-4da6-a554-51f0756c78d5%40iki.fi), [#25](https://www.postgresql.org/message-id/CAMPh8MrPMBnSYGmmqdp_pNesmupV5LS2pkBT-Zup34Ehs5d9kQ%40mail.gmail.com); `standards-compat-2`</sub>

### 4. Hidden initdb state-file protocol ties initdb internals to pg_upgrade cleanup

A later release adds an initdb option that creates storage outside the data directory, in the way --waldir does. Nobody updates the --for-pg-upgrade state file, so pg_upgrade --check --initdb leaves that storage behind. A retry then fails, or later cleanup removes the wrong thing, and operators find out in the field.

In v9 (message #25), initdb records the external WAL path, whether the directory existed before, its permissions and the nearest existing parent. pg_upgrade reads that record to empty WAL directories and restore permissions. This makes a private contract between two binaries, and v10 also touches src/common/rmtree.c and src/port/pgcheckdir.c. Heikki suggested the hidden option only to stop initdb rejecting pg_upgrade_output.d (message #24), and he has not replied on the cleanup design (message #25).

**Question:** What makes a future initdb option that creates external storage fail a test when it is missing from the state file and its cleanup?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#21](https://www.postgresql.org/message-id/e011d91c-99c4-490d-80ae-dcdba16f5955%40iki.fi), [#24](https://www.postgresql.org/message-id/434308b3-13a6-4da6-a554-51f0756c78d5%40iki.fi), [#25](https://www.postgresql.org/message-id/CAMPh8MrPMBnSYGmmqdp_pNesmupV5LS2pkBT-Zup34Ehs5d9kQ%40mail.gmail.com), [#27](https://www.postgresql.org/message-id/CAMPh8MrYxbKqaZzFHE9U7oVa6qrDwjm8XAG08S2%3DKACj_OBUQQ%40mail.gmail.com); `maintenance-fit-1`</sub>

### 5. --initdb-options adds a third argument-quoting grammar to pg_upgrade

An operator copies a working initdb command line into --initdb-options, with a quoted locale name or a Windows path such as --waldir=C:\pg wal. pg_upgrade splits it under its own rules for backslashes, quotes and newlines, which differ from both the shell and the -o/-O handling. The operator finds the mismatch only when initdb fails or builds the wrong cluster.

The v10 docs define a new tokenizer for --initdb-options: its own quote, backslash and newline rules. The existing -o/-O strings are passed through to pg_ctl under a different convention, and the thread already showed how fragile -O splitting is (message #8). Each new release would have to keep this grammar stable. A repeatable option that carries one initdb argument per use needs no grammar at all.

**Question:** Why not take one initdb argument per --initdb-options occurrence, so pg_upgrade never has to define or maintain its own quoting rules?

<sub>fixable, level 2, design; messages [#8](https://www.postgresql.org/message-id/CAKAnmmKb7o9iWCXEt0uY%3DpZwo50OZD%3DE%3DHm%2Bf18DZM%2Bh7%2BU-dw%40mail.gmail.com), [#20](https://www.postgresql.org/message-id/CAMPh8MojTuNXGhL4tKyiKe2B4yNri8M61CAChEeHu%3DKciikHGA%40mail.gmail.com), [#23](https://www.postgresql.org/message-id/CAMPh8MpLR8sax%3DfG6ShzkE_LF9JFL7MR8eyX3WrNguYp57Xpvw%40mail.gmail.com); `standards-compat-1`</sub>

## Questions for the author

1. Once the docs-only 0001 is applied, which concrete failures remain that pg_controldata plus pg_upgrade --check cannot catch before any data is copied?
2. What can --initdb do that a wrapper reading pg_controldata cannot, given that message #15 narrows the required match to WAL segment size and checksums?
3. Can --check --initdb initialize into a throwaway location, so that a failed state-file read or unconfirmed shutdown never leaves files that block the real run?
4. What makes a future initdb option that creates storage outside the data directory fail a test when the state file and its cleanup do not cover it?
5. Why does --initdb-options not take exactly one initdb argument per use, so pg_upgrade needs no quoting rules of its own?

## Touch points

- **on-disk format / pg_upgrade:** pg_upgrade now creates and removes the new cluster and its separate WAL directory; no on-disk format changes.
- **ABI / extension API:** The patch changes src/include/port.h, src/port/pgcheckdir.c and src/common/rmtree.c, which other frontend and backend code uses.
- **psql UX conventions:** No psql change, but pg_upgrade gains two options, a quoting grammar, and a --check that writes to disk.
- **security / privileges:** pg_upgrade removes directories and restores permissions from a state file, and passes user-supplied arguments to initdb.
- **maintenance burden ("who maintains this in 10 years"):** A hidden initdb option, a state file read by pg_upgrade, cleanup paths and a custom tokenizer need care in every release.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** Message #5 asks whether this belongs in wrappers such as pg_upgradecluster, and the thread has not settled it.

## Who would commit it, and when

- Owner: qualified committer engaged. Someone who knows both pg_upgrade and initdb would have to own it. Heikki Linnakangas shaped the hidden initdb option in message #24 but has not replied on the v9 and v10 cleanup design.
- Priority: fix design first; audience unclear; review cost high. The thread names the target users, operators who run pg_upgrade without a wrapper, but shows one reviewer's wish as demand. The v10 patch is 73 KiB across pg_upgrade, initdb, src/common and src/port, and its cleanup design and in-tree placement still lack agreement.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 4 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

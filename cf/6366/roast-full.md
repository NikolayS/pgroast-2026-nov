# Design pre-review: QoL improvements for pgindent and combining pgindent, pgperltidy, pgperlcritic and pgperlsyncheck into one tool

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6366](https://commitfest.postgresql.org/patch/6366/) (Needs review); authors: Jelte Fennema-Nio (jeltef)
- Thread: [Add "format" target to make and ninja to run pgindent and pgperltidy](https://www.postgresql.org/message-id/flat/DFCDD5H4J7VX.3GJKRBBDCKQ86%40jeltef.nl), started 2025-12-31
- Messages read: 21 of 21
- Patch prose read: v4-0001-pgindent-Use-git-ls-files-to-discover-files.patch, v4-0002-pgindent-Default-to-indenting-the-current-directo.patch, v4-0003-pgindent-Allow-parallel-pgindent-runs.patch, v4-0004-pgindent-Try-to-find-pg_bsd_indent-binary-in-comm.patch, v4-0005-Move-pgindent-to-separate-pgcheck-script.patch, v4-0006-Combine-all-formatting-and-linting-tools-into-pgc.patch, v4-0007-pgcheck-Add-easy-way-of-getting-perltidy.patch, v1-0001-Add-format-target-for-make-and-ninja.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The pgindent fixes in 0001 to 0004 have support in the thread, but three design choices in v4 lack agreement. Two committers object to the project shipping get_perltidy (already-exists-1, maintenance-fit-1). The README makes all tools the default, which no reviewer asked for (demand-2). Moving pgindent into pgcheck leaves the objection in message #12 unanswered (maintenance-fit-2). Nobody has replied to v4.

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: No code was read, so the signature check in get_perltidy, the parallel run on Windows and the behaviour outside a git tree could not be checked. Only 3 of 21 messages were visible; later messages are known through the objections.

## The proposal, restated

**Problem.** Running the source formatters is hard, mostly for new contributors. pgindent needs pg_bsd_indent in PATH and, with meson, picks up files in the build directory when run on "./". pgperltidy needs exactly perltidy 20230309, and pgperltidy, pgperlcritic and pgperlsyncheck lack pgindent flags such as --commit, --check, --diff and --jobs.

**Proposed design.** v1 added a make/ninja "format" target. v4 drops that and changes the scripts: pgindent finds files with git ls-files, defaults to the current directory, runs in parallel and searches build directories for pg_bsd_indent. A new src/tools/pgcheck/pgcheck combines pgindent, pgperltidy, pgperlcritic and pgperlsyncheck, keeps the old scripts as wrappers, and adds a get_perltidy script that downloads, verifies and installs perltidy into the source tree.

**User-visible surface.**
- New developer tool src/tools/pgcheck/pgcheck with options such as --pgindent, --perltidy, --perlcritic, --perl-syncheck, --commit, --check, --diff and --jobs.
- New script src/tools/pgindent/get_perltidy that installs perltidy 20230309 into src/tools/pgindent/perltidy/.
- pgindent with no arguments indents the current directory and, on a directory search, skips files git does not track.
- pgperltidy, pgperlcritic and pgperlsyncheck become wrappers around pgcheck and accept its flags.
- v1 only: "make format" and "ninja -C build format" targets, which v4 drops.

**What the author claims.**
- Parallel runs cut a full pgindent run from about 7 seconds to 1 second, and with perltidy from over a minute to about 7 seconds.
- get_perltidy checks signatures, so it is safer than the current README, which tells users to download an unverified tarball.
- Easier perltidy runs could later let perltidy run on every commit, like pgindent, though merging does not force that.
- Postgres developers all use git, so git ls-files is an acceptable dependency.
- The author says Claude Code wrote most of the v2 code; he reviewed it but is not a Perl expert.

**Thread state.** Peter Eisentraut committed the two temp-file cleanup patches and then removed himself as reviewer. Tom Lane wants the perltidy integration and get_perltidy rejected as a supply-chain risk, Daniel Gustafsson has the same worry, and Andrew Dunstan supported unifying the tools; nobody has replied to v4, which needs a rebase.

## 2. Prior attempts

No precedent found (corpus searched for items before 2025-12-31; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. get_perltidy commits the project to owning a downloader for CPAN code

fixable (first rated fatal; changed in the counter-check) · level 2, rubric item 15 · maintenance burden ("who maintains this in 10 years"), security / privileges, doesn't fit what Postgres is about (extension / external tool / already solved) · `maintenance-fit-1`

CPAN moves the tarball, or the signing key that get_perltidy trusts is rotated or compromised. A developer then runs src/tools/pgindent/get_perltidy, and it either fails or installs code that the project's own script told them to trust. Some committer must fix the script on every supported branch.

pg_bsd_indent became a commit requirement only after it lived in the tree, and Tom Lane says in message #19 that perltidy will never be in the tree and that no cleanup makes 0007 acceptable. Daniel Gustafsson raises the same supply-chain concern in message #17. The author says in message #20 that signature checks make the script safer than the README's unverified tarball, but the proposal does not say which key is checked or who maintains that trust anchor. The script also removes the manual install instructions, which Daniel asked to keep.

Thread: [#17 (Daniel Gustafsson, 2026-03-13)](https://www.postgresql.org/message-id/EAB73E05-6551-4537-A742-C091C9206B25%40yesql.se), [#19 (Tom Lane, 2026-03-27)](https://www.postgresql.org/message-id/1194980.1774623323%40sss.pgh.pa.us), [#20 (Jelte Fennema-Nio, 2026-03-27)](https://www.postgresql.org/message-id/CAGECzQTHHiLmS62LADQk%2BCMxoC5u0RtYzpjBbB7hKaq70YQn-Q%40mail.gmail.com)

**Question:** Which key does get_perltidy verify, where does that trust anchor come from, and which committer updates it when it changes?

Counter-check: held. Two committers object to the project shipping an installer, and 0007 removes the manual instructions that message #17 asked to keep. The script is one separable patch, so dropping it leaves 0001 to 0006 intact, which makes this fixable.

### 2. pgcheck ties pgindent to Perl tools with a different lifecycle

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years"), backpatchability · `maintenance-fit-2`

The project moves to a newer perltidy, or changes pgperlcritic rules. That change now edits the same script that every committer runs before each C commit and at the release-time indent run. A later pgindent fix on master also no longer applies cleanly to back branches, which still have src/tools/pgindent/pgindent.

Peter Eisentraut objected in message #12 that these tools run at different times and update differently, and he preferred separate make subtargets. v4 answers this by moving pgindent into a new combined pgcheck script (0005, 0006). That makes the coupling larger, and no committer has replied to v4. The useful pgindent changes in 0001 to 0004 do not need the merge.

Thread: [#12 (Peter Eisentraut, 2026-03-12)](https://www.postgresql.org/message-id/de91907e-fa32-4bec-968b-1099479dae16%40eisentraut.org), [#18 (Peter Eisentraut, 2026-03-27)](https://www.postgresql.org/message-id/4645a523-8774-45d8-894e-9c83d5954155%40eisentraut.org), [#21 ("Jelte Fennema-Nio", 2026-04-03)](https://www.postgresql.org/message-id/DHJCUQA4HF9H.10EN25MHG0YRW%40jeltef.nl)

**Question:** Why must pgindent itself move into pgcheck, when 0001 to 0004 could land alone and the Perl tools could share code without the move?

Counter-check: held. Peter Eisentraut's objection in message #12 has no answer in v4, which widens the coupling, and 0001 to 0004 do not need the merge. The back-branch part is weak, because the wrappers keep the old command names and pgindent fixes are rarely back-patched.

### 3. Git-only directory search changes behaviour for non-core trees

fixable · level 2, rubric item 12 · design, maintenance burden ("who maintains this in 10 years") · `maintenance-fit-4`

An extension maintainer runs pgindent on a directory in a tree that is not a git checkout, such as an unpacked tarball or another VCS. The directory search finds no tracked files and processes nothing, or exits with an error. The maintainer finds out only when their CI stops reformatting.

Peter Eisentraut noted in message #18 that pgindent is also used on trees that are not stock Postgres. The author answers in message #13 that all Postgres developers use git. v4 still indents files that are named explicitly, but the proposal does not say what a directory search does outside a git tree.

Thread: [#13 ("Jelte Fennema-Nio", 2026-03-13)](https://www.postgresql.org/message-id/DH1IHNI06K3E.35YUXK4EVN8D6%40jeltef.nl), [#18 (Peter Eisentraut, 2026-03-27)](https://www.postgresql.org/message-id/4645a523-8774-45d8-894e-9c83d5954155%40eisentraut.org)

**Question:** What does pgindent or pgcheck do on a directory argument outside a git work tree, and does it keep the old directory walk there?

Counter-check: held. Message #13 answers only for core developers, and message #18 later names non-stock trees as users of pgindent. The 0001 commit message does not say what a directory search does outside a git work tree.

### 4. get_perltidy repeats a one-line cpanm command and adds a download path

fixable · level 1b, rubric item 12 · doesn't fit what Postgres is about (extension / external tool / already solved), maintenance burden ("who maintains this in 10 years"), security / privileges · `already-exists-1`

The project ships get_perltidy, so it now owns a downloader, its trust anchor and its URL handling. When CPAN changes a mirror, key or archive layout, the script breaks for everyone using it. The README no longer gives the manual cpanm instructions to fall back on.

Message #10 shows that `cpanm -l $HOME/perltidy-20230309 Perl::Tidy@20230309` installs the exact version in one command, and pgperltidy already checks the version (message #4). The author answers that get_perltidy verifies signatures (message #20), but the material does not say which key it checks or where that key comes from. Messages #17 and #19 object to the project taking responsibility for installing code. Message #17 also asks to keep the manual instructions, yet the v4 README hunk removes them.

Thread: [#4 (Andrew Dunstan, 2025-12-31)](https://www.postgresql.org/message-id/5611b8aa-8496-4632-92b4-e096654850d1%40dunslane.net), [#10 (Dagfinn Ilmari Mannsåker, 2026-01-05)](https://www.postgresql.org/message-id/875x9g53q9.fsf%40wibble.ilmari.org), [#17 (Daniel Gustafsson, 2026-03-13)](https://www.postgresql.org/message-id/EAB73E05-6551-4537-A742-C091C9206B25%40yesql.se), [#19 (Tom Lane, 2026-03-27)](https://www.postgresql.org/message-id/1194980.1774623323%40sss.pgh.pa.us), [#20 (Jelte Fennema-Nio, 2026-03-27)](https://www.postgresql.org/message-id/CAGECzQTHHiLmS62LADQk%2BCMxoC5u0RtYzpjBbB7hKaq70YQn-Q%40mail.gmail.com)

**Question:** What does get_perltidy verify that the documented cpanm command does not, which key does it trust, and why must the project ship and maintain that check?

Counter-check: held. The cpanm command in message #10 installs the exact version in one step, so the existing route can do the job. The author's signature argument cannot be weighed against it until the proposal says what the script verifies.

### 5. Combining perltidy into the default run rests on a policy nobody agreed to

fixable · level 1b, rubric item 2 · desirability, maintenance burden ("who maintains this in 10 years") · `demand-2`

A contributor runs src/tools/pgcheck/pgcheck with no arguments, as the README suggests, and with the right perltidy installed it also reformats Perl files the patch never touched. The contributor finds out when the diff picks up unrelated files and must be cleaned by hand.

The pgcheck README says it runs all tools on the current directory by default. Message #2 notes that perltidy is not enforced, so whole-tree runs change files outside the patch. The stated reason for combining the tools is to get perltidy run on every commit (message #13), but message #20 calls that policy a separate, later discussion, and Peter Eisentraut (message #12) says the tools run at different times in practice. Andrew Dunstan (message #4) wanted unification only with perltidy off unless a flag is given.

Thread: [#2 (Ashutosh Bapat, 2025-12-31)](https://www.postgresql.org/message-id/CAExHW5tuJTmuikfkabWq%2BmvomucY3iQ_vV-10KAsZWM7Yg6tvg%40mail.gmail.com), [#4 (Andrew Dunstan, 2025-12-31)](https://www.postgresql.org/message-id/5611b8aa-8496-4632-92b4-e096654850d1%40dunslane.net), [#12 (Peter Eisentraut, 2026-03-12)](https://www.postgresql.org/message-id/de91907e-fa32-4bec-968b-1099479dae16%40eisentraut.org), [#13 ("Jelte Fennema-Nio", 2026-03-13)](https://www.postgresql.org/message-id/DH1IHNI06K3E.35YUXK4EVN8D6%40jeltef.nl), [#20 (Jelte Fennema-Nio, 2026-03-27)](https://www.postgresql.org/message-id/CAGECzQTHHiLmS62LADQk%2BCMxoC5u0RtYzpjBbB7hKaq70YQn-Q%40mail.gmail.com)

**Question:** Who, apart from the author, wants pgindent and the Perl tools to run together by default before perltidy is enforced on commits?

Counter-check: held. The README makes all tools the default, and message #2 reports that perltidy changes files outside the patch because the tree is not kept perltidy-clean. Message #4 supported unification only with perltidy behind a flag, and v4 does not answer that.

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
| security / privileges | **yes** | get_perltidy downloads third-party Perl code and installs it into the source tree on developer machines. |
| backpatchability | **yes** | 0005 moves pgindent to src/tools/pgcheck/pgcheck, so master and the back branches no longer share the script. |
| maintenance burden ("who maintains this in 10 years") | **yes** | The project takes on a combined pgcheck script, three wrappers and a downloader with a trust anchor. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | get_perltidy does a job that the cpanm command in message #10 does in one line. |

## 5. Questions the author must answer before anyone reviews code

1. Which key or checksum does get_perltidy verify against, and is it committed in the tree or fetched at run time?
2. What does get_perltidy verify that the cpanm command in message #10 does not, and why does 0007 remove the manual install instructions?
3. When perltidy 20230309 is not installed, does a bare pgcheck run skip perltidy, warn, or fail?
4. Why must pgindent move into pgcheck, when 0001 to 0004 could be committed alone and the Perl tools could share code without the move?
5. What does a directory run of pgindent or pgcheck do outside a git work tree, such as an unpacked tarball?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. It needs a committer who knows pgindent and the Perl tools under src/tools. Peter Eisentraut committed two small patches and then removed himself as reviewer, and no committer has replied to v4.
- **Liability:** The owner signs for a script that every committer runs before each commit, including parallel runs on Windows. With 0007 the owner also signs for a downloader and its trust anchor on each branch that has it.
- **Author follow-up:** The author posted four versions and moved the entry forward through October 2026. He says he is not a Perl expert, and the patch has needed a rebase since 2026-10-05.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** niche
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** Only Postgres developers and extension authors use these tools, and the patch set is about 40 KiB of Perl scripts with no backend code. 0001 to 0004 have support and can go ahead once the get_perltidy, default-run and pgcheck merge questions are settled or split off.

## Appendix

### Held up in the counter-check, not listed above

- `standards-compat-2` (fixable, level 2) Bare pgcheck runs every tool, including the version-pinned perltidy. Not listed: not among the strongest objections.
- `security-1` (fixable, level 2) get_perltidy does not say what its signature check trusts. Not listed: not among the strongest objections.
- `maintenance-fit-3` (fixable, level 2) No committer who knows these Perl tools has taken ownership of v4. Not listed: not among the strongest objections.

### Refuted in the counter-check

- `prior-attempts-1` get_perltidy does not meet the condition the project set for pgindent. Refuted (not specific): Duplicate of maintenance-fit-1 and security-1. Its framing around the pgindent-cleanliness condition is also answered in message #20, which says merging does not make perltidy a commit requirement.
- `standards-compat-1` Git-only directory search changes pgindent behaviour for non-git and untracked trees. Refuted (not specific): Duplicate of maintenance-fit-4. Its untracked-file case is the documented intent of 0001, which still indents files named explicitly on the command line.
- `demand-1` get_perltidy serves few users, and a one-line workaround exists. Refuted (not specific): Duplicate of already-exists-1. Its count of users is also weak evidence, because message #15 gives the install burden as the reason few people run perltidy today.

### Review angles

- prior-attempts: 1 objection(s). The only listed precedent [P14] is about making a pgindent check fail in CI for submitters; its reason does not apply here, because this proposal does not enforce formatting on anyone. No precedent found for unifying the tools.
- standards-compat: 2 objection(s). The get_perltidy supply-chain concern and perltidy enforcement policy belong to other angles. Here the main gaps are the default behaviour of pgcheck and of the git-based file search.
- replication-ondisk: 0 objection(s). The proposal changes only developer scripts under src/tools and README files. It touches no on-disk format, WAL, replication, catalog, wire protocol or extension ABI, so this angle has nothing to object to.
- security: 1 objection(s). The other changes have no privilege aspect: git ls-files discovery, the default directory, parallel runs, finding pg_bsd_indent and the pgcheck wrappers. The concern about odd file names was addressed in message #21.
- maintenance-fit: 4 objection(s). Objections are listed above. The precedent [P14] is about enforcing pgindent in CI, not about tooling or perltidy installs, so it is not cited.
- already-exists: 1 objection(s). The pgindent fixes (git ls-files, default directory, finding pg_bsd_indent, parallel runs) and the shared flags in pgcheck have no existing equivalent of the same effort, so this angle raises nothing against them.
- demand: 2 objection(s). The pgindent changes (git ls-files, current-directory default, parallel runs, finding pg_bsd_indent) have support from Tom Lane, Andrew Dunstan and Peter Eisentraut, so this angle raises nothing against them.

### Precedents retrieved

- P1: [Revert 9.5 pgindent changes to atomics directory files](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=befa3e648ce018d84cd2a0df701927c56fe3da4e) (revert, Reverted, 2015-05-25) - judged not relevant
- P2: [Revert pgindent changes to ecpg include files that are part of ecpg regession test output, and update pgindent script to avoid them in the future.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2cc6ff45f8813593e7c8c0757cd13a7b63a41768) (revert, Reverted, 2010-02-26) - judged not relevant
- P3: [Revert pgindent length back to 79 because we are going to fix the BSD indent bug.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=19cb4571465024d7068eddf3eda8c9fe6851cde1) (revert, Reverted, 2005-11-13) - judged not relevant
- P4: [Reverted changes made by pgindent Made show.pgc set all variables before displaying them.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=fcc2fbac22148ed640dff24caef7c57d20bdad06) (revert, Reverted, 2006-10-04) - judged not relevant
- P5: [Add llvm version into the version string](https://commitfest.postgresql.org/patch/5202/) (cf, Returned with feedback, 2024-09-24) - judged not relevant; reason summarised from its thread
- P6: [Unified file access using virtual file descriptors](https://commitfest.postgresql.org/patch/4420/) (cf, Returned with feedback, 2024-02-01) - judged not relevant; reason summarised from its thread
- P7: [Revert "pg_resetwal: Improve error with wrong/missing data directory"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=5daa50f962bd06e49af690efc2a8a27fdd37fb7d) (revert, Reverted, 2023-09-29) - judged not relevant
- P8: [Revert logical snapshot filename format change in SnapBuildSnapshotExists().](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e9a31c0cc604b015363d0aa5b018e8cea9d1d9a7) (revert, Reverted, 2025-08-14) - judged not relevant
- P9: [Package version in PG_VERSION and version()](https://commitfest.postgresql.org/patch/1418/) (cf, Rejected, 2018-01-19) - judged not relevant; reason summarised from its thread
- P10: [Avoid hiding shared filesets in pg_ls_tmpdir (pg_ls_* functions for showing metadata ...)](https://commitfest.postgresql.org/patch/2377/) (cf, Returned with feedback, 2024-01-22) - judged not relevant; reason summarised from its thread
- P11: [Allow an extention to be updated without a script](https://commitfest.postgresql.org/patch/4154/) (cf, Withdrawn, 2023-02-01) - judged not relevant; reason summarised from its thread
- P12: [Unicode mapping scripts cleanup](https://commitfest.postgresql.org/patch/360/) (cf, Returned with feedback, 2015-10-03) - judged not relevant; reason summarised from its thread
- P13: [Revert "Add redo LSN to pgstats files"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=5721e5453ebc59360b6756fe72d7499c4a02177c) (revert, Reverted, 2025-03-16) - judged not relevant
- P14: [Add code indentation check to cirrus-ci](https://commitfest.postgresql.org/patch/4691/) (cf, Withdrawn, 2024-01-24) - used in the review; reason summarised from its thread
- P15: [Make ON_ERROR_STOP stop on shell script failure](https://commitfest.postgresql.org/patch/3936/) (cf, Returned with feedback, 2023-07-06) - judged not relevant; reason summarised from its thread
- P16: [ICU integration](https://commitfest.postgresql.org/patch/766/) (cf, Returned with feedback, 2016-09-28) - judged not relevant
- P17: [pgbench with multi-line scripts](https://commitfest.postgresql.org/patch/319/) (cf, Returned with feedback, 2015-12-22) - judged not relevant
- P18: [pg_replslotdata - a tool for displaying replication slot information](https://commitfest.postgresql.org/patch/3425/) (cf, Rejected, 2022-01-17) - judged not relevant
- P19: [Reduce the time required for a database recovery from archive.](https://commitfest.postgresql.org/patch/2732/) (cf, Returned with feedback, 2020-12-01) - judged not relevant
- P20: [Revert "Remove hard coded formats for INT64 and use configured settings instead."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=348c10efe08f01872daeeb62f32c8e362fcbba69) (revert, Reverted, 2011-04-27) - judged not relevant
- P21: [Patch for migration of the pg_commit_ts directory](https://commitfest.postgresql.org/patch/5710/) (cf, Withdrawn, 2025-10-15) - judged not relevant
- P22: [Relax requirement for INTO with SELECT in pl/pgsql](https://commitfest.postgresql.org/patch/586/) (cf, Returned with feedback, 2016-09-22) - judged not relevant
- P23: [Revert "Add TAP test to automate the equivalent of check_guc"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=52377bb81a2c0427d357bba71e7149de02d016ce) (revert, Reverted, 2022-02-12) - judged not relevant
- P24: [Reduce the time required for a database recovery from archive.](https://commitfest.postgresql.org/patch/2935/) (cf, Returned with feedback, 2021-10-01) - judged not relevant

### Warnings

- attachment limit (8) reached; v1-0002-Allow-running-pgperltidy-from-any-directory.patch not read
- attachment limit (8) reached; v1-0003-Add-pgpertidy-to-format-target.patch not read

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 16; wall time: 180 s; cost reported by the CLI: $2.61
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 4 from cache, 4 summarised now, 0 failed; quotes: 3 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 106 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:49:41.950Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

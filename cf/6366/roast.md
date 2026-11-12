# Design pre-review: QoL improvements for pgindent and combining pgindent, pgperltidy, pgperlcritic and pgperlsyncheck into one tool

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6366](https://commitfest.postgresql.org/patch/6366/); [thread](https://www.postgresql.org/message-id/flat/DFCDD5H4J7VX.3GJKRBBDCKQ86%40jeltef.nl), 21 of 21 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The pgindent fixes in 0001 to 0004 have support in the thread, but three design choices in v4 lack agreement. Two committers object to the project shipping get_perltidy (already-exists-1, maintenance-fit-1). The README makes all tools the default, which no reviewer asked for (demand-2). Moving pgindent into pgcheck leaves the objection in message #12 unanswered (maintenance-fit-2). Nobody has replied to v4.

Not checked: No code was read, so the signature check in get_perltidy, the parallel run on Windows and the behaviour outside a git tree could not be checked. Only 3 of 21 messages were visible; later messages are known through the objections.

## Prior attempts

None found before 2025-12-31.

## Objections a reviewer is likely to raise

### 1. get_perltidy commits the project to owning a downloader for CPAN code

CPAN moves the tarball, or the signing key that get_perltidy trusts is rotated or compromised. A developer then runs src/tools/pgindent/get_perltidy, and it either fails or installs code that the project's own script told them to trust. Some committer must fix the script on every supported branch.

pg_bsd_indent became a commit requirement only after it lived in the tree, and Tom Lane says in message #19 that perltidy will never be in the tree and that no cleanup makes 0007 acceptable. Daniel Gustafsson raises the same supply-chain concern in message #17. The author says in message #20 that signature checks make the script safer than the README's unverified tarball, but the proposal does not say which key is checked or who maintains that trust anchor. The script also removes the manual install instructions, which Daniel asked to keep.

**Question:** Which key does get_perltidy verify, where does that trust anchor come from, and which committer updates it when it changes?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#17](https://www.postgresql.org/message-id/EAB73E05-6551-4537-A742-C091C9206B25%40yesql.se), [#19](https://www.postgresql.org/message-id/1194980.1774623323%40sss.pgh.pa.us), [#20](https://www.postgresql.org/message-id/CAGECzQTHHiLmS62LADQk%2BCMxoC5u0RtYzpjBbB7hKaq70YQn-Q%40mail.gmail.com); `maintenance-fit-1`</sub>

### 2. pgcheck ties pgindent to Perl tools with a different lifecycle

The project moves to a newer perltidy, or changes pgperlcritic rules. That change now edits the same script that every committer runs before each C commit and at the release-time indent run. A later pgindent fix on master also no longer applies cleanly to back branches, which still have src/tools/pgindent/pgindent.

Peter Eisentraut objected in message #12 that these tools run at different times and update differently, and he preferred separate make subtargets. v4 answers this by moving pgindent into a new combined pgcheck script (0005, 0006). That makes the coupling larger, and no committer has replied to v4. The useful pgindent changes in 0001 to 0004 do not need the merge.

**Question:** Why must pgindent itself move into pgcheck, when 0001 to 0004 could land alone and the Perl tools could share code without the move?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#12](https://www.postgresql.org/message-id/de91907e-fa32-4bec-968b-1099479dae16%40eisentraut.org), [#18](https://www.postgresql.org/message-id/4645a523-8774-45d8-894e-9c83d5954155%40eisentraut.org), [#21](https://www.postgresql.org/message-id/DHJCUQA4HF9H.10EN25MHG0YRW%40jeltef.nl); `maintenance-fit-2`</sub>

### 3. Git-only directory search changes behaviour for non-core trees

An extension maintainer runs pgindent on a directory in a tree that is not a git checkout, such as an unpacked tarball or another VCS. The directory search finds no tracked files and processes nothing, or exits with an error. The maintainer finds out only when their CI stops reformatting.

Peter Eisentraut noted in message #18 that pgindent is also used on trees that are not stock Postgres. The author answers in message #13 that all Postgres developers use git. v4 still indents files that are named explicitly, but the proposal does not say what a directory search does outside a git tree.

**Question:** What does pgindent or pgcheck do on a directory argument outside a git work tree, and does it keep the old directory walk there?

<sub>fixable, level 2, design; messages [#13](https://www.postgresql.org/message-id/DH1IHNI06K3E.35YUXK4EVN8D6%40jeltef.nl), [#18](https://www.postgresql.org/message-id/4645a523-8774-45d8-894e-9c83d5954155%40eisentraut.org); `maintenance-fit-4`</sub>

### 4. get_perltidy repeats a one-line cpanm command and adds a download path

The project ships get_perltidy, so it now owns a downloader, its trust anchor and its URL handling. When CPAN changes a mirror, key or archive layout, the script breaks for everyone using it. The README no longer gives the manual cpanm instructions to fall back on.

Message #10 shows that `cpanm -l $HOME/perltidy-20230309 Perl::Tidy@20230309` installs the exact version in one command, and pgperltidy already checks the version (message #4). The author answers that get_perltidy verifies signatures (message #20), but the material does not say which key it checks or where that key comes from. Messages #17 and #19 object to the project taking responsibility for installing code. Message #17 also asks to keep the manual instructions, yet the v4 README hunk removes them.

**Question:** What does get_perltidy verify that the documented cpanm command does not, which key does it trust, and why must the project ship and maintain that check?

<sub>fixable, level 1b, doesn't fit what Postgres is about (extension / external tool / already solved); messages [#4](https://www.postgresql.org/message-id/5611b8aa-8496-4632-92b4-e096654850d1%40dunslane.net), [#10](https://www.postgresql.org/message-id/875x9g53q9.fsf%40wibble.ilmari.org), [#17](https://www.postgresql.org/message-id/EAB73E05-6551-4537-A742-C091C9206B25%40yesql.se), [#19](https://www.postgresql.org/message-id/1194980.1774623323%40sss.pgh.pa.us), [#20](https://www.postgresql.org/message-id/CAGECzQTHHiLmS62LADQk%2BCMxoC5u0RtYzpjBbB7hKaq70YQn-Q%40mail.gmail.com); `already-exists-1`</sub>

### 5. Combining perltidy into the default run rests on a policy nobody agreed to

A contributor runs src/tools/pgcheck/pgcheck with no arguments, as the README suggests, and with the right perltidy installed it also reformats Perl files the patch never touched. The contributor finds out when the diff picks up unrelated files and must be cleaned by hand.

The pgcheck README says it runs all tools on the current directory by default. Message #2 notes that perltidy is not enforced, so whole-tree runs change files outside the patch. The stated reason for combining the tools is to get perltidy run on every commit (message #13), but message #20 calls that policy a separate, later discussion, and Peter Eisentraut (message #12) says the tools run at different times in practice. Andrew Dunstan (message #4) wanted unification only with perltidy off unless a flag is given.

**Question:** Who, apart from the author, wants pgindent and the Perl tools to run together by default before perltidy is enforced on commits?

<sub>fixable, level 1b, desirability; messages [#2](https://www.postgresql.org/message-id/CAExHW5tuJTmuikfkabWq%2BmvomucY3iQ_vV-10KAsZWM7Yg6tvg%40mail.gmail.com), [#4](https://www.postgresql.org/message-id/5611b8aa-8496-4632-92b4-e096654850d1%40dunslane.net), [#12](https://www.postgresql.org/message-id/de91907e-fa32-4bec-968b-1099479dae16%40eisentraut.org), [#13](https://www.postgresql.org/message-id/DH1IHNI06K3E.35YUXK4EVN8D6%40jeltef.nl), [#20](https://www.postgresql.org/message-id/CAGECzQTHHiLmS62LADQk%2BCMxoC5u0RtYzpjBbB7hKaq70YQn-Q%40mail.gmail.com); `demand-2`</sub>

## Questions for the author

1. Which key or checksum does get_perltidy verify against, and is it committed in the tree or fetched at run time?
2. What does get_perltidy verify that the cpanm command in message #10 does not, and why does 0007 remove the manual install instructions?
3. When perltidy 20230309 is not installed, does a bare pgcheck run skip perltidy, warn, or fail?
4. Why must pgindent move into pgcheck, when 0001 to 0004 could be committed alone and the Perl tools could share code without the move?
5. What does a directory run of pgindent or pgcheck do outside a git work tree, such as an unpacked tarball?

## Touch points

- **security / privileges:** get_perltidy downloads third-party Perl code and installs it into the source tree on developer machines.
- **backpatchability:** 0005 moves pgindent to src/tools/pgcheck/pgcheck, so master and the back branches no longer share the script.
- **maintenance burden ("who maintains this in 10 years"):** The project takes on a combined pgcheck script, three wrappers and a downloader with a trust anchor.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** get_perltidy does a job that the cpanm command in message #10 does in one line.

## Who would commit it, and when

- Owner: silent. It needs a committer who knows pgindent and the Perl tools under src/tools. Peter Eisentraut committed two small patches and then removed himself as reviewer, and no committer has replied to v4.
- Priority: fix design first; audience niche; review cost medium. Only Postgres developers and extension authors use these tools, and the patch set is about 40 KiB of Perl scripts with no backend code. 0001 to 0004 have support and can go ahead once the get_perltidy, default-run and pgcheck merge questions are settled or split off.

---

[roast-full.md](roast-full.md) has 3 more objection(s) that held, 3 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 2.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

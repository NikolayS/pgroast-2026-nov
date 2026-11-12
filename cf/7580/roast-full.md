# Design pre-review: doc: Fix description of relative extension "directory" setting

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7580](https://commitfest.postgresql.org/patch/7580/) (Needs review); authors: mahendrakar s (mahendrakarsrinivasarao)
- Thread: [\[PATCH\] doc: Fix description of relative extension "directory" setting](https://www.postgresql.org/message-id/flat/CABkiuWpGiLzDg4D8zRBJ6DHnwYgHqkFxz2gDstXgeOdzh6bdZQ%40mail.gmail.com), started 2026-10-31
- Messages read: 7 of 7
- Patch prose read: v4-0001-doc-Fix-description-of-relative-extension-directo.patch, v4-0002-doc-Fix-description-of-system-in-extension_contro.patch, v4-0003-Fix-long-description-of-extension_control_path.patch, 0001-doc-Fix-description-of-relative-extension-directory.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium)

No objection survived at levels 1a, 1b or 2. The patches change documentation and one GUC description string so that they match the code and the TAP test already shipped in 18, and message #1 gives a reproducible case. The remaining open points are the backpatch to 18 and 19, which no committer has agreed to yet, and the wording of 0003, which is not shown.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The 0003 text and the code of get_extension_control_directories() were not read, so the exact-match $system claim and the new long_desc wording rest on the commit messages. The function quoted in message #2 supports the 0001 claim.

## The proposal, restated

**Problem.** Since Postgres 18 (commit 81eaaa2c41d), the docs say a relative control file "directory" is resolved against the control file's directory. The code resolves it against the parent of the "extension" directory, that is SHAREDIR or the extension_control_path element. The docs also say an element that starts with $system is expanded, but only an exact "$system" is, which misleads extension authors and administrators.

**Proposed design.** 0001 restores the pre-18 wording of "directory" in extend.sgml, extended to cover extension_control_path, and notes in config.sgml that "directory" can move script files elsewhere. 0002 says an element must be exactly $system and gets no "extension" suffix. 0003 changes the extension_control_path long description in guc_parameters.dat to match, on master only.

**User-visible surface.**
- Docs: description of the "directory" control file parameter in extend.sgml.
- Docs: description of extension_control_path in config.sgml, covering "directory" and $system.
- GUC long_desc text of extension_control_path, a translatable string, master only.

**What the author claims.**
- The 18 docs do not match the code; a repro shows scripts load from the base directory.
- The TAP test in src/test/modules/test_extensions expects the base directory behavior.
- The base directory rule keeps the "directory = 'extension/foo'" MODULEDIR convention working.
- 0001 and 0002 apply cleanly to REL_19_STABLE and REL_18_STABLE and should be backpatched to 18.
- "$system/foo" makes CREATE EXTENSION fail, while pg_available_extensions silently skips the element.

**Thread state.** Reviewer Matheus Alcantara (who wrote 0002) and Chao Li both say v4 looks good, and Matheus plans to mark it Ready for Committer. The author's idea of a check hook for invalid elements was moved to a separate thread.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-10-31; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

No objection held up strongly enough in the counter-check to list here.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | 0001 changes the documented rule that extension authors follow for the "directory" control file parameter; the behavior stays the same. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | **yes** | No new GUC, but 0002 and 0003 change the documented description and long_desc of the existing extension_control_path. |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | 0001 and 0002 ask for a backpatch to 18; 0003 changes a translatable string and stays on master. |
| maintenance burden ("who maintains this in 10 years") | no |  |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Does a committer agree to backpatch 0001 and 0002 to REL_18_STABLE and REL_19_STABLE?
2. What is the new long_desc text in 0003, and does it match the config.sgml wording from 0001?
3. Should the docs for 18 also say that pg_available_extensions silently skips an element like "$system/foo" while CREATE EXTENSION fails?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. A committer who knows the extension loading code and extension_control_path would have to confirm the wording and the backpatch; none has replied in the visible thread.
- **Liability:** Small: the committer signs for the accuracy of two documentation paragraphs in 18, 19 and master, and one translatable GUC string on master.
- **Author follow-up:** The author posted revisions up to v4 within about a week in response to review, and moved the check hook idea to a separate thread.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patches are a few kilobytes of documentation that correct text contradicted by the code and by the repro in message #1. Two reviewers say v4 looks good; the audience is extension authors and administrators who set "directory" or extension_control_path.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent found: the precedent search returned no proposal about the same or a similar idea. The thread points to only one earlier discussion: message #6 links an extension_control_path bug about empty path elements, which is a separate code issue and was not rejected. This docs fix gives no prior rejection reason to apply.
- standards-compat: 0 objection(s). The patch changes no behavior. It brings the 18 docs in line with what the code and the TAP test already do, so no query, script or dump behaves differently after upgrade. The 0002 choice of exact-match $system has a stated reason, because $system already names the extension directory, and the prefix form never worked. Keeping the translatable long_desc change on master only follows existing backpatch practice.
- replication-ondisk: 0 objection(s). The patches change only documentation and, in 0003, the GUC long_desc string. Nothing in them touches on-disk format, WAL, replication, catalogs, the wire protocol or the extension ABI. A translatable string is a backpatch concern, and 0003 already keeps it on master only.
- security: 0 objection(s). This is a docs-only fix plus one GUC long_desc string, and it does not change who can do what. extension_control_path and the "directory" behavior already exist. Documenting the real resolution rule and the exact-match $system rule exposes no new privilege, data, or file system access. A correct description helps administrators see where script files load from.
- maintenance-fit: 0 objection(s). This is a docs-only fix that makes the text match code and tests already shipped in 18, so it adds no hot-path cost, API or new mechanism to maintain. Keeping the translatable long_desc change on master only follows the usual practice. 0002 documents the exact-match $system rule and gives its reason, a sound one, in the commit message. No committer has replied yet, so the backpatch to 18 has no committer agreement.
- already-exists: 0 objection(s). This is a docs-only fix that makes the documentation match existing behavior. No existing feature, extension or tool supplies correct documentation, so nothing is duplicated. The open questions, such as backpatch agreement and whether to change the $system code instead, belong to other angles.
- demand: 0 objection(s). This is a docs-only fix where the current text in 18 describes behavior the code does not have, and message #1 shows a reproducible case. Every user of the "directory" and $system settings is affected, so demand is not in question. No objection from the demand angle.

### Precedents retrieved

- P1: [Strip Windows newlines from extension script files manually.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6cfb3a33746990602c790199adc2debb2c4bbb87) (revert, Partially reverted, 2024-10-28) - judged not relevant
- P2: [Configurable location for extension .control files](https://commitfest.postgresql.org/patch/170/) (cf, Returned with feedback, 2015-07-30) - judged not relevant; reason summarised from its thread
- P3: [Refactor add_path for clarity and extensibility](https://commitfest.postgresql.org/patch/3599/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P4: [Document creating an extension in replication setup](https://commitfest.postgresql.org/patch/3494/) (cf, Returned with feedback, 2022-04-07) - judged not relevant; reason summarised from its thread
- P5: [Add control-C support for some psql meta-commands](https://commitfest.postgresql.org/patch/3412/) (cf, Withdrawn, 2021-11-15) - judged not relevant; reason summarised from its thread
- P6: [Revert exporting of internal GUC variable "data_directory".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=da33a3894e0fc440ac53cdc0f2e360e703b13b8c) (revert, Reverted, 2015-05-29) - judged not relevant
- P7: [logical replication access control](https://commitfest.postgresql.org/patch/1033/) (cf, Returned with feedback, 2017-04-06) - judged not relevant; reason summarised from its thread
- P8: [A separate table level option to control compression](https://commitfest.postgresql.org/patch/2001/) (cf, Returned with feedback, 2019-04-08) - judged not relevant; reason summarised from its thread
- P9: [Checkpoint extension hook](https://commitfest.postgresql.org/patch/4868/) (cf, Returned with feedback, 2024-07-23) - judged not relevant; reason summarised from its thread
- P10: [Support % wildcard in extension upgrade scripts](https://commitfest.postgresql.org/patch/3654/) (cf, Returned with feedback, 2023-08-01) - judged not relevant; reason summarised from its thread
- P11: [Fast path for empty relids in check_outerjoin_delay()](https://commitfest.postgresql.org/patch/1914/) (cf, Returned with feedback, 2019-01-31) - judged not relevant; reason summarised from its thread
- P12: [Support % wildcard in extension upgrade scripts](https://commitfest.postgresql.org/patch/3892/) (cf, Withdrawn, 2022-09-14) - judged not relevant
- P13: [Extend docs on how to write correct postgres-dialect C including extensions and bgworkers](https://commitfest.postgresql.org/patch/2953/) (cf, Returned with feedback, 2021-11-04) - judged not relevant
- P14: [A hook for path-removal decision on add_path](https://commitfest.postgresql.org/patch/2234/) (cf, Returned with feedback, 2021-10-05) - judged not relevant
- P15: [Support % wildcard in extension upgrade script filenames](https://commitfest.postgresql.org/patch/4489/) (cf, Returned with feedback, 2024-02-01) - judged not relevant
- P16: [Per-table storage parameters for TableAM/IndexAM extensions](https://commitfest.postgresql.org/patch/3495/) (cf, Returned with feedback, 2022-08-02) - judged not relevant
- P17: [(ALTER SYSTEM) SET +=/-= operator support](https://commitfest.postgresql.org/patch/2750/) (cf, Returned with feedback, 2021-03-12) - judged not relevant
- P18: [New hooks in the connection path](https://commitfest.postgresql.org/patch/3735/) (cf, Returned with feedback, 2023-04-04) - judged not relevant
- P19: [Mingw: Fix import library extension, build actual static libraries](https://commitfest.postgresql.org/patch/2082/) (cf, Returned with feedback, 2019-09-30) - judged not relevant
- P20: [Extending the range operators so they can cope with range <op> element and element <op> range in addition to the existing range <op> range](https://commitfest.postgresql.org/patch/2747/) (cf, Returned with feedback, 2021-04-08) - judged not relevant
- P21: [pg_receivewal fail to streams when the partial file to write is not fully initialized present in the wal receiver directory](https://commitfest.postgresql.org/patch/3503/) (cf, Returned with feedback, 2024-01-22) - judged not relevant
- P22: [Revert "pg_resetwal: Improve error with wrong/missing data directory"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=5daa50f962bd06e49af690efc2a8a27fdd37fb7d) (revert, Reverted, 2023-09-29) - judged not relevant
- P23: [introduce optimized linear search functions that return index of matching element](https://commitfest.postgresql.org/patch/3906/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P24: [Revert "Add TAP test to automate the equivalent of check_guc"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=52377bb81a2c0427d357bba71e7149de02d016ce) (revert, Reverted, 2022-02-12) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 18; wall time: 104 s; cost reported by the CLI: $1.49
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 0 from cache, 8 summarised now, 0 failed; quotes: 7 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 44 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T01:01:45.112Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

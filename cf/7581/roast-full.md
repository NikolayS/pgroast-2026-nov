# Design pre-review: extension_control_path: ignore empty elements and deduplicate directories

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7581](https://commitfest.postgresql.org/patch/7581/) (Needs review); authors: Chao Li (chaol)
- Thread: [extension_control_path: ignore empty elements and deduplicate directories](https://www.postgresql.org/message-id/flat/DC2D9FA5-45C4-494F-99FC-F9B95AD4FF55%40gmail.com), started 2026-11-03
- Messages read: 9 of 9
- Patch prose read: v4-0001-Reject-empty-elements-in-extension_control_path.patch, v4-0002-Reject-relative-paths-consistently-in-extension_c.patch, v1-0001-Ignore-empty-elements-in-extension_control_path.patch, v1-0002-Deduplicate-extension-control-search-directories.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The goal is a valid bug fix and three reviewers reproduced the defect. The open point is the behaviour for invalid elements: v4 raises errors on every lookup, while message #9 reports an off-list committer preference for skipping them (standards-compat-1). The thread also has not chosen one rule for 18, 19 and master (standards-compat-2). Both are fixable, but the list must agree on them before code review is useful.

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read. Whether every caller of get_extension_control_directories() passes the right skip_invalid_path value, and whether the canonicalization changes in v4-0002 alter the reported locations, could not be checked.

## The proposal, restated

**Problem.** An empty element in extension_control_path, as in '$system:', expands to the literal directory /extension, which is then searched and can supply extensions. The views also list extensions from relative elements that CREATE EXTENSION later rejects. This affects anyone who sets the parameter with a stray separator or a relative path.

**Proposed design.** v4-0001 makes get_extension_control_directories() reject empty elements with the existing "zero-length component" error, and v4-0002 moves the absolute-path check there too. extension_file_exists(), used for the unknown-language hint, skips such elements instead, so the original CREATE FUNCTION or DO error is kept. The v1 directory deduplication patch was dropped in v3.

**User-visible surface.**
- extension_control_path: a nonempty value with an empty element now raises "zero-length component in parameter" on every lookup; an entirely empty value still means $system.
- extension_control_path: a relative element now makes pg_available_extensions, pg_available_extension_versions and CREATE EXTENSION fail with "is not an absolute path".
- CREATE FUNCTION and DO with an unknown language keep the "language does not exist" error and skip invalid elements when building the hint.
- config.sgml adds the sentence "Empty elements within a nonempty value are not allowed."

**What the author claims.**
- Searching /extension because of a trailing colon is unintended and easy to trigger by mistake.
- The views and CREATE EXTENSION have disagreed on empty and relative elements since the parameter was added in 18.
- The author thinks the error-raising change does not need a back-patch.
- Deduplicating directories was low value but harmless, for example with extension directories on network storage.

**Thread state.** Three reviewers took part, and two of them objected to deduplication, so the author dropped it. In the last message Matheus reports that Peter, off list, prefers silently ignoring invalid elements, which conflicts with v4 raising errors, and no new version follows.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-11-03; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Rejecting empty and relative elements breaks settings that work today

fixable · level 2, rubric item 12 · backpatchability · `standards-compat-1`

An admin on 18 has extension_control_path = '$system:' or '$system:$system/foo' in postgresql.conf, and extensions install fine. After upgrading, the server starts without complaint, but every pg_available_extensions query and every CREATE EXTENSION fails with "zero-length component" or "is not an absolute path". Any role running the views or a tool that lists extensions finds out first, not the admin.

Message #5 and message #7 show that a trailing separator and a trailing relative element both work on 18 and become errors with v4 on every lookup. The check runs at use time, so SET and ALTER SYSTEM accept the value and the failure appears later, possibly for unprivileged roles. Message #9 says nonexistent directories are skipped today and reports that the off-list committer view is to skip invalid elements the same way. v4 still raises errors, so it departs from the project's existing handling of path elements that do not apply.

Thread: [#5 ("Matheus Alcantara", 2026-11-09)](https://www.postgresql.org/message-id/DMQY63XYCO3K.59C7KPX3CQXI%40gmail.com), [#7 ("Matheus Alcantara", 2026-11-10)](https://www.postgresql.org/message-id/DMRS0Z8S19SG.3NFZJL43WDEKC%40gmail.com), [#9 (Matheus Alcantara, 2026-11-11)](https://www.postgresql.org/message-id/6cc26872-dfda-42f1-9b79-4001e58f71fd%40gmail.com)

**Question:** Why should a value that works on 18 fail on every extension lookup after upgrade, when skipping it matches how nonexistent directories are already treated?

Counter-check: held. Messages #5 and #7 confirm that '$system:' and '$system:$system/foo' work on 18 and fail on every lookup with v4. Message #9 reports a committer preference for skipping, and no later version answers it. Raising at use time does match dynamic_library_path, so the consistency point is weaker than stated.

### 2. Back-branch behaviour left open gives one GUC different meanings by version

fixable · level 2, rubric item 6 · backpatchability · `standards-compat-2`

An operator manages 18, 19 and 20 servers from one configuration template that contains '$system:'. If master raises errors while 18 and 19 skip the element or keep searching /extension, the same template gives three different results. The operator finds out when extension listing fails on only the newest cluster.

In message #6 the author proposes erroring on master and perhaps skipping silently on released branches. Message #7 objects that the same setting should not behave differently across versions of a parameter that was only added in 18. The thread does not settle this, and v4 does not say which behaviour 18 and 19 get. A committer would likely ask for one rule for all branches before committing anything.

Thread: [#6 (Chao Li, 2026-11-10)](https://www.postgresql.org/message-id/1CB2B5A9-1BA6-4F5D-A3D2-28424E8BBF1C%40gmail.com), [#7 ("Matheus Alcantara", 2026-11-10)](https://www.postgresql.org/message-id/DMRS0Z8S19SG.3NFZJL43WDEKC%40gmail.com), [#9 (Matheus Alcantara, 2026-11-11)](https://www.postgresql.org/message-id/6cc26872-dfda-42f1-9b79-4001e58f71fd%40gmail.com)

**Question:** Which single behaviour for empty and relative elements will 18, 19 and 20 share, and will v4 change to match it?

Counter-check: held. The author suggests errors on master and perhaps skipping on released branches in message #6. Message #7 asks for one rule on all branches, and the thread ends without a decision or a matching patch version.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | v4-0002 makes find_in_paths() static, which matters if any of this is back-patched. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | **yes** | No new GUC, but v4 changes which values of extension_control_path are usable and documents the restriction in config.sgml. |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | **yes** | A '$system:' setting carried into a new cluster would make CREATE EXTENSION fail during restore. |
| psql UX conventions | no |  |
| security / privileges | **yes** | The patch stops an unintended search of /extension; the v1 DEBUG3 message that showed paths to any role was dropped. |
| backpatchability | **yes** | The defect exists in 18 and 19, and the thread has not decided what those branches get. |
| maintenance burden ("who maintains this in 10 years") | **yes** | get_extension_control_directories() gains a skip_invalid_path mode that each caller must choose. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Will the next version skip invalid elements silently, as message #9 reports, or keep the v4 errors, and why?
2. If empty elements are skipped, are relative elements skipped too, or do they still raise "is not an absolute path"?
3. Which single behaviour for empty and relative elements will 18, 19 and master share?
4. If errors are kept, why not reject the value in a check hook at SET time instead of on every lookup?
5. Should the CommitFest entry title drop "deduplicate directories", since the thread dropped that patch?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows extension.c and the extension_control_path code from 18. Message #9 relays a committer view from off list, but no committer has posted in the visible thread.
- **Liability:** The owner signs for a behaviour change to a GUC added in 18: either a new error on settings that work today, or a silent skip that stays the rule on all branches.
- **Author follow-up:** The author posted four versions in about a week and dropped the deduplication patch after review. No version follows message #9 yet.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The fix is two small patches in extension.c for a mistake that only affects clusters with a stray separator or a relative element. The error-or-skip choice and the branch scope must be settled before a committer can act.

## Appendix

### Refuted in the counter-check

- `maintenance-fit-1` Two validation modes in one helper, against the off-list committer preference. Refuted (no failure scenario): A future lookup that raises on an invalid setting is what v4 intends, and the error is visible. Message #5 found a missing case in v2, before the flag existed, and the conflict with message #9 is covered by standards-compat-1.
- `maintenance-fit-2` Release branches may behave differently for one setting. Refuted (not specific): Duplicate of standards-compat-2 for the branch question. Its failure scenario repeats the upgrade breakage in standards-compat-1.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. [P2] made the extension control directory configurable, and [P14] has no recorded reason, so neither reason applies to path validation. Message #7 points to an earlier documentation thread with the same '$system/foo' inconsistency; that thread asked about a check hook and did not reject the idea. Message #9 reports an off-list committer preference for silently skipping invalid elements, which v4 has not yet followed.
- standards-compat: 2 objection(s). Other parts of the contract look consistent. v4 reuses the existing dynamic_library_path error message, keeps '' meaning $system, keeps the unknown-language error, and dropped deduplication.
- replication-ondisk: 0 objection(s). The patch changes only how extension.c parses a superuser-only GUC. It adds no on-disk, WAL, catalog, protocol or replication state. Two side points belong to other angles: a '$system:' setting carried into a v20 cluster could fail CREATE EXTENSION during restore, and back-patching would need find_in_paths() to stay non-static.
- security: 0 objection(s). No security objection. The only exposure in the thread, the DEBUG3 message that showed the share path to any role (message #3), was dropped from v4. The error texts name the parameter but not its value, and only superusers can set extension_control_path.
- maintenance-fit: 2 objection(s). The proposal is a small, contained bugfix in extension.c with no hot-path cost. The objections above concern only which behaviour to keep and how many modes the code must carry.
- already-exists: 0 objection(s). This is a bug fix to an existing parameter. No existing feature, extension or tool can stop extension_control_path from turning an empty element into /extension, so the need is not already met. The dropped deduplication patch would have been the only duplication of existing behaviour, since the views already deduplicate by extension name.
- demand: 0 objection(s). This is a bug fix, so demand is shown by the defect itself: a trailing colon makes the server search /extension, and the views disagree with CREATE EXTENSION on relative elements. Prashant (message #2), Surya (message #3) and Matheus (message #5) reproduced both. The dedup patch, the only part with weak demand, was dropped in message #6. Error versus silent skip is a design and compatibility question for other angles.

### Precedents retrieved

- P1: [Refactor add_path for clarity and extensibility](https://commitfest.postgresql.org/patch/3599/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P2: [Configurable location for extension .control files](https://commitfest.postgresql.org/patch/170/) (cf, Returned with feedback, 2015-07-30) - used in the review; reason summarised from its thread
- P3: [Fast path for empty relids in check_outerjoin_delay()](https://commitfest.postgresql.org/patch/1914/) (cf, Returned with feedback, 2019-01-31) - judged not relevant; reason summarised from its thread
- P4: [Strip Windows newlines from extension script files manually.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6cfb3a33746990602c790199adc2debb2c4bbb87) (revert, Partially reverted, 2024-10-28) - judged not relevant
- P5: [Fix search_path for maintenance commands](https://commitfest.postgresql.org/patch/4367/) (cf, Withdrawn, 2024-01-18) - judged not relevant; reason summarised from its thread
- P6: [Permute underscore separated components of columns before fuzzy matching](https://commitfest.postgresql.org/patch/4282/) (cf, Returned with feedback, 2024-10-07) - judged not relevant; reason summarised from its thread
- P7: [Allow excluding pg_temp or pg_catalog from search_path.](https://commitfest.postgresql.org/patch/4508/) (cf, Withdrawn, 2023-11-17) - judged not relevant; reason summarised from its thread
- P8: [Revert "Fix search_path to a safe value during maintenance operations."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2fcc7ee7affe2fb708d7b2727e93981216b5d2f8) (revert, Reverted, 2023-06-10) - judged not relevant
- P9: [Add checks on search_path for IMMUTABLE and SECURITY DEFINER functions](https://commitfest.postgresql.org/patch/3684/) (cf, Returned with feedback, 2022-10-12) - judged not relevant; reason summarised from its thread
- P10: [Revert "pg_resetwal: Improve error with wrong/missing data directory"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=5daa50f962bd06e49af690efc2a8a27fdd37fb7d) (revert, Reverted, 2023-09-29) - judged not relevant
- P11: [Revert exporting of internal GUC variable "data_directory".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=da33a3894e0fc440ac53cdc0f2e360e703b13b8c) (revert, Reverted, 2015-05-29) - judged not relevant
- P12: [pg_regress.c: Fix "make check" on Mac OS X: Pass DYLD_LIBRARY_PATH](https://commitfest.postgresql.org/patch/4340/) (cf, Returned with feedback, 2023-12-02) - judged not relevant; reason summarised from its thread
- P13: [makes pgbench functions available on server side - Common Library Extraction (patch 1 - refactoring only) & Server Extension (patch 2 adds extension)](https://commitfest.postgresql.org/patch/7225/) (cf, Withdrawn, 2026-10-20) - judged not relevant; reason summarised from its thread
- P14: [Expand the use of check_canonical_path() for more GUCs](https://commitfest.postgresql.org/patch/2564/) (cf, Returned with feedback, 2020-07-10) - used in the review
- P15: [Revert "Validate subscription conninfo on owner change"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ac230cd3ce93c4752489564cfe6b37b584a465ac) (revert, Reverted, 2026-08-05) - judged not relevant
- P16: [A hook for path-removal decision on add_path](https://commitfest.postgresql.org/patch/2234/) (cf, Returned with feedback, 2021-10-05) - judged not relevant
- P17: [Fix checksum verification in base backups for zero page headers](https://commitfest.postgresql.org/patch/2719/) (cf, Returned with feedback, 2021-09-13) - judged not relevant
- P18: [Revert "Add TAP test to automate the equivalent of check_guc"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=52377bb81a2c0427d357bba71e7149de02d016ce) (revert, Reverted, 2022-02-12) - judged not relevant
- P19: [Fix base backup checksum verification for random or zero page headers](https://commitfest.postgresql.org/patch/2308/) (cf, Returned with feedback, 2020-07-30) - judged not relevant
- P20: [New hooks in the connection path](https://commitfest.postgresql.org/patch/3735/) (cf, Returned with feedback, 2023-04-04) - judged not relevant
- P21: [Checkpoint extension hook](https://commitfest.postgresql.org/patch/4868/) (cf, Returned with feedback, 2024-07-23) - judged not relevant
- P22: [Extending the range operators so they can cope with range <op> element and element <op> range in addition to the existing range <op> range](https://commitfest.postgresql.org/patch/2747/) (cf, Returned with feedback, 2021-04-08) - judged not relevant
- P23: [Dynamic result sets from procedures](https://commitfest.postgresql.org/patch/2911/) (cf, Withdrawn, 2023-10-30) - judged not relevant
- P24: [introduce optimized linear search functions that return index of matching element](https://commitfest.postgresql.org/patch/3906/) (cf, Returned with feedback, 2022-10-12) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 20; wall time: 140 s; cost reported by the CLI: $2.26
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 0 from cache, 8 summarised now, 0 failed; quotes: 7 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 66 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T01:01:51.629Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

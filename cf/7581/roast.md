# Design pre-review: extension_control_path: ignore empty elements and deduplicate directories

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7581](https://commitfest.postgresql.org/patch/7581/); [thread](https://www.postgresql.org/message-id/flat/DC2D9FA5-45C4-494F-99FC-F9B95AD4FF55%40gmail.com), 9 of 9 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The goal is a valid bug fix and three reviewers reproduced the defect. The open point is the behaviour for invalid elements: v4 raises errors on every lookup, while message #9 reports an off-list committer preference for skipping them (standards-compat-1). The thread also has not chosen one rule for 18, 19 and master (standards-compat-2). Both are fixable, but the list must agree on them before code review is useful.

Not checked: The code was not read. Whether every caller of get_extension_control_directories() passes the right skip_invalid_path value, and whether the canonicalization changes in v4-0002 alter the reported locations, could not be checked.

## Prior attempts

None found before 2026-11-03.

## Objections a reviewer is likely to raise

### 1. Rejecting empty and relative elements breaks settings that work today

An admin on 18 has extension_control_path = '$system:' or '$system:$system/foo' in postgresql.conf, and extensions install fine. After upgrading, the server starts without complaint, but every pg_available_extensions query and every CREATE EXTENSION fails with "zero-length component" or "is not an absolute path". Any role running the views or a tool that lists extensions finds out first, not the admin.

Message #5 and message #7 show that a trailing separator and a trailing relative element both work on 18 and become errors with v4 on every lookup. The check runs at use time, so SET and ALTER SYSTEM accept the value and the failure appears later, possibly for unprivileged roles. Message #9 says nonexistent directories are skipped today and reports that the off-list committer view is to skip invalid elements the same way. v4 still raises errors, so it departs from the project's existing handling of path elements that do not apply.

**Question:** Why should a value that works on 18 fail on every extension lookup after upgrade, when skipping it matches how nonexistent directories are already treated?

<sub>fixable, level 2, backpatchability; messages [#5](https://www.postgresql.org/message-id/DMQY63XYCO3K.59C7KPX3CQXI%40gmail.com), [#7](https://www.postgresql.org/message-id/DMRS0Z8S19SG.3NFZJL43WDEKC%40gmail.com), [#9](https://www.postgresql.org/message-id/6cc26872-dfda-42f1-9b79-4001e58f71fd%40gmail.com); `standards-compat-1`</sub>

### 2. Back-branch behaviour left open gives one GUC different meanings by version

An operator manages 18, 19 and 20 servers from one configuration template that contains '$system:'. If master raises errors while 18 and 19 skip the element or keep searching /extension, the same template gives three different results. The operator finds out when extension listing fails on only the newest cluster.

In message #6 the author proposes erroring on master and perhaps skipping silently on released branches. Message #7 objects that the same setting should not behave differently across versions of a parameter that was only added in 18. The thread does not settle this, and v4 does not say which behaviour 18 and 19 get. A committer would likely ask for one rule for all branches before committing anything.

**Question:** Which single behaviour for empty and relative elements will 18, 19 and 20 share, and will v4 change to match it?

<sub>fixable, level 2, backpatchability; messages [#6](https://www.postgresql.org/message-id/1CB2B5A9-1BA6-4F5D-A3D2-28424E8BBF1C%40gmail.com), [#7](https://www.postgresql.org/message-id/DMRS0Z8S19SG.3NFZJL43WDEKC%40gmail.com), [#9](https://www.postgresql.org/message-id/6cc26872-dfda-42f1-9b79-4001e58f71fd%40gmail.com); `standards-compat-2`</sub>

## Questions for the author

1. Will the next version skip invalid elements silently, as message #9 reports, or keep the v4 errors, and why?
2. If empty elements are skipped, are relative elements skipped too, or do they still raise "is not an absolute path"?
3. Which single behaviour for empty and relative elements will 18, 19 and master share?
4. If errors are kept, why not reject the value in a check hook at SET time instead of on every lookup?
5. Should the CommitFest entry title drop "deduplicate directories", since the thread dropped that patch?

## Touch points

- **ABI / extension API:** v4-0002 makes find_in_paths() static, which matters if any of this is back-patched.
- **new GUC:** No new GUC, but v4 changes which values of extension_control_path are usable and documents the restriction in config.sgml.
- **pg_dump / restore round-trip:** A '$system:' setting carried into a new cluster would make CREATE EXTENSION fail during restore.
- **security / privileges:** The patch stops an unintended search of /extension; the v1 DEBUG3 message that showed paths to any role was dropped.
- **backpatchability:** The defect exists in 18 and 19, and the thread has not decided what those branches get.
- **maintenance burden ("who maintains this in 10 years"):** get_extension_control_directories() gains a skip_invalid_path mode that each caller must choose.

## Who would commit it, and when

- Owner: silent. Someone who knows extension.c and the extension_control_path code from 18. Message #9 relays a committer view from off list, but no committer has posted in the visible thread.
- Priority: fix design first; audience niche; review cost low. The fix is two small patches in extension.c for a mistake that only affects clusters with a stray separator or a relative element. The error-or-skip choice and the branch scope must be settled before a committer can act.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 2 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

# Design pre-review: pageinspect: validate line pointers before using them

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7229](https://commitfest.postgresql.org/patch/7229/) (Needs review); authors: shihao zhong (zhong950419)
- Thread: [\[PATCH\] pageinspect: validate line pointers before using them](https://www.postgresql.org/message-id/flat/CAGRkXqRWDJR7Ma%3D589-cOFaOK2pG-XxUgXzYvje9eMxxSiOm8g%40mail.gmail.com), started 2026-08-31
- Messages read: 6 of 6
- Patch prose read: v4-0002-pageinspect-validate-page-headers-and-BRIN-GIN-an.patch, v4-0001-pageinspect-validate-line-pointers-before-using-t.patch, 0001-pageinspect-validate-line-pointers-before-using-them.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No surviving objection questions the goal: the thread shows reproducible crashes and out-of-bounds reads, and the cost falls only on pageinspect callers. The remaining points are fixable design questions at level 2. standards-compat-1 asks which header checks are needed for memory safety, maintenance-fit-2 asks for a stated scope, and maintenance-fit-1 asks for a shared helper. None of them would stop a committer from reading further.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, the exact fields verify_page_header() checks and which functions call it could not be confirmed. Whether value decoding can still read past a stored value also depends on the code.

## The proposal, restated

**Problem.** pageinspect functions that take a raw page as bytea can read outside the page image, or fail an assertion, when given a corrupt or crafted page. The author shows ASan heap-buffer-overflow reports and backend crashes for GiST, B-tree, heap and BRIN functions. The functions are superuser-only, so the author frames this as robustness for people inspecting damaged pages.

**Proposed design.** 0001 checks each line pointer and B-tree posting lists before use in the GiST, B-tree and hash item functions, and turns an assertion on LP_DEAD pivot tuples into an error. 0002 adds verify_page_header(), using the header checks of PageIsVerified() without the checksum, and checks BRIN line pointers, BRIN and GiST stored values and GIN posting lists. page_header() still shows damaged headers.

**User-visible surface.**
- New ERROR from the item functions on corrupt input, for example "invalid line pointer at offset 1 in GiST page".
- bt_page_items() raises ERRCODE_DATA_CORRUPTED "invalid pivot tuple at offset %u: line pointer marked dead" where it used to assert or return NULL.
- heap_page_items(), brin_page_items(), GIN and hash functions now reject pages whose header fails the new header checks.
- page_header() still accepts a page with a damaged header.

**What the author claims.**
- Without the patch, crafted pages crash the backend or trip ASan in GiST, B-tree, heap and BRIN functions.
- heap_page_items() already guards line pointers, and the patch brings the other access methods to the same level.
- This is a robustness fix and not a security issue, because the functions are superuser-only.
- Normal output for valid pages is unchanged, and each fix comes with regression tests.

**Thread state.** Greg Burd found the LP_DEAD pivot assertion and Yilin Zhang found a BRIN tuple-data overread; v4 folds in both fixes. No one has reviewed the overall approach yet, and CFbot reports v4 needs a rebase.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-08-31; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Whole-page header check takes item listings away from forensic users

fixable · level 2, rubric item 12 · design, backpatchability · `standards-compat-1`

A DBA investigating a damaged heap block runs heap_page_items() on a page where only pd_flags has a flipped bit, or pd_special is wrong. Before the patch the call returned the line pointers and tuples, which are still in bounds. After the patch it raises an error, and the DBA loses the data needed to salvage rows.

Message #2 adds verify_page_header() with the same checks PageIsVerified() makes and applies it to heap_page_items(), brin_page_items(), GIN and hash. Some of those checks, such as pd_flags or pd_special alignment, have no bearing on whether reading the line pointer array is safe. heap_page_items() already guards each line pointer (message #1), so this is a behaviour change for the main users of the function, who inspect corrupt pages on purpose. A committer would likely ask for clamping pd_lower and checking only fields that bound memory access, and would weigh this more heavily if the patch is back-patched, which the thread does not settle.

Thread: [#1 (shihao zhong, 2026-08-31)](https://www.postgresql.org/message-id/CAGRkXqRWDJR7Ma%3D589-cOFaOK2pG-XxUgXzYvje9eMxxSiOm8g%40mail.gmail.com), [#2 (shihao zhong, 2026-09-01)](https://www.postgresql.org/message-id/CAGRkXqQ%2BNvzXW%2B2AjkT1ACR-W%2By-cDoy%2BmzLXriMAV2ovVRA5g%40mail.gmail.com)

**Question:** Which header checks are needed to keep reads inside the page image, and why should a page that fails only the other checks raise an error instead of returning its items?

Counter-check: held. PageIsVerified() also tests pd_flags bits and pd_special alignment, which do not bound reads of the line pointer array. The thread does not say why a page failing only those tests should lose its item listing, and page_header() is not a substitute for it.

### 2. The hardening contract for raw-page input has no stated limit

fixable · level 2, rubric item 6 · maintenance burden ("who maintains this in 10 years"), backpatchability · `maintenance-fit-2`

A user hands brin_page_items() or gist_page_items() a page whose stored values fit inside the tuple but hold a crafted varlena header or opclass datum. The type output function reads past the value and the backend crashes. Each such report then arrives as a bug against a fix that looked complete, and someone has to decide whether to back-patch it.

Message #6 says v4 checks that values fit inside the tuple but not what they contain. Messages #3 and #5 each found a new crash in a version that looked complete, which suggests the cases go further than the patch covers. The proposal does not say where the boundary lies: which functions must never crash on any bytea, and which stay best-effort. Without that boundary, every new pageinspect function and every new on-page format must add checks, and no test fails when one is missing.

Thread: [#3 (Greg Burd, 2026-10-20)](https://www.postgresql.org/message-id/G1mth6ql6rdb8blD-g98kuDstGPcjBeGKlJdI6zcqI1jVS3ZXnBJMduZu_pQ8_QlyU5u1KyGDHft_rTr3nfzRLYSAUBUBbSWM7glRXpRmK0%3D%40protonmail.com), [#5 ("Yilin Zhang", 2026-10-21)](https://www.postgresql.org/message-id/6b27b31e.4e13.1a15d51e4c8.Coremail.jiezhilove%40126.com), [#6 (shihao zhong, 2026-10-22)](https://www.postgresql.org/message-id/CAGRkXqTV5wo%3D1O-2VKy5jR8bfXujvVJg_ix3%3Dp6kWhvt12he4g%40mail.gmail.com)

**Question:** Which pageinspect functions does the patch promise are safe on arbitrary bytea, and how is value decoding through type output functions covered or excluded from that promise?

Counter-check: held. Message #6 says v4 does not check what stored values contain, and the thread does not say which functions are meant to be safe on any bytea. Step-by-step hardening is existing pageinspect practice, so this asks for a scope statement and does not count against the approach.

### 3. Duplicated header checks can drift from PageIsVerified()

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years") · `maintenance-fit-1`

A later commit tightens or changes the header checks in PageIsVerified(), for example a new pd_flags bit or a new rule for all-zero pages. Nothing makes verify_page_header() in pageinspect change with it. pageinspect then rejects valid pages or accepts pages the server treats as corrupt, and someone finds out from a user report.

Message #2 says verify_page_header() repeats the header checks of PageIsVerified() without the checksum. It keeps a second copy in contrib. The author offers a shared helper in bufpage.c, and message #2 got no answer on it. With a shared helper, a change to the rules in one place covers both callers, and a reviewer has no second copy to miss.

Thread: [#2 (shihao zhong, 2026-09-01)](https://www.postgresql.org/message-id/CAGRkXqQ%2BNvzXW%2B2AjkT1ACR-W%2By-cDoy%2BmzLXriMAV2ovVRA5g%40mail.gmail.com)

**Question:** Would a shared header-check helper in bufpage.c, used by both PageIsVerified() and pageinspect, replace verify_page_header() without changing what either caller accepts?

Counter-check: held. Message #2 confirms verify_page_header() repeats the PageIsVerified() header checks, and the offer of a shared helper is still unanswered. The drift risk is small because these rules change rarely, but nothing ties the two copies together, so it stays a fixable design question.

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
| security / privileges | **yes** | The patch hardens superuser-only functions against crafted bytea input; it adds no function or privilege. |
| backpatchability | **yes** | The entry is tagged Bugfix, and the new errors on damaged pages would change behaviour in supported branches if back-patched. |
| maintenance burden ("who maintains this in 10 years") | **yes** | verify_page_header() repeats the header checks of PageIsVerified(), and each new on-page format needs its own checks. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Which header checks in verify_page_header() are needed to keep reads inside the page image, and why should a page failing only the others raise an error?
2. Which pageinspect functions does the series promise are safe on any bytea, and is value decoding through type output functions inside or outside that promise?
3. Is the series meant for back-patching, and if so, how do supported branches handle pages that returned rows before and now raise an error?
4. Does a new all-zero page still return the same result from heap_page_items() and the other item functions as before the patch?
5. Do bt_page_stats(), bt_metap() and hash_metapage_info() get the same header checks, or are they safe without them?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows the page layout code in bufpage.c and the index access method page formats that pageinspect decodes.
- **Liability:** The owner signs for the new error behaviour on damaged pages, for any back-patch decision, and for follow-up crash reports on cases the checks do not cover.
- **Author follow-up:** The author folded two reviewer-found fixes into v4 and offered a shared helper, but v4 has needed a rebase since 2026-11-10.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** medium
- **Why:** The series fixes reproducible crashes in a tool used by a small group of superusers who inspect damaged pages, and it stays inside contrib/pageinspect. The two patches are about 36 KiB across six access method files, and each check needs someone who knows that page format.

## Appendix

### Refuted in the counter-check

- `already-exists-1` Second copy of the PageIsVerified() header checks in pageinspect. Refuted (not specific): Duplicate of maintenance-fit-1. Both describe the same drift between verify_page_header() and PageIsVerified() and ask for the same shared helper.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. [P6] and [P7] deal with checksum verification of page headers in base backups, which is a different feature, and neither row records a reason that could apply here. No thread message points to an earlier attempt at hardening pageinspect against corrupt pages.
- standards-compat: 1 objection(s). Within the standards and compatibility angle, no new SQL syntax, GUC, catalog object or pg_dump state is added. The only contract concern is the stricter rejection of damaged pages.
- replication-ondisk: 0 objection(s). The patch changes only how contrib/pageinspect reads a bytea page image in backend memory. It adds no on-disk format, WAL record, catalog column, wire message or exported API. The touched files are all inside contrib/pageinspect, and pageinspect.h is private to that module, so pg_upgrade, standbys, subscribers and extensions are not affected. Back-patching would change error behavior only, with no ABI break. Precedents [P6] and [P7] concern checksum checks in base backups, and neither records a reason that applies here.
- security: 0 objection(s). The patch adds no new function, privilege or default, and these pageinspect functions stay superuser-only. It narrows what a crafted bytea can do, so a committer would see no new exposure from the security angle; back-patch handling is for other panel members.
- maintenance-fit: 2 objection(s). No hot-path cost: the checks run only inside superuser-only pageinspect functions. The listed precedents record no reason, so none is cited.
- already-exists: 1 objection(s). No existing feature makes the pageinspect raw-page functions safe on corrupt input. amcheck verifies relations and does not check user-supplied bytea images, so the need itself is not already met.
- demand: 0 objection(s). No demand objection. The patch fixes backend crashes and out-of-bounds reads in a tool whose main use is inspecting damaged pages, and its cost falls only on pageinspect callers. Three people in the thread reproduced and extended the cases.

### Precedents retrieved

- P1: [Allow pageinspect's bt_page_stats function to return a set of rows instead of a single row](https://commitfest.postgresql.org/patch/3713/) (cf, Returned with feedback, 2022-10-12) - judged not relevant; reason summarised from its thread
- P2: [pageinspect: Hash index support](https://commitfest.postgresql.org/patch/758/) (cf, Returned with feedback, 2016-12-20) - judged not relevant
- P3: [Pageinspect - add functions on GIN and GiST indexes from gevel](https://commitfest.postgresql.org/patch/1207/) (cf, Returned with feedback, 2017-11-27) - judged not relevant; reason summarised from its thread
- P4: [Revert the GinMaxItemSize calculation so that we fit 3 tuples per page.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=930fd68455576dada48521a6cef642caddea3c63) (revert, Reverted, 2014-12-30) - judged not relevant
- P5: [Revert "Add single-item cache when looking at topmost XID of a subtrans XID"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=b4529005fd387e863bfa9eb863629b1183c0449c) (revert, Reverted, 2022-05-28) - judged not relevant
- P6: [Fix checksum verification in base backups for zero page headers](https://commitfest.postgresql.org/patch/2719/) (cf, Returned with feedback, 2021-09-13) - used in the review
- P7: [Fix base backup checksum verification for random or zero page headers](https://commitfest.postgresql.org/patch/2308/) (cf, Returned with feedback, 2020-07-30) - used in the review
- P8: [new diagnostic items for the dynamic sql](https://commitfest.postgresql.org/patch/3537/) (cf, Returned with feedback, 2022-12-14) - judged not relevant; reason summarised from its thread
- P9: [brin: avoid errors processing work item following concurrent reindex](https://commitfest.postgresql.org/patch/2838/) (cf, Returned with feedback, 2021-12-02) - judged not relevant; reason summarised from its thread
- P10: [Fix assertion failure in SnapBuildInitialSnapshot()](https://commitfest.postgresql.org/patch/4149/) (cf, Returned with feedback, 2024-02-01) - judged not relevant; reason summarised from its thread
- P11: [allow building trusted languages without the untrusted versions](https://commitfest.postgresql.org/patch/3647/) (cf, Returned with feedback, 2022-08-01) - judged not relevant; reason summarised from its thread
- P12: [Revert pg_relation_check_pages()](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e152506adef4bc503ea7b8ebb4fedc0b8eebda81) (revert, Reverted, 2020-11-04) - judged not relevant
- P13: [Harden against premature discarding of WAL segments in TAP tests](https://commitfest.postgresql.org/patch/3333/) (cf, Withdrawn, 2021-11-03) - judged not relevant; reason summarised from its thread
- P14: [Skip all-visible pages during second HeapScan of CIC](https://commitfest.postgresql.org/patch/1057/) (cf, Returned with feedback, 2017-03-24) - judged not relevant; reason summarised from its thread
- P15: [autovectorize page checksum code included elsewhere](https://commitfest.postgresql.org/patch/4649/) (cf, Withdrawn, 2024-04-12) - judged not relevant
- P16: [Revert "Avoid the creation of the free space map for small heap relations".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=7db0cde6b58eef2ba0c70437324cbc7622230320) (revert, Reverted, 2019-05-07) - judged not relevant
- P17: [Show subtransaction overflow status](https://commitfest.postgresql.org/patch/3444/) (cf, Rejected, 2022-03-22) - judged not relevant
- P18: [SLRU data page checksums](https://commitfest.postgresql.org/patch/1682/) (cf, Returned with feedback, 2018-10-02) - judged not relevant
- P19: [Removing unused trailing linepointers from heap pages](https://commitfest.postgresql.org/patch/3035/) (cf, Returned with feedback, 2021-12-02) - judged not relevant
- P20: [Avoid possible null pointer dereference](https://commitfest.postgresql.org/patch/5506/) (cf, Withdrawn, 2025-02-06) - judged not relevant
- P21: [Assert that locks are properly held in PageGetLSN](https://commitfest.postgresql.org/patch/1279/) (cf, Returned with feedback, 2017-10-02) - judged not relevant
- P22: [Use ReadRecentBuffer() for btree root page](https://commitfest.postgresql.org/patch/4419/) (cf, Returned with feedback, 2024-10-07) - judged not relevant
- P23: [In bitmap scan explain node show how many relevant heap pages were not examined](https://commitfest.postgresql.org/patch/2443/) (cf, Returned with feedback, 2020-03-31) - judged not relevant
- P24: [Nonreplayable XLog records by means of overflows and >MaxAllocSize lengths](https://commitfest.postgresql.org/patch/3590/) (cf, Returned with feedback, 2022-12-02) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 13; wall time: 110 s; cost reported by the CLI: $1.19
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 66 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:30:56.329Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

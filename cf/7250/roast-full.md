# Design pre-review: glist: _Generic wrapper for selective dlist/dclist usage

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7250](https://commitfest.postgresql.org/patch/7250/) (Needs review); authors: Matthias van de Meent (mmeent)
- Thread: [glist: _Generic wrapper for selective dlist/dclist usage](https://www.postgresql.org/message-id/flat/CAEze2WjUs6P1QCUHij9WOnkS7CX%2BtvVqRiFiGdXoyevh417h4A%40mail.gmail.com), started 2026-09-02
- Messages read: 8 of 8
- Patch prose read: v1-0001-ilist.h-Introduce-glist-convenience-macros.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The idea is clear, and one reviewer found the macros correct, but the proposal fails at desirability as a standalone patch. It touches only ilist.h and adds no caller, so no build compiles the _Generic dispatch (maintenance-fit-1). The thread names one prospective user, in an uncommitted series, and no second one (demand-1). The proposal gives no number for the memory cost of using dclist in all builds, so the existing type may be enough (already-exists-1).

First failing level, from the base up: **Level 1b: desirability**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: failed
- Level 2: architecture and design: not evaluated
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, the set of wrapped functions and the behaviour of the _Generic dispatch on all supported compilers and in C++ header checks could not be judged. One reviewer found the macros correct in message #4.

## The proposal, restated

**Problem.** Developers who want a list element count only in some builds, such as MEMORY_CONTEXT_CHECKING, must choose between dclist everywhere, which costs memory in production builds, and #ifdef wrappers around each dlist/dclist call. The author wants this first for list validation in the proposed Proxy memory contexts.

**Proposed design.** The patch adds glist_* macros to src/include/lib/ilist.h. They use C11 _Generic to send each call to the dlist_* or dclist_* function based on the list type. A field can then be a dlist in one build and a dclist in another without #ifdef at each call site.

**User-visible surface.**
- C API: new glist_* macros in src/include/lib/ilist.h that wrap several dlist_* and dclist_* functions.

**What the author claims.**
- Calling code needs no #ifdef around the most common dlist/dclist APIs and gains little visual overhead.
- Using dclist in all builds wastes memory when the count is needed only in checking builds.
- Hand-written #ifdef wrappers would mean a lot of effort, repeated for each user.
- glist is meant for fields whose type differs by build, not as a default replacement for dlist or dclist.
- No real API in the headers uses _Generic yet, so a separate thread is the place to gather feedback on that.

**Thread state.** Tristan Partin found the macros correct, asked for a usage comment in ilist.h and suggested the name dglist, which the author rejected to leave room for slist. David Rowley and Japin Li object that the patch adds dead code with no caller and should be part of the Proxy memory context series.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-09-02; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. New header API with no caller cannot be checked or maintained

fixable · level 1b, rubric item 9 · maintenance burden ("who maintains this in 10 years"), ABI / extension API · `maintenance-fit-1`

A later patch changes a dclist_* signature or adds a dlist_* variant and updates every caller. The glist_* macros in ilist.h expand only where they are used, so with no user the build stays green. The first developer who tries glist finds it broken or out of step, possibly years later.

The patch touches only src/include/lib/ilist.h and adds no user. Japin Li could not name an existing place to use it (message #8), and David Rowley asked for a caller in the same series (message #5). The author answers that a separate thread gathers feedback on _Generic as an API style (message #7). That is a reason to discuss the idea, but a committer would likely not take ownership of macros that nothing compiles.

Thread: [#5 (David Rowley, 2026-09-02)](https://www.postgresql.org/message-id/CAApHDvo437%2BLn-3C7%2BEDNW5k3dMpSKb-z6PCr_0quPpf%3DYqboA%40mail.gmail.com), [#7 (Matthias van de Meent, 2026-09-03)](https://www.postgresql.org/message-id/CAEze2WhB_cofsSuT113CFqwJBp0gpDQckCdRzbFcsuiqX138Rg%40mail.gmail.com), [#8 (Japin Li, 2026-09-03)](https://www.postgresql.org/message-id/SY7PR01MB10921344A12E4017BB4D76B28B6B62%40SY7PR01MB10921.ausprd01.prod.outlook.com)

**Question:** Will the Proxy memory context series carry this patch as its first step, with a caller that compiles the glist_* macros in normal and MEMORY_CONTEXT_CHECKING builds?

Counter-check: held. A macro body is checked only where it is expanded, and the patch touches only ilist.h. With no caller, no build compiles the _Generic dispatch, and message #7 does not offer a caller.

### 2. New API with no caller and one prospective user

fixable · level 1b, rubric item 3 · demand, doesn't fit what Postgres is about (extension / external tool / already solved), maintenance burden ("who maintains this in 10 years") · `demand-1`

A committer pushes glist_* into ilist.h as a standalone patch. The Proxy memory context series then changes direction, or drops its debug-only count after review. ilist.h is left with _Generic macros that nothing calls, and every later change to dlist or dclist must keep them in sync, with no caller to show breakage.

The only use named is the author's own Proxy memory context series (message #1). Message #3 says glist is not meant for code that is always dlist or always dclist, and Japin Li could name no existing place where it fits (message #8). David Rowley questions whether that one use is worth having (message #5); message #7 answers him, but nobody else has agreed yet. The author wants a separate thread to collect feedback on _Generic in headers, and so far that has produced no second user.

Thread: [#1 (Matthias van de Meent, 2026-09-02)](https://www.postgresql.org/message-id/CAEze2WjUs6P1QCUHij9WOnkS7CX%2BtvVqRiFiGdXoyevh417h4A%40mail.gmail.com), [#3 (Matthias van de Meent, 2026-09-02)](https://www.postgresql.org/message-id/CAEze2Wj7GJjfPm8HLtTnaS2EWFCj8fOcRXv8uWebDrC9ZTc5YQ%40mail.gmail.com), [#5 (David Rowley, 2026-09-02)](https://www.postgresql.org/message-id/CAApHDvo437%2BLn-3C7%2BEDNW5k3dMpSKb-z6PCr_0quPpf%3DYqboA%40mail.gmail.com), [#7 (Matthias van de Meent, 2026-09-03)](https://www.postgresql.org/message-id/CAEze2WhB_cofsSuT113CFqwJBp0gpDQckCdRzbFcsuiqX138Rg%40mail.gmail.com), [#8 (Japin Li, 2026-09-03)](https://www.postgresql.org/message-id/SY7PR01MB10921344A12E4017BB4D76B28B6B62%40SY7PR01MB10921.ausprd01.prod.outlook.com)

**Question:** Which callers besides the Proxy memory context series need a list that is dlist in one build and dclist in another, and could glist ship as a patch inside that series?

Counter-check: held. The thread names one prospective user, in an uncommitted series, and message #8 found no existing place for glist. Message #7 explains the separate thread but adds no second user and does not commit to shipping a caller.

### 3. Plain dclist or a local wrapper already covers the one known user

fixable · level 1b, rubric item 5 · doesn't fit what Postgres is about (extension / external tool / already solved), maintenance burden ("who maintains this in 10 years"), ABI / extension API · `already-exists-1`

The Proxy memory context code can declare its list as a dclist in all builds, or keep a small #ifdef MEMORY_CONTEXT_CHECKING wrapper next to its only use. If glist goes into ilist.h, every later change to the dlist or dclist API also needs a matching glist macro, for a single caller in a different patch series.

dclist already keeps the element count, and using it in every build costs one count field per list head plus an increment per insert or delete. The author calls this a waste of memory (message #1) but does not say how many such list heads a proxy context has. The point that hand-written wrappers would be duplicated for each user (message #1) does not hold yet, because the thread names no second user and Japin Li found none (message #8).

Thread: [#1 (Matthias van de Meent, 2026-09-02)](https://www.postgresql.org/message-id/CAEze2WjUs6P1QCUHij9WOnkS7CX%2BtvVqRiFiGdXoyevh417h4A%40mail.gmail.com), [#3 (Matthias van de Meent, 2026-09-02)](https://www.postgresql.org/message-id/CAEze2Wj7GJjfPm8HLtTnaS2EWFCj8fOcRXv8uWebDrC9ZTc5YQ%40mail.gmail.com), [#5 (David Rowley, 2026-09-02)](https://www.postgresql.org/message-id/CAApHDvo437%2BLn-3C7%2BEDNW5k3dMpSKb-z6PCr_0quPpf%3DYqboA%40mail.gmail.com), [#8 (Japin Li, 2026-09-03)](https://www.postgresql.org/message-id/SY7PR01MB10921344A12E4017BB4D76B28B6B62%40SY7PR01MB10921.ausprd01.prod.outlook.com)

**Question:** How much memory does an unconditional dclist cost the Proxy memory contexts in production builds, and which second caller would need the same build-dependent list type?

Counter-check: held. dclist in all builds, or a local #ifdef wrapper, can do the job for the one named user. Message #1 calls the memory cost a waste but gives no number, and the thread names no second user.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | The patch adds glist_* macros to src/include/lib/ilist.h, a header that backend and extension code includes. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | The glist_* macros mirror dlist_* and dclist_* functions and have no caller that would show breakage. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | A dclist in all builds, or a local #ifdef wrapper, may already cover the one named user. |

## 5. Questions the author must answer before anyone reviews code

1. Will the Proxy memory context series carry this patch as its first step, with a caller that compiles the glist_* macros in normal and MEMORY_CONTEXT_CHECKING builds?
2. How much memory does a dclist in all builds cost the Proxy memory contexts in production, counted per list head and per context?
3. Which caller besides the Proxy memory context series needs a field that is a dlist in one build and a dclist in another?
4. What does the count kept only in checking builds verify, given David Rowley's point that it checks only its own tracking?
5. Which dlist_* and dclist_* functions get glist_* wrappers, and how do the macros behave with compilers or build modes that lack _Generic?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. The material does not identify any thread participant as a committer. The owner would be someone who knows ilist.h and the memory context code, and who accepts _Generic as an API style in headers.
- **Liability:** The owner signs for the first _Generic-based API in the headers and for keeping the glist_* macros in step with dlist and dclist. With no caller, nothing reports breakage.
- **Author follow-up:** The author answered reviewers in messages #3, #6 and #7 and plans to use the macros in the Proxy memory context series.

**Not now / show demand: yes.** This is an outcome, not a rejection. Evidence that would change it: A patch in the same series that calls the glist_* macros in normal and MEMORY_CONTEXT_CHECKING builds, plus either a number for the memory that a dclist in all builds would cost or a second caller with the same need.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** bundle
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch is about 5 KiB in one header and serves one prospective caller, the Proxy memory context series. Two reviewers asked for it to be part of that series, where a caller would compile and justify it.

## Appendix

### Refuted in the counter-check

- `prior-attempts-1` Standalone macros with no caller, outside the series that would use them. Refuted (not specific): Duplicate of demand-1, which gives the same failure scenario without the precedent. P8 is about removing an unused macro and lapsed for lack of activity, so it must not be used as support here.
- `maintenance-fit-2` A partial mirror API must follow every dlist and dclist change. Refuted (no failure scenario): A missing glist wrapper is a compile error at the first call site, so it fails loudly and the developer adds it then. The commit message limits glist to the most common functions, and maintenance-fit-1 already covers undetected drift.

### Review angles

- prior-attempts: 1 objection(s). No precedent proposes _Generic wrappers over dlist and dclist. [P1] shares only the topic of helper macros, and its withdrawal reason is not shown.
- standards-compat: 0 objection(s). The proposal adds only internal C macros to ilist.h and changes no SQL syntax, GUC, catalog, pg_dump output, psql behaviour or client protocol, so nothing user-facing can break on upgrade. Two compatibility points could not be checked without the code. First, the macros may not compile as C++, but they fail only where expanded and cpluspluscheck would catch a header that uses them. Second, the material does not say which dlist/dclist functions the macros cover. The missing caller belongs to the desirability angle.
- replication-ondisk: 0 objection(s). The patch adds only C preprocessor macros to src/include/lib/ilist.h. It changes no on-disk format, WAL record, catalog, wire protocol message or exported struct, so it has no persistent or cross-node state. A struct whose list field differs by build would have a different layout per build, but MEMORY_CONTEXT_CHECKING builds already change struct layouts, and the patch itself has no such caller.
- security: 0 objection(s). The proposal adds compile-time C macros in ilist.h that pick between dlist_* and dclist_* functions. It adds no SQL surface, privilege, catalog entry, view, or data path that a database user can reach. The count is only kept in checking builds, so it offers no new lever for denial of service or information exposure.
- maintenance-fit: 2 objection(s). Two objections from the maintenance angle. Correctness of the _Generic dispatch was not judged, and Tristan Partin found the macros correct (message #4).
- already-exists: 1 objection(s). One objection only: the need is met by the existing dclist type or a wrapper local to the caller, and the shared macros add value only once more than one caller exists.
- demand: 1 objection(s). One objection: no demand is shown beyond the author's own pending series, so this is a not-now case. It is not a rejection of the idea.

### Precedents retrieved

- P1: [Safer hash table initialization macro](https://commitfest.postgresql.org/patch/6276/) (cf, Withdrawn, 2026-03-09) - used in the review; reason summarised from its thread
- P2: [Change simple_heap_insert() to a macro](https://commitfest.postgresql.org/patch/1826/) (cf, Rejected, 2018-11-02) - judged not relevant; reason summarised from its thread
- P3: [Generic WAL compression](https://commitfest.postgresql.org/patch/1363/) (cf, Returned with feedback, 2018-03-21) - judged not relevant; reason summarised from its thread
- P4: [num_sa_scans in genericcostestimate](https://commitfest.postgresql.org/patch/3871/) (cf, Returned with feedback, 2022-10-12) - judged not relevant; reason summarised from its thread
- P5: [Generic type subscription](https://commitfest.postgresql.org/patch/793/) (cf, Returned with feedback, 2017-01-31) - judged not relevant; reason summarised from its thread
- P6: [Add information about generic/custom plan to pg_stat_statements](https://commitfest.postgresql.org/patch/2713/) (cf, Withdrawn, 2021-04-05) - judged not relevant; reason summarised from its thread
- P7: [Rename files used for index AMs to less generic names](https://commitfest.postgresql.org/patch/2385/) (cf, Returned with feedback, 2020-01-10) - judged not relevant; reason summarised from its thread
- P8: [Remove dead macro exec_subplan_get_plan](https://commitfest.postgresql.org/patch/3894/) (cf, Rejected, 2023-04-04) - used in the review; reason summarised from its thread
- P9: [Revert addition of PGDLLEXPORT in PG_FUNCTION_INFO_V1 macro.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=4f52fd3c6d28a4380a5afc51ae6f774c91837a38) (revert, Reverted, 2016-10-12) - judged not relevant
- P10: [Get rid of the pre-C11 _Alignof define in zic.c](https://commitfest.postgresql.org/patch/6453/) (cf, Withdrawn, 2026-02-13) - judged not relevant
- P11: [Fix some inconsistencies with open-coded visibilitymap_set() callers](https://commitfest.postgresql.org/patch/5861/) (cf, Withdrawn, 2025-07-14) - judged not relevant
- P12: [Revert "Show index search count in EXPLAIN ANALYZE."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=d00107cd63e780753aa25563fa37603369997d0c) (revert, Reverted, 2025-03-05) - judged not relevant
- P13: [Shared Memory Context](https://commitfest.postgresql.org/patch/2166/) (cf, Returned with feedback, 2019-09-30) - judged not relevant
- P14: [Rearrange execution of PARAM_EXTERN Params for plpgsql's benefit.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6719b238e8f0ea83c39d2b1728c7670cdaa34e06) (revert, Partially reverted, 2017-12-21) - judged not relevant
- P15: [pgbench - option to build using ppoll() for larger connection counts](https://commitfest.postgresql.org/patch/1307/) (cf, Returned with feedback, 2017-11-28) - judged not relevant
- P16: [Separate memory contexts for relcache and catcache](https://commitfest.postgresql.org/patch/4554/) (cf, Returned with feedback, 2024-12-11) - judged not relevant
- P17: [Shared Memory Context](https://commitfest.postgresql.org/patch/2325/) (cf, Returned with feedback, 2020-02-01) - judged not relevant
- P18: [Revert "Rename routines for write/read of pgstats file"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=381b3cfe2be02aa114ef7a28c4ec24feaf377c63) (revert, Reverted, 2026-07-14) - judged not relevant
- P19: [Atomic commit support for foreign data wrappers](https://commitfest.postgresql.org/patch/180/) (cf, Returned with feedback, 2016-01-31) - judged not relevant
- P20: [Revert function to get memory context stats for processes](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=fb844b9f06568aa29303122ac0bb7b4c3cce2542) (revert, Reverted, 2025-05-23) - judged not relevant
- P21: [Extending the range operators so they can cope with range <op> element and element <op> range in addition to the existing range <op> range](https://commitfest.postgresql.org/patch/2747/) (cf, Returned with feedback, 2021-04-08) - judged not relevant
- P22: [Revert "Temporarily install debugging in partition_prune test"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ec1a405adc24bc20523eef83ccc85232ec23eaa9) (revert, Reverted, 2024-05-14) - judged not relevant
- P23: [Revert "Change SHA2 implementation based on OpenSSL to use EVP digest routines"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=fe0a1dc52c7332a65b44db8e8408a5fd1d8fc8fb) (revert, Reverted, 2020-09-29) - judged not relevant
- P24: [Temporary file access API](https://commitfest.postgresql.org/patch/3722/) (cf, Returned with feedback, 2022-10-12) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 17; wall time: 139 s; cost reported by the CLI: $1.38
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 2 from cache, 6 summarised now, 0 failed; quotes: 6 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 69 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:31:30.372Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

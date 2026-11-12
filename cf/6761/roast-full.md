# Design pre-review: Remove invalid SS2/SS3 handling from EUC-KR routines

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6761](https://commitfest.postgresql.org/patch/6761/) (Needs review); authors: 성준 장 (sungjun)
- Thread: [Remove invalid SS2/SS3 handling from EUC-KR routines](https://www.postgresql.org/message-id/flat/CAE%2BcgNgTWvCT2%2BHZYRzQA8-wSQrj-FjPQQNffn%3D_3DOpz0pKgA%40mail.gmail.com), started 2026-05-12
- Messages read: 10 of 10
- Patch prose read: v4-0002-Make-EUC-KR-encoding-routines-self-contained.patch, v4-0001-Add-tests-for-EUC_KR-encoding-properties.patch, v1-0001-Make-EUC-KR-encoding-routines-self-contained.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium)

No objection survived the critic. The only one raised, demand-1, was refuted: the new width estimates match the true 2-byte maximum of EUC_KR, and the effects reach only EUC_KR databases on master. The patch aligns mblen, dsplen and maxmblen with KS X 2901 and with pg_euckr_verifychar(). Confidence is medium because no one has replied to v4 and two points could not be checked without the code.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, two points could not be checked: whether pg_upgrade keeps existing TOAST tables for EUC_KR tables under the new size rule, and whether each reader of mblen and maxmblen is safe with unverified 0x8F bytes.

## The proposal, restated

**Problem.** EUC-KR, per KS X 2901, has only ASCII and KS X 1001, so SS2 (0x8E) and SS3 (0x8F) are never valid lead bytes. Postgres still routes EUC-KR mblen, dsplen and mb2wchar through the shared pg_euc_* helpers, which handle SS2/SS3, and the docs list 1-3 bytes per character. The verifier already rejects these bytes, so the code and the docs disagree with it.

**Proposed design.** The patch replaces pg_euckr_mblen(), pg_euckr_dsplen() and pg_euckr2wchar_with_len() with EUC-KR-only versions that know 1-byte and 2-byte sequences, like UHC. It lowers maxmblen for EUC_KR from 3 to 2 and fixes the charset table to "1-2". v4 is split: 0001 adds tests that record current behavior, 0002 makes the change.

**User-visible surface.**
- pg_encoding_max_length('EUC_KR') returns 2 instead of 3.
- In EUC_KR databases, the planner's width estimates for char(n) and varchar(n) shrink, for example from 34 to 24 for char(10).
- A new table whose columns all have a bounded width may no longer get a TOAST table, for example one varchar(700) column.
- Invalid-byte-sequence errors starting with 0x8F show two bytes instead of three.
- The Bytes/Char entry for EUC_KR in the charset documentation table changes from 1-3 to 1-2.

**What the author claims.**
- KS X 2901 leaves code sets 2 and 3 undefined, so no valid EUC-KR text contains SS2 or SS3.
- Valid EUC-KR data is handled exactly as before, because pg_euckr_verifychar() never admits 0x8E or 0x8F.
- Only 0x8F changes length; pg_euc_mblen() already returned 2 for 0x8E.
- lpad('x', 400000000) fails on master and succeeds with v4 in EUC_KR databases.
- The full parallel_schedule in an EUC_KR database shows only two changed lines, and the change is for master only.

**Thread state.** Henson Choi reviewed and marked it Ready for Committer, but John Naylor set it back to Waiting on Author for missing tests and showed plan width changes. v4 adds the tests and documents these effects, and no one has replied to v4 yet.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-05-12; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

No objection held up strongly enough in the counter-check to list here.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | New tables with bounded-width columns may no longer get a TOAST table, and unverified 0x8F text on disk splits differently. |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | pg_wchar_table[PG_EUC_KR].maxmblen in src/common/wchar.c drops from 3 to 2, which extensions and frontend code can read. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | **yes** | Width estimates for char(n) and varchar(n) shrink in EUC_KR databases, for example from 34 to 24 for char(10). |
| pg_dump / restore round-trip | **yes** | A restored table in an EUC_KR database may get no TOAST table where the old cluster had one. |
| psql UX conventions | no |  |
| security / privileges | **yes** | The patch changes character length and buffer sizing rules in wchar.c, an area with a recent CVE fix for EUC_CN. |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | Three EUC-KR-specific routines replace calls to the shared pg_euc_* helpers, in the same shape as UHC. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Does pg_upgrade in binary-upgrade mode still create the TOAST table for an EUC_KR table that had one in the old cluster but would not get one under maxmblen = 2?
2. How do the new routines split unverified text containing 0x8F that is already on disk, such as shared catalog names written from a database with another encoding?
3. Does any reader of mblen or maxmblen for EUC_KR still assume a 3-byte maximum after the change?
4. Were the checks P1 to P5 in Henson Choi's review list completed against v4?
5. Is src/test/locale/ affected by the change, as Michael Paquier asked in message #3?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. John Naylor reviewed the patch, showed the plan width changes and set the status; Michael Paquier asked for tests. The owner needs to know the multibyte encoding code in wchar.c.
- **Liability:** The owner signs for EUC_KR length handling in wchar.c, for the plan and TOAST-table changes in EUC_KR databases after upgrade to version 20, and for any reader of unverified 0x8F data that behaves differently.
- **Author follow-up:** The author produced v4 with the requested tests and a commit message that lists the user-visible effects, about a month after the status was set back to Waiting on Author.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The change affects only EUC_KR databases, adds no new mechanism, and the 0002 patch is about 6.5 KB in wchar.c plus docs and tests. v4 answers the reviewers' requests and waits for a reply.

## Appendix

### Refuted in the counter-check

- `demand-1` No reported user problem to weigh against plan and TOAST changes. Refuted (no failure scenario): The new width estimates match the true 2-byte maximum of EUC_KR, so the scenario of a slower query is speculative. The effects reach only EUC_KR databases, are listed in the v4 commit message, and are limited to master, where planner estimates change in every major release.

### Review angles

- prior-attempts: 0 objection(s). The precedent search found no earlier proposal with the same or a similar idea. Message #1 points to an earlier discussion and message #8 to related Korean encoding fixes (CF 6913, BUG #19354), but the thread does not report a rejection of either, so no prior reason applies here.
- standards-compat: 0 objection(s). Nothing from the standards and compatibility angle supports rejection. The patch brings mblen, dsplen and maxmblen in line with KS X 2901 and with the existing pg_euckr_verifychar(), which already rejects 0x8E and 0x8F, so no valid EUC_KR text changes meaning. The visible effects are smaller width estimates in plans, fewer TOAST tables for new tables, pg_encoding_max_length('EUC_KR') returning 2, and shorter 0x8F error messages. Message #10 and the 0002 commit message list all of these, and the change targets master only. Without the code I could not check one case: whether pg_upgrade --binary-upgrade still creates a TOAST table for a table that had one in the old cluster but would not get one under the new size rule. A committer would likely want that answered before commit, but the brief gives no concrete failure there.
- replication-ondisk: 0 objection(s). The patch changes no on-disk format, WAL record, catalog, wire message or ABI, and targets master only, so no mixed-version standby or backpatch exists. Valid EUC_KR data is verified on input and splits the same way. Only unverified 0x8F text, mostly shared-catalog names, splits differently, and nothing persisted is derived from that split. The TOAST-table change could not be checked without the code: whether pg_upgrade still carries over existing TOAST tables for EUC_KR tables that would no longer get one.
- security: 0 objection(s). No privilege, exposure or denial-of-service issue found: no new object, privilege or default, and after the patch no EUC-KR routine treats a character as longer than 2 bytes, so maxmblen and mblen agree. The effect on unverified 0x8F bytes, such as shared catalog text, and on buffer sizing by maxmblen could not be checked without the code.
- maintenance-fit: 0 objection(s). From the maintenance and fit angle there is no objection. The patch makes EUC-KR's three routines the same shape as UHC's, adds no new mechanism, GUC or API, and costs nothing on a common path. The plan-width and TOAST-table changes fall only on EUC_KR databases, are listed in the commit message, and are for master only. John Naylor, who knows the encoding code, is engaged (message #9), and v4 answers his request for tests. Henson Choi's EUC-CN note (message #2) is a separate follow-up and does not bind the project to anything. Without the code, two things could not be checked: whether each mblen and maxmblen reader is covered, and how unverified 0x8F data already on disk behaves.
- already-exists: 0 objection(s). Postgres has no existing feature, option or extension that makes the EUC_KR mblen, dsplen and mb2wchar routines match the verifier, or that reports maxmblen 2 for EUC_KR. The patch changes core encoding tables that no extension or hook can reach. It adds no second mechanism and reuses the existing UHC shape.
- demand: 1 objection(s). This is a correctness cleanup, and its costs fall only on EUC_KR databases. The single objection asks for the user-facing benefit to be stated. It is not a reason to reject the patch.

### Precedents retrieved

- P1: [Encoding protection for pgcrypto](https://commitfest.postgresql.org/patch/4809/) (cf, Returned with feedback, 2024-10-07) - judged not relevant; reason summarised from its thread
- P2: [Clarify the comments about varlena header encoding](https://commitfest.postgresql.org/patch/3820/) (cf, Returned with feedback, 2022-10-12) - judged not relevant; reason summarised from its thread
- P3: [Add uri percent-encoding for binary data](https://commitfest.postgresql.org/patch/2299/) (cf, Returned with feedback, 2020-07-29) - judged not relevant; reason summarised from its thread
- P4: [Nonreplayable XLog records by means of overflows and >MaxAllocSize lengths](https://commitfest.postgresql.org/patch/3590/) (cf, Returned with feedback, 2022-12-02) - judged not relevant; reason summarised from its thread
- P5: [Make octet_length_client() the same as octet_length()?](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Multi-Language_Support) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P6: [Add octet_length_server() and octet_length_client()](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Multi-Language_Support) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P7: [Precompute string lengths in PerformRadiusTransaction](https://commitfest.postgresql.org/patch/6377/) (cf, Withdrawn, 2026-03-25) - judged not relevant; reason summarised from its thread
- P8: [Defend against encoding ID version skew](https://commitfest.postgresql.org/patch/4601/) (cf, Withdrawn, 2023-10-05) - judged not relevant; reason summarised from its thread
- P9: [hstore: add hstore_length(hstore) function](https://commitfest.postgresql.org/patch/643/) (cf, Returned with feedback, 2016-09-28) - judged not relevant; reason summarised from its thread
- P10: [Revert recent COPY manual change about encoding; add mention of "server".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=edf344a5e3bde87d8d58ca239842e0301039a9b9) (revert, Reverted, 2010-02-23) - judged not relevant
- P11: [Add test module for verifying backtrace functionality](https://commitfest.postgresql.org/patch/4823/) (cf, Withdrawn, 2024-08-24) - judged not relevant; reason summarised from its thread
- P12: [Verify true root on replicas with amcheck](https://commitfest.postgresql.org/patch/2418/) (cf, Returned with feedback, 2020-04-08) - judged not relevant
- P13: [Parallel leader process info in EXPLAIN output](https://commitfest.postgresql.org/patch/2315/) (cf, Returned with feedback, 2020-03-17) - judged not relevant
- P14: [Incorrect handling of OOM in WAL replay leading to data loss](https://commitfest.postgresql.org/patch/4590/) (cf, Withdrawn, 2023-10-24) - judged not relevant
- P15: [bytea PL/Perl transform](https://commitfest.postgresql.org/patch/4252/) (cf, Returned with feedback, 2024-10-07) - judged not relevant
- P16: [KNN-spgist](https://commitfest.postgresql.org/patch/976/) (cf, Returned with feedback, 2017-03-24) - judged not relevant
- P17: [Add a max_parallel_workers GUC](https://commitfest.postgresql.org/patch/703/) (cf, Returned with feedback, 2016-10-03) - judged not relevant
- P18: [Refactor bytea_sortsupport()](https://commitfest.postgresql.org/patch/5288/) (cf, Returned with feedback, 2025-02-13) - judged not relevant
- P19: [Maximize page freezing](https://commitfest.postgresql.org/patch/2909/) (cf, Withdrawn, 2021-07-15) - judged not relevant
- P20: [Maximize page freezing](https://commitfest.postgresql.org/patch/3785/) (cf, Withdrawn, 2022-09-26) - judged not relevant
- P21: [Allow pg_verify_checksums to specify a database to scan](https://commitfest.postgresql.org/patch/1782/) (cf, Returned with feedback, 2018-10-02) - judged not relevant
- P22: [Cube extension kNN support](https://commitfest.postgresql.org/patch/138/) (cf, Returned with feedback, 2015-07-07) - judged not relevant
- P23: [LSN Map](https://commitfest.postgresql.org/patch/89/) (cf, Returned with feedback, 2015-07-06) - judged not relevant
- P24: [Direct SSL Connections](https://commitfest.postgresql.org/patch/4135/) (cf, Returned with feedback, 2023-12-02) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 17; wall time: 110 s; cost reported by the CLI: $1.92
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 2 from cache, 6 summarised now, 0 failed; quotes: 4 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 53 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:02:47.749Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

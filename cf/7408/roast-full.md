# Design pre-review: Fix PGTYPESdate_fmt_asc overflow when a year does not fit "yyyy"

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7408](https://commitfest.postgresql.org/patch/7408/) (Needs review); authors: Andrey Rachitskiy (a.rachitskiy)
- Thread: [Fix PGTYPESdate_fmt_asc overflow when a year does not fit "yyyy"](https://www.postgresql.org/message-id/flat/CAB8bMiuv0T175FkbcFi_jGN1kp_0S_cYLyHXFiL7RvyuqqQrjg%40mail.gmail.com), started 2026-08-11
- Messages read: 7 of 7
- Patch prose read: v2-0001-ecpg-date-fmt-asc-year-overflow.patch, 0001-ecpg-date-fmt-asc-year-overflow.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No objection survived at levels 1a or 1b: the overflow is reproducible with ASan and the goal is a bug fix. The one surviving objection, standards-compat-1, is a fixable level 2 point: the patch targets stable branches but does not state the return value or the contents of outbuf on the new failure. A committer who knows ECPG must decide between an error and wider output, and how far back the fix goes.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, the return value and the state of outbuf on failure could not be checked. Whether the length check covers every token, including negative years printed with "%04u", is also unverified.

## The proposal, restated

**Problem.** PGTYPESdate_fmt_asc() writes a year of five or more digits over the four-character "yyyy" token, so a caller who sizes outbuf as strlen(fmtstring)+1 gets a heap buffer overflow. This affects ECPG programs that format dates with year >= 10000.

**Proposed design.** When a replacement is longer than its token, PGTYPESdate_fmt_asc() rejects it and sets errno to PGTYPES_DATE_BAD_DATE. The patch adds a dt_test case that formats 10000-01-01 with "yyyy".

**User-visible surface.**
- PGTYPESdate_fmt_asc() now fails with errno PGTYPES_DATE_BAD_DATE for years that do not fit the "yyyy" token, where before it wrote past the token.

**What the author claims.**
- ASan reports a heap-buffer-overflow on the next strstr() in the token loop.
- Callers usually size outbuf as strlen(fmtstring)+1, as dt_test does.
- The fix belongs in stable branches (CommitFest target version: stable).

**Thread state.** Zsolt Parragi found a wrong line number in the expected stderr, and v2 fixed it. David Rowley asked the author to stop sending reminders, and no one has reviewed the design.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-08-11; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Back-branch change to the PGTYPESdate_fmt_asc() failure contract is not defined

fixable · level 2, rubric item 6 · backpatchability, ABI / extension API · `standards-compat-1`

An ECPG application on a minor release formats far-future dates with a trailing "yyyy" into a large buffer, then prints outbuf without checking the return value. After the upgrade the call fails, and outbuf holds whatever the patch leaves there. The user finds out from wrong report output, not from an error.

The patch targets stable branches and turns a case that had no error into a new error, with errno set to PGTYPES_DATE_BAD_DATE. The commit message does not say what the function returns or what outbuf holds on this failure, and no documentation hunk lists the case. For a library contract changed in a minor release, a committer would likely ask for both to be stated. A committer would also likely ask whether a caller with a large enough buffer ever got a correct five-digit year before the patch.

Thread: [#1 (Andrey Rachitskiy, 2026-08-11)](https://www.postgresql.org/message-id/CAB8bMiuv0T175FkbcFi_jGN1kp_0S_cYLyHXFiL7RvyuqqQrjg%40mail.gmail.com), [#4 (Andrey Rachitskiy, 2026-08-20)](https://www.postgresql.org/message-id/CAB8bMiuOxjEJMVg%3DrWDE92AFDUsOjHCiWjRgg0JvFnEhcPZn5Q%40mail.gmail.com)

**Question:** What do the return value and outbuf look like on the new failure, and did any caller with a large buffer get correct output before the patch?

Counter-check: held. The commit message and thread do not say what PGTYPESdate_fmt_asc() returns or what outbuf holds on the new failure. A caller with a zero-filled larger buffer and a trailing "yyyy" could get a correct five-digit year today, so the back-branch behavior change needs an answer.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | PGTYPESdate_fmt_asc() in libpgtypes gains a new failure case for callers; the signature does not change. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | **yes** | The patch closes a client-side heap buffer overflow in libpgtypes. |
| backpatchability | **yes** | The CommitFest entry targets stable branches, and the fix turns a case with no error into an error. |
| maintenance burden ("who maintains this in 10 years") | no |  |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. What does PGTYPESdate_fmt_asc() return, and what does outbuf hold, when the year does not fit "yyyy"?
2. Did a caller with a buffer larger than strlen(fmtstring)+1 get a correct five-digit year before the patch, and does that caller now get an error?
3. Can any other token, such as a month or day name, produce a replacement longer than the token, and does the length check cover it?
4. Which stable branches should receive the fix?
5. Does the ECPG documentation for PGTYPESdate_fmt_asc() need to list the new failure case?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows ECPG and pgtypeslib, and who can judge a behavior change in a client library on stable branches. No such committer has reviewed the design in the visible thread.
- **Liability:** The owner signs for a new error return in a shipped client library on every stable branch that gets the fix, and for any ECPG application that relied on the old output.
- **Author follow-up:** The author sent v2 after the report in message #3 about the wrong line number in the expected stderr, which suggests the author stays for fixes.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs champion
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch is about 6 KiB, touches one pgtypeslib file plus dt_test, and fixes an out-of-bounds write that only ECPG programs with years of 10000 or more reach. It waits on a committer who knows ECPG to settle the failure contract and the back-patch range.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. The precedent search returned no rows on the same or a similar idea, and no thread message points to an earlier discussion of this overflow or of PGTYPESdate_fmt_asc, so the prior-attempts angle has nothing to raise.
- standards-compat: 1 objection(s). The other compatibility points look sound. The patch adds no syntax, GUC, catalog entry or dump object, and it reuses an existing errno value.
- replication-ondisk: 0 objection(s). The patch changes only the formatting logic of PGTYPESdate_fmt_asc() in the client-side libpgtypes and its regression test. It adds no on-disk, WAL, catalog, wire protocol or replication state, and it changes no function signature or exported struct, so the library ABI and backpatching are unaffected from this angle. The new error return in stable branches is a behaviour change for callers, which other angles cover.
- security: 0 objection(s). The patch closes a client-side heap overflow in libpgtypes and adds no new privilege, default, or exposure, so I found no security objection. Without the code I cannot confirm that the length check covers every token, including negative years printed with %04u.
- maintenance-fit: 0 objection(s). From the maintenance and fit angle there is nothing to object to: a bounded fix to an out-of-bounds write in pgtypeslib adds no API, hook or work for code that does not use it. Two points remain open: no ecpg committer has reviewed it, and the new error in stable branches affects callers that gave a larger outbuf and got "10000" back.
- already-exists: 0 objection(s). This is a bug fix for an out-of-bounds write in an existing ECPG library function. No existing feature, option or extension gives callers a safe result here, so the need is not already met. Choices such as error versus wider output, and the back-patch scope, belong to other angles.
- demand: 0 objection(s). This is a fix for a heap buffer overflow in a shipped libpq-adjacent library, so demand is not the right test. Years of 10000 or more are rare, but an out-of-bounds write that ASan reproduces (message #1) needs fixing whether or not many callers hit it.

### Precedents retrieved

- P1: [Show subtransaction overflow status](https://commitfest.postgresql.org/patch/3444/) (cf, Rejected, 2022-03-22) - judged not relevant; reason summarised from its thread
- P2: [Nonreplayable XLog records by means of overflows and >MaxAllocSize lengths](https://commitfest.postgresql.org/patch/3590/) (cf, Returned with feedback, 2022-12-02) - judged not relevant; reason summarised from its thread
- P3: [ECPG: Declare Statement](https://commitfest.postgresql.org/patch/936/) (cf, Returned with feedback, 2017-11-16) - judged not relevant; reason summarised from its thread
- P4: [Improve overflow checks potentially optimized away by compiler](https://commitfest.postgresql.org/patch/400/) (cf, Returned with feedback, 2015-12-01) - judged not relevant; reason summarised from its thread
- P5: [Revert pgindent changes to ecpg include files that are part of ecpg regession test output, and update pgindent script to avoid them in the future.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2cc6ff45f8813593e7c8c0757cd13a7b63a41768) (revert, Reverted, 2010-02-26) - judged not relevant
- P6: [Add generate_series(date,date,\[step\])](https://commitfest.postgresql.org/patch/492/) (cf, Returned with feedback, 2016-03-17) - judged not relevant; reason summarised from its thread
- P7: [to_date_valid()](https://commitfest.postgresql.org/patch/662/) (cf, Returned with feedback, 2016-09-29) - judged not relevant
- P8: [Correct negative/zero year in to_date/to_timestamp](https://commitfest.postgresql.org/patch/533/) (cf, Rejected, 2016-03-11) - judged not relevant; reason summarised from its thread
- P9: [Revert COPY OUT to follow the pre-8.3 handling of ASCII control characters, namely that \r, \n, \t, \b, \f, \v are dumped as those two-character...](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a63b63ff96293e153d25e8e054a830d70f69938a) (revert, Reverted, 2007-12-03) - judged not relevant
- P10: [Refactor ECPGconnect() and allow IPv6 connection](https://commitfest.postgresql.org/patch/3005/) (cf, Returned with feedback, 2021-10-01) - judged not relevant; reason summarised from its thread
- P11: [Support UTF16/32 host variable in ECPG](https://commitfest.postgresql.org/patch/1337/) (cf, Returned with feedback, 2017-11-28) - judged not relevant; reason summarised from its thread
- P12: [Revert change of ecpglib major version](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=74f2a8aa27cb7bd2dbab3ed58e6b5c239a6b4a31) (revert, Reverted, 2019-09-19) - judged not relevant
- P13: [Revert my bad decision of about a year ago to make PortalDefineQuery responsible for copying the query string into the new Portal.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1591fcbec77f6544a0f665758bae912c68e6fa42) (revert, Reverted, 2008-04-02) - judged not relevant
- P14: [Fix out-of-bounds with function PQescapeInternal](https://commitfest.postgresql.org/patch/4910/) (cf, Rejected, 2024-04-04) - judged not relevant
- P15: [Revert support for improved tracking of nested queries](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f85f6ab051b7cf6950247e5fa6072c4130613555) (revert, Reverted, 2025-06-12) - judged not relevant
- P16: [Add Change Badges to documentation](https://commitfest.postgresql.org/patch/2309/) (cf, Returned with feedback, 2019-11-25) - judged not relevant
- P17: [Revert "Cannot use WL_SOCKET_WRITEABLE without WL_SOCKET_READABLE."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a58db3aa10e62e4228aa409ba006014fa07a8ca2) (revert, Reverted, 2020-12-15) - judged not relevant
- P18: [Revert a few small patches that were intended for version 19.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e1458f2f1b56cfe07b3e8fc99b223ea8c852d7ce) (revert, Reverted, 2025-06-11) - judged not relevant
- P19: [Revert badly-broken patch to dump comments on composite-type columns.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=18661f282bae5599abb68f53bc8b451b2d67bd75) (revert, Reverted, 2004-08-06) - judged not relevant
- P20: [Parse dates that include CE and BCE](https://commitfest.postgresql.org/patch/3682/) (cf, Returned with feedback, 2022-08-02) - judged not relevant
- P21: [TimestampTz->Text->TimestampTz casting fails with DateStyle 'Postgres'](https://commitfest.postgresql.org/patch/5371/) (cf, Returned with feedback, 2025-01-07) - judged not relevant
- P22: [Support empty ranges with bounds information](https://commitfest.postgresql.org/patch/3028/) (cf, Rejected, 2021-03-03) - judged not relevant
- P23: [Revert binary change in copyright year adjustment.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1358801b7a7afed840b780d7c52e20921a483ed1) (revert, Reverted, 2012-01-02) - judged not relevant
- P24: [Memory-bounded HashAgg](https://commitfest.postgresql.org/patch/63/) (cf, Rejected, 2015-01-19) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 13; wall time: 89 s; cost reported by the CLI: $0.77
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 47 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:51:48.849Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

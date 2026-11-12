# Design pre-review: pgcrypto: Fix sha-crypt rounds= values that do not fit in int (BUG #19743)

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7507](https://commitfest.postgresql.org/patch/7507/) (Needs review); authors: Prashant Chinnam (prashantelephant2)
- Thread: [\[PATCH\] pgcrypto: Fix sha-crypt rounds= values that do not fit in int (BUG #19743)](https://www.postgresql.org/message-id/flat/SN7P223MB0600215618AD31B5ACE2109EDD642%40SN7P223MB0600.NAMP223.PROD.OUTLOOK.COM), started 2026-10-23
- Messages read: 13 of 13
- Patch prose read: v1-0001-pgcrypto-Fix-sha-crypt-rounds-values-that-do-not-fit.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium)

No objection survived. The only one raised, standards-compat-1, was refuted: per message #9, strtoint() already saturates on REL_18_STABLE, so the patch changes only NOTICE text and tests there. The change is a fix for a reported bug (BUG #19743) inside existing pgcrypto code, reusing the existing clamp-with-NOTICE path. Confidence is medium because only 3 of 13 messages and the v1 patch prose were read.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The v2 to v5 patch prose and the code were not read, so the final test design, the 10-character NOTICE cap and the dependence on commit 426551c8cf49 could not be checked.

## The proposal, restated

**Problem.** crypt() parses the rounds= option of a sha256crypt or sha512crypt salt with strtoint() and never checks for overflow, so values beyond int wrap modulo 2^32 before clamping. rounds=4294968296 silently hashes with 1000 rounds and no NOTICE. This affects users who pass raw salt strings instead of using gen_salt() (BUG #19743).

**Proposed design.** Out-of-range rounds= values now reach the existing clamp to PX_SHACRYPT_ROUNDS_MIN or PX_SHACRYPT_ROUNDS_MAX with a NOTICE. v2 used strtoi64(), and v4 went back to strtoint() after commit 426551c8cf49 made it saturate on overflow. The clamp NOTICEs echo the token as supplied, capped at 10 characters plus "...".

**User-visible surface.**
- crypt() with $5$ or $6$ salts: overflowing rounds= values now clamp with a NOTICE instead of wrapping silently.
- Clamp NOTICE text now shows the rounds token as supplied, truncated to 10 characters with "..." appended.
- Non-canonical tokens such as rounds=+10 or rounds=010 now appear verbatim in the NOTICE.

**What the author claims.**
- The fix is backpatchable to REL_18_STABLE, the only branch with sha-crypt support.
- Negative rounds values keep the existing clamp behaviour; making them an ERROR belongs in a separate master-only change.
- The pgcrypto regression suite passes 25/25 and core regression passes.
- An unbounded 10000-digit token produced a NOTICE of about 10000 characters before the print cap.

**Thread state.** Takeshi Ideriha shaped the design, and Chao Li reviewed v5 and said it looks good. Chao Li will propose a separate patch to reject an empty rounds= value. Negative-value errors are deferred to another discussion.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-10-23; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

No objection held up strongly enough in the counter-check to list here.

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
| security / privileges | **yes** | An overflowing rounds= value no longer yields a silently weaker 1000-round password hash. |
| backpatchability | **yes** | The author proposes a backpatch to REL_18_STABLE, and the NOTICE text changes for tokens such as rounds=+10. |
| maintenance burden ("who maintains this in 10 years") | no |  |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Is commit 426551c8cf49 present on every branch targeted for backpatch, so that strtoint() saturates there as v4 and v5 assume?
2. Are the positive-overflow tests stable under statement_timeout on slow or heavily loaded buildfarm animals, or can they be written without a timeout?
3. Is the changed NOTICE text for in-range non-canonical tokens such as rounds=+10 and rounds=010 acceptable in a REL_18 minor release?
4. What does the final v5 commit message say about the behaviour change on master versus REL_18_STABLE?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows pgcrypto's crypt() code and the backpatch rules for contrib; neither reviewer is identified as a committer in the visible thread.
- **Liability:** A small parsing and NOTICE change in contrib/pgcrypto/crypt-sha.c, plus regression tests that depend on statement_timeout and may need attention on slow buildfarm animals; a backpatch adds a NOTICE text change in a minor release.
- **Author follow-up:** The author answered review within a day and, per the brief, revised the patch through v5; this is a first submission, so there is no longer-term sign.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The fix affects only users who pass raw sha-crypt salts with overflowing rounds= values, but it closes a silent weakening of password hashes. The patch touches one source file and one test, and a reviewer has already said v5 looks good.

## Appendix

### Refuted in the counter-check

- `standards-compat-1` Backpatch changes hash output for salts that already wrap. Refuted (wrong facts): Message #9 says commit 426551c8cf49 already made strtoint() saturate and is on REL_18_STABLE, so overflowing tokens reach the existing clamp there without this patch. On that branch v5 changes only the NOTICE text and tests, so it does not cause the hash change described.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. [P12] and [P17] cover integer parsing in front-end utilities and command-line options, not pgcrypto salt parsing, and neither records a reason. The thread points to no earlier discussion of this idea, only to the strtoint() change in message #9, which this patch builds on.
- standards-compat: 1 objection(s). The other user-visible changes look acceptable from this angle. The NOTICE text now echoes the token, capped at 10 characters, and that changes no SQL syntax, GUC, catalog or dump behaviour.
- replication-ondisk: 0 objection(s). The patch changes how crypt() parses one salt option and what one NOTICE prints. It adds no WAL records, catalog changes, on-disk format changes, protocol messages or extension API changes. Stored hashes keep working because verification reads the rounds value written into the stored hash, which this patch leaves unchanged.
- security: 0 objection(s). From the security angle the change narrows a weakness: an overflowing rounds= value no longer gives a silently weaker 1000-round hash. A user could already ask for 999999999 rounds with an in-range value, so the clamp to the maximum adds no new denial-of-service path. The NOTICE echoes only the caller's own input, and message #6 and message #11 capped its length. Stored hashes keep the canonical rounds value, so verifying existing hashes is unaffected.
- maintenance-fit: 0 objection(s). This is a small bug fix inside existing pgcrypto code, and it adds no GUC, catalog, API, dependency or hot-path cost. It uses strtoint() saturation that is already backpatched, and the overflow tests would catch a later change to that behaviour. The statement_timeout test stability and the backpatch decision are implementation or committer questions, outside this angle. Neither precedent's recorded reason applies.
- already-exists: 0 objection(s). This is a bug fix for silent wraparound in crypt() rounds= parsing, so no existing feature, extension or idiom gives users correct clamping for raw salt strings. gen_salt() avoids the bug but does not fix crypt() on caller-supplied salts. The patch reuses the existing clamp-with-NOTICE path and the existing strtoint() helper, so it adds no second mechanism.
- demand: 0 objection(s). This is a bug fix for a reported bug (BUG #19743), where crypt() silently hashes with 1000 rounds instead of the value given, with no NOTICE. A field report, a reproduction on master and agreement from two reviewers (messages #2, #13) are enough demand for a correctness fix. The cost stays inside pgcrypto's sha-crypt path, so the demand angle has no objection to send.

### Precedents retrieved

- P1: [Upgrade pgcrypto to crypt_blowfish 1.3](https://commitfest.postgresql.org/patch/3338/) (cf, Returned with feedback, 2022-08-01) - judged not relevant; reason summarised from its thread
- P2: [Encoding protection for pgcrypto](https://commitfest.postgresql.org/patch/4809/) (cf, Returned with feedback, 2024-10-07) - judged not relevant; reason summarised from its thread
- P3: [Avoid obscure DES key buffer loop bounds in pgcrypto](https://commitfest.postgresql.org/patch/6953/) (cf, Withdrawn, 2026-09-25) - judged not relevant; reason summarised from its thread
- P4: [(DUPLICATE) Removing dead code in pgcrypto](https://commitfest.postgresql.org/patch/3860/) (cf, Withdrawn, 2022-08-30) - judged not relevant; reason summarised from its thread
- P5: [Show subtransaction overflow status](https://commitfest.postgresql.org/patch/3444/) (cf, Rejected, 2022-03-22) - judged not relevant; reason summarised from its thread
- P6: [Revert "Make some use of anonymous unions \[pgcrypto\]"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f5aabe6d58e08179cc656539c4bd0145525bf22f) (revert, Reverted, 2025-09-30) - judged not relevant
- P7: [Support target_session_attrs=read-only and eliminate the added round-trip to detect session status.](https://commitfest.postgresql.org/patch/1148/) (cf, Returned with feedback, 2017-11-28) - judged not relevant; reason summarised from its thread
- P8: [Support target_session_attrs=read-only and eliminate the added round-trip to detect session status.](https://commitfest.postgresql.org/patch/1568/) (cf, Returned with feedback, 2018-03-01) - judged not relevant; reason summarised from its thread
- P9: [Nonreplayable XLog records by means of overflows and >MaxAllocSize lengths](https://commitfest.postgresql.org/patch/3590/) (cf, Returned with feedback, 2022-12-02) - judged not relevant; reason summarised from its thread
- P10: [Improve overflow checks potentially optimized away by compiler](https://commitfest.postgresql.org/patch/400/) (cf, Returned with feedback, 2015-12-01) - judged not relevant
- P11: [Revert "Fix corner case with PGP decompression in pgcrypto"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=38f60f174e3279069b5547b5f4015eb4a8492037) (revert, Reverted, 2020-07-22) - judged not relevant
- P12: [standardize integer parsing in front-end utilities](https://commitfest.postgresql.org/patch/2197/) (cf, Withdrawn, 2020-03-05) - used in the review
- P13: [Sequence Access Methods, round two](https://commitfest.postgresql.org/patch/4677/) (cf, Returned with feedback, 2026-02-27) - judged not relevant
- P14: [parse-time support function for "any" datatype](https://commitfest.postgresql.org/patch/2081/) (cf, Withdrawn, 2021-03-09) - judged not relevant
- P15: [Revert "Add support for parsing of large XML data (>= 10MB)"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f2743a7d70e7b2891277632121bb51e739743a47) (revert, Reverted, 2024-01-26) - judged not relevant
- P16: [Extensible parsing](https://commitfest.postgresql.org/patch/3100/) (cf, Rejected, 2022-01-17) - judged not relevant
- P17: [Check integers in command line parameters](https://commitfest.postgresql.org/patch/1121/) (cf, Returned with feedback, 2017-10-02) - used in the review
- P18: [Apply a better fix to mdunlinkfork().](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=85d8b30724c0fd117a683cc72706f71b28463a05) (revert, Partially reverted, 2022-11-09) - judged not relevant
- P19: [lax reloptions parsing for views](https://commitfest.postgresql.org/patch/3340/) (cf, Returned with feedback, 2022-03-31) - judged not relevant
- P20: [improve tab-completion behavior when parse text in single quote](https://commitfest.postgresql.org/patch/3267/) (cf, Returned with feedback, 2021-12-03) - judged not relevant
- P21: [Revert refactoring of hex code to src/common/](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2576dcfb76aa71e4222bac5a3a43f71875bfa9e8) (revert, Reverted, 2021-08-19) - judged not relevant
- P22: [ALTER TABLE SET LOGGED/UNLOGGED on a partitioned table does nothing silently](https://commitfest.postgresql.org/patch/2857/) (cf, Withdrawn, 2021-04-07) - judged not relevant
- P23: [Simplify COPY FROM parsing by forcing lookahead.](https://commitfest.postgresql.org/patch/2974/) (cf, Returned with feedback, 2021-04-06) - judged not relevant
- P24: [Add OID descriptions to dumped parse/query/plan trees](https://commitfest.postgresql.org/patch/5987/) (cf, Withdrawn, 2025-08-25) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 12; wall time: 80 s; cost reported by the CLI: $1.36
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 7 from cache, 1 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 49 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T04:03:00.044Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

# Design pre-review: pgcrypto crypt() can return text with invalid encoding

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7417](https://commitfest.postgresql.org/patch/7417/) (Needs review); authors: Andrey Rachitskiy (a.rachitskiy)
- Thread: [BUG #19600: pgcrypto crypt() can return text with invalid encoding](https://www.postgresql.org/message-id/flat/19600-e5fb479f50f5022d%40postgresql.org), started 2026-08-03
- Messages read: 3 of 3
- Patch prose read: 0001-pgcrypto-Reject-crypt-results-invalid-in-database-encoding-BUG-19600.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

The goal and the design copy what pgp_*_decrypt_text already do, and no objection survived at levels 1a or 1b. The one surviving objection, standards-compat-1, is a fixable back-branch question: crypt(pw, stored_hash) starts to raise an error for stored hashes that hold invalid encoding. A committer who knows pgcrypto has to decide between master only and back branches.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read. The claim that the check reads only the finished result, and so leaves valid hashes unchanged, rests on message #3; the back branches were checked for applicability but not built.

## The proposal, restated

**Problem.** crypt() copies a fixed number of salt bytes into its text result, so a multibyte character that straddles the cut leaves a truncated sequence in the output. The server stores that text, but upper() then fails on it and COPY cannot reload it. Any user can trigger this, because crypt() is executable by PUBLIC; the documented gen_salt() idiom cannot, since gen_salt() emits ASCII only.

**Proposed design.** pg_crypt() calls pg_verifymbstr() on the result before it returns it, as pgp_sym_decrypt_text and pgp_pub_decrypt_text already do. A truncated result now raises "invalid byte sequence for encoding". The patch adds a crypt-utf8 regression test that non-UTF8 databases skip.

**User-visible surface.**
- crypt() raises "invalid byte sequence for encoding" when its result would hold a truncated multibyte sequence.
- crypt(pw, stored_hash) raises an error for stored hashes that already hold invalid encoding, for the right password and for a wrong one.

**What the author claims.**
- The reporter says this is the same class of bug the earlier pgp_*_decrypt_text fix covered, but crypt() has no bytea variant to fall back on.
- The reporter says traditional DES copies 2 salt bytes, xDES copies 9 and MD5 copies up to 8, so the cut point differs by algorithm.
- The reviewer reports that ASCII salts and valid multibyte salts return the same hashes as before the patch.
- The reviewer reports that SHA-crypt and Blowfish already reject non-ASCII salts.
- The reviewer reports the patch applies cleanly to REL_18 and REL_19, and to REL_14 through REL_17 with Makefile adjustments, but did not build them.

**Thread state.** The reporter, the author and one reviewer took part, and the reviewer tested the patch and called it good. The reviewer asked for xDES and MD5 test cases and a release-note mention of stored hashes that will now fail; the author has not replied.

## 2. Prior attempts

### Encoding protection for pgcrypto (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/4809/](https://commitfest.postgresql.org/patch/4809/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CAGRkXqRZyo0gLxPJqUsDqtWYBbgM14betsHiLRPj9mo2=z9VvA@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAGRkXqRZyo0gLxPJqUsDqtWYBbgM14betsHiLRPj9mo2=z9VvA@mail.gmail.com)
- **Date:** decided 2024-10-07; started 2024-02-05
- **Outcome:** Returned with feedback (cf)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (AI summary of the linked thread; check it there):** The visible messages do not show why. Cary Huang reported failing tests and questioned aborting the transaction instead of issuing a warning. The author posted v2 and argued that callers who want no encoding check can use the bytea variants, and no later message follows.
- **Reason (quote):** quote unavailable
- **What is different this time:** P2 added pg_verifymbstr() to pgp_sym_decrypt_text and pgp_pub_decrypt_text; this proposal adds the same check to crypt(), which has no bytea variant. Message #3 reports that this check later went in as d536aee5566.


## 3. Objections a reviewer is likely to raise

### 1. Back-patching turns working password checks into errors after minor upgrade

fixable · level 2, rubric item 6 · backpatchability, SQL-standard conflict or non-standard syntax · `standards-compat-1`

A site stored DES or MD5 hashes made with a hand-written non-ASCII salt. After a minor upgrade, `where pwhash = crypt($1, pwhash)` raises an error instead of returning false, so those users cannot log in. Any query that evaluates crypt() over several rows fails as a whole if one row holds such a hash.

Message #3 says these hashes stop working for the right password and for a wrong one, and that crypt() has no bytea variant to fall back on. Message #3 also checked that the patch applies to REL_14 through REL_19, but the thread has not decided whether to back-patch. Committers usually keep minor releases free of changes that make working queries fail. The proposal also gives no way to find or rehash affected rows before upgrading, and the author has not replied to the release-note request.

Thread: [#2 (Andrey Rachitskiy, 2026-08-03)](https://www.postgresql.org/message-id/CAB8bMivmoRuUcNF1pEFx%3DTr2Q_b31VwtJse4P22pTMgoG_uC1w%40mail.gmail.com), [#3 (Ishjaap Singh, 2026-10-28)](https://www.postgresql.org/message-id/CA%2BY2BmeT0uQBP5ewhSqxTOAx6COr0uEYuLpiBfeyFRxD5e3ZDw%40mail.gmail.com)

**Question:** Will this go to master only, and if back-patched, what query lets users find affected stored hashes before the upgrade, and how do they rehash them?

Counter-check: held. Message #3 confirms that stored invalid hashes raise an error for right and wrong passwords, and it calls the trade-off acceptable. The thread does not decide on back-patching or give a way to find affected rows, so the objection stands as fixable.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | No format change, but stored hashes with invalid encoding survive pg_upgrade and then make crypt() raise an error. |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | **yes** | see objection standards-compat-1: Back-patching turns working password checks into errors after minor upgrade |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | **yes** | The fix stops crypt() from producing text that COPY cannot reload; rows already stored stay unrestorable. |
| psql UX conventions | no |  |
| security / privileges | **yes** | crypt() is executable by PUBLIC, so any user can store invalid text today; the patch closes that path. |
| backpatchability | **yes** | Message #3 checked REL_14 through REL_19; a back-patch makes existing password checks raise an error after a minor upgrade. |
| maintenance burden ("who maintains this in 10 years") | **yes** | Small: one pg_verifymbstr() call in pg_crypt() and one regression test with an alternative expected file for non-UTF8 databases. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Should this fix go to master only, or also to REL_14 through REL_19, where existing stored hashes start to raise errors after a minor upgrade?
2. What query lets users find stored crypt() hashes that hold invalid encoding before they upgrade?
3. How should a user rehash an affected row, given that crypt(pw, stored_hash) now raises an error even for the right password?
4. Was rejecting non-ASCII salts up front for DES, xDES and MD5 considered, as SHA-crypt and Blowfish already do?
5. Will the next version add the xDES and MD5 test cases and the release-note text that message #3 asked for?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. No committer has spoken in the thread. It needs someone who knows pgcrypto and the back-branch policy for behaviour changes.
- **Liability:** The owner signs for the back-branch decision: failed logins on sites with affected stored hashes after a minor upgrade, and the release-note text. The code itself is one check that copies existing pgcrypto practice.
- **Author follow-up:** The author posted the patch on the day of the report but has not answered the review in message #3, about two weeks old at the review date.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs champion
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** Only callers who pass a hand-written non-ASCII salt to DES, xDES or MD5 crypt() are affected, and the 4.3 KB patch touches pgcrypto only. One reviewer tested it and called it good; what remains is a committer's decision on back branches.

## Appendix

### Refuted in the counter-check

- `replication-ondisk-1` Back-patch turns already stored hashes into errors at minor upgrade. Refuted (not specific): Duplicate of standards-compat-1. Its point that pg_upgrade and physical standbys keep such rows alive supports that objection but is not a separate one.
- `maintenance-fit-1` Back-patching turns unrestorable hashes into failed logins after a minor update. Refuted (not specific): Duplicate of standards-compat-1. It describes the same failed logins after a minor update and asks the same question about master-only versus back branches.

### Review angles

- prior-attempts: 0 objection(s). The only precedent, [P2], is the same check on other pgcrypto functions, and its reason is recorded only as an AI summary. That summary names two concerns: tests failing under some encodings, and raising an error where a warning might do. This patch adds a skip file and was tested in UTF8, LATIN1 and SQL_ASCII databases. Message #3 weighed the error-versus-warning trade-off for crypt(), which has no bytea fallback, and judged it acceptable. Whether the fix should go to back branches, given that stored hashes start to fail after a minor upgrade, falls under the backpatchability angle, not prior attempts.
- standards-compat: 1 objection(s). The contract change itself fits existing practice: pgp_*_decrypt_text already validates its text result. No new syntax, GUC or catalog object is added. The fix also restores the pg_dump and COPY round-trip.
- replication-ondisk: 1 objection(s). Apart from existing stored data, the patch changes no on-disk format, WAL record, catalog, wire protocol or extension ABI. Standbys and subscribers are not affected differently from the primary.
- security: 0 objection(s). From the security angle there is nothing to object to. The patch removes the one case where an unprivileged user can store text that is invalid in the database encoding through crypt(), and it adds no new privilege, exposure or default. The new error shows only bytes the caller would have received anyway, and the lockout of already-stored invalid hashes is an upgrade and back-branch question for other angles.
- maintenance-fit: 1 objection(s). Beyond the back-branch question, the maintenance and fit cost is small. The check copies an existing pgcrypto practice, adds negligible work to a function that is slow by design, and adds no API or new mechanism.
- already-exists: 0 objection(s). No existing feature does this job. The gen_salt() idiom avoids the bug only when callers use it, and crypt() has no bytea variant, so nothing stops invalid text reaching a table. The patch reuses pg_verifymbstr(), the check pgp_*_decrypt_text already use, so it extends an existing mechanism and does not add a second one.
- demand: 0 objection(s). This is a bug fix. crypt() stores text that is invalid in the database encoding, and then upper() and COPY reload fail on it, so demand needs no proof from field reports. The cost falls only on callers who pass non-ASCII salts. The real question is the effect on already stored hashes in back branches, which belongs to the compatibility angle.

### Precedents retrieved

- P1: [Upgrade pgcrypto to crypt_blowfish 1.3](https://commitfest.postgresql.org/patch/3338/) (cf, Returned with feedback, 2022-08-01) - judged not relevant; reason summarised from its thread
- P2: [Encoding protection for pgcrypto](https://commitfest.postgresql.org/patch/4809/) (cf, Returned with feedback, 2024-10-07) - used in the review; reason summarised from its thread
- P3: [(DUPLICATE) Removing dead code in pgcrypto](https://commitfest.postgresql.org/patch/3860/) (cf, Withdrawn, 2022-08-30) - judged not relevant; reason summarised from its thread
- P4: [Revert "Make some use of anonymous unions \[pgcrypto\]"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f5aabe6d58e08179cc656539c4bd0145525bf22f) (revert, Reverted, 2025-09-30) - judged not relevant
- P5: [Revert "Fix corner case with PGP decompression in pgcrypto"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=38f60f174e3279069b5547b5f4015eb4a8492037) (revert, Reverted, 2020-07-22) - judged not relevant
- P6: [Rename "getdatabaseencoding()" to "pg_database_encoding()", and document](https://commitfest.postgresql.org/patch/7024/) (cf, Withdrawn, 2026-07-18) - judged not relevant; reason summarised from its thread
- P7: [Clarify the comments about varlena header encoding](https://commitfest.postgresql.org/patch/3820/) (cf, Returned with feedback, 2022-10-12) - judged not relevant; reason summarised from its thread
- P8: [Revert COPY OUT to follow the pre-8.3 handling of ASCII control characters, namely that \r, \n, \t, \b, \f, \v are dumped as those two-character...](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a63b63ff96293e153d25e8e054a830d70f69938a) (revert, Reverted, 2007-12-03) - judged not relevant
- P9: [Revert non-text output formats for pg_dumpall](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=7ca548f23a60887541493b2da47f2b2720c35ee3) (revert, Reverted, 2026-06-18) - judged not relevant
- P10: [Remove unused function parameters, part 1: contrib](https://commitfest.postgresql.org/patch/6263/) (cf, Withdrawn, 2026-02-13) - judged not relevant; reason summarised from its thread
- P11: [Missing dependency on oauth_validator test](https://commitfest.postgresql.org/patch/6407/) (cf, Withdrawn, 2026-01-19) - judged not relevant; reason summarised from its thread
- P12: [NOT VALID for Unique Indexes](https://commitfest.postgresql.org/patch/2937/) (cf, Returned with feedback, 2021-10-01) - judged not relevant; reason summarised from its thread
- P13: [Convert contrib SQL functions to new style](https://commitfest.postgresql.org/patch/3074/) (cf, Returned with feedback, 2021-09-01) - judged not relevant
- P14: [to_date_valid()](https://commitfest.postgresql.org/patch/662/) (cf, Returned with feedback, 2016-09-29) - judged not relevant
- P15: [Revert "Convert contrib/seg's bool-returning SQL functions to V1 call convention."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f050423052bc9265d4cd27555058435edd4bef87) (revert, Reverted, 2016-04-28) - judged not relevant
- P16: [Prevent multibyte characters from being quoted in format() and quote_ident()](https://commitfest.postgresql.org/patch/491/) (cf, Rejected, 2016-01-28) - judged not relevant
- P17: [COPY: validate option presence rather than option values](https://commitfest.postgresql.org/patch/6744/) (cf, Withdrawn, 2026-06-29) - judged not relevant
- P18: [Invalid permission check in pg_stats for functional indexes](https://commitfest.postgresql.org/patch/2274/) (cf, Returned with feedback, 2020-07-05) - judged not relevant
- P19: [Validate user-supplied c_args in meson builds](https://commitfest.postgresql.org/patch/6596/) (cf, Withdrawn, 2026-06-10) - judged not relevant
- P20: [Completed unaccent dictionary with many missing characters](https://commitfest.postgresql.org/patch/3631/) (cf, Returned with feedback, 2023-01-31) - judged not relevant
- P21: [Revert part of commit dddf4cdc3.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=c114229ca2519620703a4be4e38181290cad8c0a) (revert, Reverted, 2019-10-25) - judged not relevant
- P22: [bytea PL/Perl transform](https://commitfest.postgresql.org/patch/4252/) (cf, Returned with feedback, 2024-10-07) - judged not relevant
- P23: [Revert "Fix broken multibyte regression tests."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ad2e233385ee3e441454f20a02ac0f68f7873bc2) (revert, Reverted, 2016-01-25) - judged not relevant
- P24: [Non-volatile WAL buffer](https://commitfest.postgresql.org/patch/2615/) (cf, Withdrawn, 2021-06-18) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 14; wall time: 89 s; cost reported by the CLI: $1.05
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 5 from cache, 3 summarised now, 0 failed; quotes: 3 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 62 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:53:20.072Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

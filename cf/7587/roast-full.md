# Design pre-review: pg_combinebackup: Don't point only to pg_checksums for mixed checksum states

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7587](https://commitfest.postgresql.org/patch/7587/) (Needs review); authors: Raja Sai pranav (pranav-error)
- Thread: [Offline data checksum changes can cause incorrect checksum state on standbys](https://www.postgresql.org/message-id/flat/anwm6UPxoVS41QA2%40bdtpg), started 2026-08-12
- Messages read: 44 of 44
- Patch prose read: v8-021-test-fixes.txt, v1-0001-Keep-offline-data-checksum-changes-local.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: low)

No objection survived at levels 1a, 1b or 2. The patch rewords one pg_combinebackup hint and one documentation paragraph and leaves the refusal as it is. Confidence is low because the material shows neither the new hint wording nor the new docs text, and nobody has replied to message #44.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The new hint wording and the new docs text were not in the material, so their accuracy and clarity could not be checked. Whether the hint still gives the user a next step depends on that wording.

## The proposal, restated

**Problem.** pg_combinebackup refuses a backup chain whose backups have different data checksum states. Its hint then says the state was changed "with pg_checksums", even when the user enabled checksums online with pg_enable_data_checksums(). The same wrong hint appears when the full backup was taken while checksums were still being enabled (version 3 in pg_control).

**Proposed design.** The patch removes "with pg_checksums" from the hint. It also rewrites the Limitations paragraph of the pg_combinebackup docs to cover offline enabling, online enabling and the in-progress state. The refusal itself stays as it is.

**User-visible surface.**
- Changed hint text for the pg_combinebackup error "only some backups have checksums enabled".
- Changed Limitations paragraph in the pg_combinebackup documentation.

**What the author claims.**
- The current hint names a tool that the user may never have run.
- pg_combinebackup tests and test_checksums 024_combinebackup_mixed (PG_TEST_EXTRA=checksum) pass, and the docs validate.
- After an online enable, the refusal is stricter than needed, because the worker WAL-logs every page and the next incremental backup sends every relation file in full.
- pg_combinebackup cannot tell an online enable from an offline one using the control files, so refusing both is the safe choice.

**Thread state.** The 44-message thread produced the refusal in pg_combinebackup, which was pushed to master in September and not yet backpatched at that time. Message #44 posts this follow-up patch and nobody has replied to it.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-08-12; 24 candidates retrieved, none is the same or a similar idea).

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
| security / privileges | no |  |
| backpatchability | **yes** | The refusal comes from the v19 open-item work, but the CommitFest entry targets version 20. |
| maintenance burden ("who maintains this in 10 years") | no |  |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. What is the exact new hint text, and does it still tell the user what to do next, for example take a new full backup?
2. Should this change go into 19 together with the refusal, given that the CommitFest entry targets version 20?
3. Should the Limitations paragraph say that the refusal is stricter than needed after an online enable?
4. Does the new docs text say what the user should do when the full backup was taken while checksums were still being enabled?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows pg_combinebackup and the v19 data checksums work would own it. Committers took part earlier in the thread, but none has replied to message #44.
- **Liability:** One hint string, with its translations, and one documentation paragraph. The owner also decides whether to backpatch to 19, where the refusal may land.
- **Author follow-up:** No sign either way; the author posted the patch in message #44 and created the CommitFest entry, and no review has arrived yet.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** Only users who change the checksum state inside an incremental backup chain see this hint, but for them the current text names a tool they may never have run. The patch changes one string and one docs paragraph, so review is short.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent found: the precedent search returned no row about this or a similar idea. The thread does not point to an earlier attempt to change this hint, and the patch only rewords a hint and a docs paragraph, so no earlier rejection reason applies.
- standards-compat: 0 objection(s). From the standards and compatibility angle, the patch changes only the wording of one hint and one documentation paragraph. It adds no syntax, GUC, catalog object or dump format, and the behaviour stays the same. A shorter hint without the tool name still tells the user what to do. The new hint and docs text are not in the material, so their wording could not be checked. Whether the change goes into 19 together with the refusal is a backpatch question for the committer, not a compatibility risk.
- replication-ondisk: 0 objection(s). The patch changes only a client-side hint string and a documentation paragraph. It adds no on-disk, WAL, control file, catalog, protocol or ABI change, and the refusal logic stays as it is, so this angle has nothing to object to.
- security: 0 objection(s). The patch changes only a hint string and a documentation paragraph for pg_combinebackup, a client tool run by whoever already holds the backup files. Nothing changes who can do or see what, and the refusal itself stays as it is, so there is no security or privilege aspect to object to.
- maintenance-fit: 0 objection(s). The patch changes one hint string and one docs paragraph and adds no code path, GUC, catalog or API, so it carries no ongoing maintenance or hot-path cost. Not checked: the new hint and docs wording, which the material does not show, and whether the change should go into 19 along with the refusal.
- already-exists: 0 objection(s). The patch only corrects a hint and a docs paragraph for a refusal added in this release cycle. No existing feature, extension or tool gives users a correct hint, and the patch adds no new mechanism, so this angle has nothing to object to.
- demand: 0 objection(s). No demand objection applies. The patch fixes a hint that names pg_checksums when the user enabled checksums online with pg_enable_data_checksums(), and the reproducer in message #44 shows this. Every user of the new online-enable path who meets the refusal sees that hint, and the change adds no new surface, cost or maintenance burden.

### Precedents retrieved

- P1: [teach pg_combinebackup to detect deleted files where possible](https://commitfest.postgresql.org/patch/4951/) (cf, Withdrawn, 2024-08-21) - judged not relevant; reason summarised from its thread
- P2: [Enabling data checksums in an online cluster](https://commitfest.postgresql.org/patch/2611/) (cf, Withdrawn, 2021-03-09) - judged not relevant; reason summarised from its thread
- P3: [Revert "Allow on-line enabling and disabling of data checksums"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a228cc13aeabff308d6dfc98a1015865f5393fce) (revert, Reverted, 2018-04-09) - judged not relevant
- P4: [Online checksums verification](https://commitfest.postgresql.org/patch/2369/) (cf, Returned with feedback, 2020-11-05) - judged not relevant
- P5: [SLRU data page checksums](https://commitfest.postgresql.org/patch/1682/) (cf, Returned with feedback, 2018-10-02) - judged not relevant; reason summarised from its thread
- P6: [Fix checksum verification in base backups for zero page headers](https://commitfest.postgresql.org/patch/2719/) (cf, Returned with feedback, 2021-09-13) - judged not relevant
- P7: [Fix base backup checksum verification for random or zero page headers](https://commitfest.postgresql.org/patch/2308/) (cf, Returned with feedback, 2020-07-30) - judged not relevant
- P8: [Online verification of checksums](https://commitfest.postgresql.org/patch/1733/) (cf, Returned with feedback, 2019-04-09) - judged not relevant
- P9: [Allow pg_verify_checksums to specify a database to scan](https://commitfest.postgresql.org/patch/1782/) (cf, Returned with feedback, 2018-10-02) - judged not relevant; reason summarised from its thread
- P10: [invalid data in file backup_label problem on windows](https://commitfest.postgresql.org/patch/2938/) (cf, Withdrawn, 2021-04-06) - judged not relevant; reason summarised from its thread
- P11: [Checksums for slru files](https://commitfest.postgresql.org/patch/1462/) (cf, Returned with feedback, 2018-03-02) - judged not relevant; reason summarised from its thread
- P12: [An attempt to avoid locally-committed-but-not-replicated-to-standby-transactions in synchronous replication](https://commitfest.postgresql.org/patch/3665/) (cf, Returned with feedback, 2023-08-02) - judged not relevant; reason summarised from its thread
- P13: [Add GiST support for mixed-width integer operators](https://commitfest.postgresql.org/patch/5113/) (cf, Returned with feedback, 2024-10-07) - judged not relevant; reason summarised from its thread
- P14: [Synchronize logical replication slots from primary to standby](https://commitfest.postgresql.org/patch/3393/) (cf, Returned with feedback, 2022-02-18) - judged not relevant
- P15: [Online enabling of checksums](https://commitfest.postgresql.org/patch/1535/) (cf, Returned with feedback, 2019-01-31) - judged not relevant
- P16: [File level incremental backup](https://commitfest.postgresql.org/patch/94/) (cf, Rejected, 2015-03-18) - judged not relevant
- P17: [Warn if initdb's --sync-only option is mixed with other options](https://commitfest.postgresql.org/patch/3239/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P18: [Avoid mixing custom and OpenSSL BIO functions](https://commitfest.postgresql.org/patch/4834/) (cf, Withdrawn, 2024-03-12) - judged not relevant
- P19: [Replica sends an incorrect epoch in its hot standby feedback to the Master](https://commitfest.postgresql.org/patch/2462/) (cf, Withdrawn, 2020-03-03) - judged not relevant
- P20: [HINTing on UPDATE foo SET foo.bar = ..](https://commitfest.postgresql.org/patch/415/) (cf, Returned with feedback, 2016-03-09) - judged not relevant
- P21: [idle-in-transaction timeout error does not give a hint](https://commitfest.postgresql.org/patch/1892/) (cf, Withdrawn, 2019-03-31) - judged not relevant
- P22: [Warning for Datatype mismatch between subscriber table and publisher table for logical replication](https://commitfest.postgresql.org/patch/5648/) (cf, Returned with feedback, 2025-05-13) - judged not relevant
- P23: [NLS: use gettext() to translate system error messages](https://commitfest.postgresql.org/patch/6172/) (cf, Rejected, 2026-03-24) - judged not relevant
- P24: [Replica sends an incorrect epoch in its hot standby feedback to the Master](https://commitfest.postgresql.org/patch/2463/) (cf, Withdrawn, 2020-03-03) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 13; wall time: 91 s; cost reported by the CLI: $2.44
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 5 from cache, 3 summarised now, 0 failed; quotes: 3 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 43 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T01:01:53.851Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

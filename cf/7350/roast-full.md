# Design pre-review: pg_dump: Fetch sequence data only for sequences being dumped

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7350](https://commitfest.postgresql.org/patch/7350/) (Needs review); authors: Andrew Krylosov (krylosov-aa)
- Thread: [BUG #19688: pg_dump --schema scans all sequences in PostgreSQL 18, causing severe performance regression](https://www.postgresql.org/message-id/flat/19688-e90025dc375a22a3%40postgresql.org), started 2026-09-14
- Messages read: 2 of 2
- Patch prose read: v1_PG18-0001-pg_dump-Fetch-sequence-data-only-for-sequences-be.patch.txt, v1-0001-pg_dump-Fetch-sequence-data-only-for-sequences-be.patch, v1_PG19-0001-pg_dump-Fetch-sequence-data-only-for-sequences-be.patch.txt (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

The problem is stated with numbers and the demand is a field report of a v18 regression, so levels 1a and 1b pass. One fixable design objection survived, prior-attempts-1: the thread does not show that the OID list covers every sequence whose value pg_dump emits under binary upgrade, extension configuration or --data-only with filters. That is a coverage question for someone who knows pg_dump, and it does not put the idea in doubt.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read. How the patch builds the OID list, and whether it covers binary-upgrade, extension member and identity sequences, could not be checked.

## The proposal, restated

**Problem.** Since commit bd15b7db48 (v18), pg_dump calls pg_get_sequence_data() for every sequence in the database, even for a dump limited by --schema or --table. Users with many sequences, such as schema-per-tenant databases, see selective dumps become much slower than in v16. Such a dump can also block on a lock held on a sequence it does not dump.

**Proposed design.** collectSequences() passes the OIDs of the sequences whose data will be dumped to its query, and calls pg_get_sequence_data() only for those. Definitions are still fetched for all sequences from pg_sequence, a catalog scan. The author offers versions for HEAD, REL_19_STABLE and REL_18_STABLE.

**What the author claims.**
- The reporter saw a schema dump go from about 50 seconds on v16 to about 5.5 minutes on 18.6, with 142237 sequences in the database and 329 in the schema.
- With the reporter's own patch, the same dump took about 50 seconds again, with identical TOC, SQL output and restored sequence state.
- On HEAD with 20000 sequences in one schema and 4 in another, "pg_dump -n small" dropped from about 1.3 s to 0.33 s.
- A selective dump no longer waits on an AccessExclusiveLock held on an unrelated sequence, such as an uncommitted DROP SEQUENCE.
- The fix is a v18 regression fix and should be backpatched through 18; tests and output comparisons pass on all branches.

**Thread state.** Two messages on pgsql-bugs: the bug report with a proof-of-concept patch, and Andrew Krylosov's reply with v1 patches for three branches. No committer or reviewer has replied in the thread, though three reviewers are listed in the CommitFest entry.

## 2. Prior attempts

### Ignore attributes of tables part of extensions in pg_dump (related)

- **Link:** [https://commitfest.postgresql.org/patch/164/](https://commitfest.postgresql.org/patch/164/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CAB7nPqSO28tQMyvCYHVuDO_jYhzBbAE17whhpPJ0-TyCbYky6w@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAB7nPqSO28tQMyvCYHVuDO_jYhzBbAE17whhpPJ0-TyCbYky6w@mail.gmail.com)
- **Date:** decided 2015-02-23; started 2015-02-16
- **Outcome:** Rejected (cf)
- **Who objected:** Michael Paquier (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** The author withdrew the patch. A reviewer asked why it checked ext_member rather than the usual dobj.dump flag. The author concluded that skipping attributes based on either flag would break binary upgrade, which still dumps extension member tables individually.
- **Reason (quote):** "Hm. I have been pondering about this code more and I am dropping the patch as either adding a check based on the flag to track dumpable objects or ext_member visibly breaks the logic that binary upgrade and tables in extensions rely on." ([message](https://www.postgresql.org/message-id/CAB7nPqTyYdQeg%2BaR3sC1y45AfXZg8FiXsffz0P3c4KqkBxj_5Q%40mail.gmail.com))
- **What is different this time:** P7 skipped column fetching for extension member tables. This patch limits pg_get_sequence_data() calls to sequences whose data is dumped and keeps fetching definitions for all sequences.


## 3. Objections a reviewer is likely to raise

### 1. Data-OID filter may miss sequences dumped through binary upgrade or extensions

fixable · level 2, rubric item 7 · pg_dump / restore round-trip, on-disk format / pg_upgrade · `prior-attempts-1`

A user runs pg_upgrade, or dumps an extension whose configuration sequence is marked with pg_extension_config_dump. If the OID list misses a sequence whose value pg_dump still emits, the restore writes a wrong or default last_value. The next inserts then fail with duplicate keys, possibly weeks after the upgrade.

[P7] was dropped because filtering a catalog fetch on pg_dump's dump flags "visibly breaks the logic that binary upgrade and tables in extensions rely on". This patch also filters a bulk fetch, here by which sequences have data dumped. Message #2 and the commit message do not cover binary upgrade, extension member sequences, or --data-only with filters. The reporter's tests in message #1 cover one --schema dump.

**Precedents.**
- [Ignore attributes of tables part of extensions in pg_dump](https://commitfest.postgresql.org/patch/164/) (Rejected, 2015-02-23); thread: [link](https://www.postgresql.org/message-id/flat/CAB7nPqSO28tQMyvCYHVuDO_jYhzBbAE17whhpPJ0-TyCbYky6w@mail.gmail.com)

Thread: [#1 (PG Bug reporting form, 2026-09-14)](https://www.postgresql.org/message-id/19688-e90025dc375a22a3%40postgresql.org), [#2 (Andrew Krylosov, 2026-09-26)](https://www.postgresql.org/message-id/CA%2Bnn4-qhKbHqXKtWc7asZAuHZE%2B6urDBxebxF1uCLz767khSAw%40mail.gmail.com)

**Question:** For binary-upgrade dumps, extension configuration sequences and --data-only with filters, is every sequence whose value pg_dump emits in the OID list, and which test shows it?

Counter-check: held. The thread and commit messages do not say how the patch builds the OID list, and the tests shown cover only --schema dumps. [P7] concerned column fetches for extension tables, so it supports this only as an analogy.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | pg_upgrade runs pg_dump --binary-upgrade, which uses the changed collectSequences() query. |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | **yes** | The patch changes which sequences get last_value and is_called fetched for the dump. |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The author asks for a backpatch through 18 and supplies versions for REL_18_STABLE and REL_19_STABLE. |
| maintenance burden ("who maintains this in 10 years") | no |  |
| performance on the hot path | **yes** | The patch changes the sequence query that every pg_dump run against a v18 or later server executes. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. For binary-upgrade dumps, extension configuration sequences and --data-only with filters, is every sequence whose value pg_dump emits in the OID list, and which test shows it?
2. Does a full dump keep the existing bulk query, or does it also pass an OID list?
3. How does the query behave when a partial dump still includes more than 100000 sequences, for example a dump that excludes one small schema?
4. Does the patch add a test for a selective dump and for the lock wait on a sequence outside the dump?
5. Do the REL_18_STABLE and REL_19_STABLE versions produce the same output as the unpatched branches for a full dump and for pg_upgrade?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. A committer who knows pg_dump's object selection, binary-upgrade mode and the sequence data collection added by commit bd15b7db48.
- **Liability:** A backpatched change to pg_dump in 18, 19 and HEAD. If the OID list misses a sequence, a restore or pg_upgrade writes a wrong sequence value, and the error may appear long after.
- **Author follow-up:** The author supplied versions for three branches and created the CommitFest entry. The thread shows no later reply from the author.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** Message #1 reports a schema dump going from about 50 seconds to about 5.5 minutes on a schema-per-tenant database. The patch touches one function in pg_dump.c and adds no user-visible surface.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 1 objection(s). No listed precedent proposed the same fix. [P1] went the other way, toward a bulk query, and its 'results do not justify the risk' reason does not fit a measured v18 regression. Message #2 reports that the problem was first noted in the bug #19365 discussion; the thread gives no rejection reason from it.
- standards-compat: 0 objection(s). The patch changes no syntax, option, GUC, psql behaviour or dump format, and the reporter found identical TOC, SQL output and restored sequence state, so the user-facing contract is unchanged. One point needs the code to check: whether every sequence whose data is dumped, such as identity sequences, extension configuration sequences, or sequences in --data-only and binary-upgrade runs, is already marked when collectSequences() builds its OID list.
- replication-ondisk: 0 objection(s). The patch changes only which rows pg_dump's own catalog query returns. It has no on-disk format, WAL, catalog, wire protocol or server ABI change, and server state on primaries and standbys is untouched. One point cannot be checked without the code: under --binary-upgrade, which pg_upgrade uses, every sequence that needs its last_value must still be in the OID list, given the extension-member issue recorded in [P7].
- security: 0 objection(s). The proposal has no new privilege or exposure aspect. It narrows which sequences pg_dump reads and locks, so a selective dump touches fewer objects and no longer waits on locks held on sequences it does not dump. It adds no new role, function, setting or visible output, so users can do or see nothing new.
- maintenance-fit: 0 objection(s). From the maintenance angle this is a narrow fix to a v18 regression, confined to collectSequences() in pg_dump.c, with no new GUC, catalog, syntax or extension API. Restricting the query to an OID array copies how pg_dump already batches per-object queries, and full dumps lose nothing. Neither precedent applies: [P7] skipped data that binary upgrade needed, and [P1] traded memory for little gain, while this patch reduces work. Points that cannot be checked without the code: how the OID array behaves when a dump excludes one small schema and keeps over 100000 sequences, whether binary-upgrade and extension-member sequences still get their data rows, and whether a test covers selective dumps.
- already-exists: 0 objection(s). No existing feature does this job. Postgres has no pg_dump option, GUC or extension that skips pg_get_sequence_data() for sequences outside a --schema or --table dump, and the user cannot work around it without patching pg_dump. The patch fixes a v18 regression inside the existing mechanism and adds no second mechanism.
- demand: 0 objection(s). The material shows real demand: a field report with numbers (142237 sequences, a schema dump going from about 50 seconds to about 5.5 minutes) on a common schema-per-tenant design, with daily per-schema backups no longer fitting in a day. This is a v18 regression with no user-visible surface, and its cost falls on nobody else.

### Precedents retrieved

- P1: [pg_dump: optimize dumpFunc()](https://commitfest.postgresql.org/patch/5147/) (cf, Withdrawn, 2024-08-02) - used in the review; reason summarised from its thread
- P2: [Improve pg_dump dumping publication tables](https://commitfest.postgresql.org/patch/2728/) (cf, Withdrawn, 2021-01-15) - judged not relevant; reason summarised from its thread
- P3: [pg_dump --where option](https://commitfest.postgresql.org/patch/2600/) (cf, Returned with feedback, 2020-09-30) - judged not relevant; reason summarised from its thread
- P4: [pg_dump new feature: exporting functions only](https://commitfest.postgresql.org/patch/3051/) (cf, Returned with feedback, 2022-04-07) - judged not relevant; reason summarised from its thread
- P5: [pg_dump -C Grant statement on create database](https://commitfest.postgresql.org/patch/724/) (cf, Returned with feedback, 2016-10-03) - judged not relevant; reason summarised from its thread
- P6: [Add --include-table-data-where option to pg_dump, to export only a subset of table data](https://commitfest.postgresql.org/patch/1644/) (cf, Returned with feedback, 2019-01-31) - judged not relevant; reason summarised from its thread
- P7: [Ignore attributes of tables part of extensions in pg_dump](https://commitfest.postgresql.org/patch/164/) (cf, Rejected, 2015-02-23) - used in the review; reason summarised from its thread
- P8: [Improving test coverage of extensions with pg_dump](https://commitfest.postgresql.org/patch/187/) (cf, Returned with feedback, 2015-12-02) - judged not relevant
- P9: [Revert "pg_dump: Lock all relations, not just plain tables".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=d3adaabaf7d555ec8bb1d83c43f72e79f1bf0b7d) (revert, Reverted, 2020-11-06) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 11; wall time: 97 s; cost reported by the CLI: $1.18
- Precedent enrichment: 7 thread(s) lacked a recorded reason: 7 from cache, 0 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 2 of 9 kept
- Tone check: 50 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:43:22.588Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

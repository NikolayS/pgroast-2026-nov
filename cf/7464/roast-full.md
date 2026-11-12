# Design pre-review: WITHOUT OVERLAPS foreign key allows referencing EXCLUDE constraint

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7464](https://commitfest.postgresql.org/patch/7464/) (Needs review); authors: Paul Jungwirth (pjungwir)
- Thread: [WITHOUT OVERLAPS foreign key allows referencing EXCLUDE constraint](https://www.postgresql.org/message-id/flat/CA%2BrenyUy%3D%2BoZoOg0qN_wAHTL3MwVEnp4zdfX%2BBu54foz6OXpQw%40mail.gmail.com), started 2026-09-30
- Messages read: 1 of 1
- Patch prose read: v1-0001-Fix-temporal-FKs-referencing-invalid-constraints.patch, v1-0002-Warn-about-temporal-FKs-that-reference-exclusion-.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

The fix closes a correctness hole reported by a committer, and no objection survived at levels 1a or 1b. Two fixable level 2 points remain about back-branch behaviour: standards-compat-1 (a dump from an earlier 18.x fails to restore the foreign key on a fixed 18.x) and standards-compat-2 (the 18-only warning assumes the fix lands before 19.0). Both are release-policy choices that a committer who knows back-patching practice has to settle.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so the new error texts, the pg_upgrade report format and the exact conditions and frequency of the 18.x warning could not be checked. The material also does not show whether 19.0 has shipped, which decides whether standards-compat-2 applies.

## The proposal, restated

**Problem.** A foreign key with PERIOD can reference a plain EXCLUDE constraint, which does not guarantee uniqueness and whose updates do not conflict with FOR KEY SHARE. This allows serialization anomalies, as Andres Freund reported. Users of temporal foreign keys in 18.x are affected.

**Proposed design.** The patch requires the referenced index to have indisunique as well as indisexclusion, so only WITHOUT OVERLAPS primary keys and unique constraints qualify. pg_upgrade gains a check for existing bad foreign keys. A second, REL_18_STABLE-only commit warns when the server builds an RI plan for such a foreign key or clones one to a new partition.

**User-visible surface.**
- New error when a temporal foreign key references an exclusion constraint or a primary key without WITHOUT OVERLAPS.
- New pg_upgrade check that reports temporal foreign keys referencing non-unique indexes.
- New warning in 18.x when the server uses or clones such a foreign key.
- Restoring a pg_dump from 18.x that contains such a foreign key fails on fixed versions.

**What the author claims.**
- A temporal foreign key that references a WITHOUT OVERLAPS key already conflicts with FOR KEY SHARE, so it has no anomaly.
- Few users likely created temporal foreign keys on plain exclusion constraints.
- A primary key without WITHOUT OVERLAPS already failed before, but with an unhelpful error.
- Fixing this before 19 is released limits the warning code to the 18 branch.
- A catalog query, proposed for the release notes, finds affected foreign keys.

**Thread state.** Only the first message is visible. The author asks whether to add the fix to the 19 Open Items list, and Peter Eisentraut is listed as reviewer in the CommitFest.

## 2. Prior attempts

### Revert temporal primary keys and foreign keys (related)

- **Link:** [https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=8aee330af55d8a759b2b73f5a771d9d34a7b887f](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=8aee330af55d8a759b2b73f5a771d9d34a7b887f)
- **Thread:** [https://www.postgresql.org/message-id/flat/d0b64a7a-dfe4-4b84-a906-c7dedfa40a3e@eisentraut.org](https://www.postgresql.org/message-id/flat/d0b64a7a-dfe4-4b84-a906-c7dedfa40a3e@eisentraut.org), [https://www.postgresql.org/message-id/flat/CA+renyUApHgSZF9-nd-a0+OPGharLQLO=mDHcY4_qQ0+noCUVg@mail.gmail.com](https://www.postgresql.org/message-id/flat/CA+renyUApHgSZF9-nd-a0+OPGharLQLO=mDHcY4_qQ0+noCUVg@mail.gmail.com)
- **Date:** decided 2024-05-16; started 2024-01-19
- **Outcome:** Reverted (revert)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (as recorded in the corpus):** This feature set did not handle empty ranges correctly, and it's now too late for PostgreSQL 17 to fix it. The following commits are reverted: 6db4598fcb8 Add stratnum GiST support function 46a0cd4cefb Add temporal PRIMARY KEY and UNIQUE constraints 86232a49a43 Fix comment on gist_stratnum_btree 030e10ff1a3 Rename pg_constraint.conwithoutoverlaps to conperiod a88c800deb6 Use daterange and YMD in without_overlaps tests instead of tsrange. 5577a71fb0c Use half-open interval notation in without_overlaps tests 34768ee3616 Add temporal FOREIGN KEY contraints 482e108cd38 Add test for REPLICA IDENTITY with a temporal key c3db1f30cba doc: clarify PERIOD and WITHOUT OVERLAPS in CREATE TABLE 144c2ce0cc7 Fix ON CONFLICT DO NOTHING/UPDATE for temporal indexes
- **Reason (quote):** "This feature set did not handle empty ranges correctly, and it's now too late for PostgreSQL 17 to fix it."
- **What is different this time:** P1 reverted temporal keys because they mishandled empty ranges. This patch closes a remaining gap of that kind: a temporal foreign key could reference a plain EXCLUDE constraint, which does not reject empty ranges.


## 3. Objections a reviewer is likely to raise

### 1. Back-patched error breaks dump and restore within the 18.x series

fixable · level 2, rubric item 13 · pg_dump / restore round-trip, backpatchability · `standards-compat-1`

A user on 18.1 has a temporal foreign key on a plain exclusion constraint and dumps the database. They restore it into 18.3, which has the fix. pg_restore reports an error for that constraint and continues, so the table loads without the foreign key, and the user may notice only if they read the error output.

Patch 0001 is marked Backpatch-through: 18, and its commit message says restoring such keys fails. The usual expectation is that a dump taken on a minor release restores on a later minor release of the same branch. Patch 0002 adds a warning on 18, but the warning appears only on the source server and does not prevent the restore failure. A committer would likely ask whether 18 should reject only new definitions and keep accepting existing ones, for example during restore, while 19 and later reject them outright.

Thread: [#1 (Paul A Jungwirth, 2026-09-30)](https://www.postgresql.org/message-id/CA%2BrenyUy%3D%2BoZoOg0qN_wAHTL3MwVEnp4zdfX%2BBu54foz6OXpQw%40mail.gmail.com)

**Question:** Why should 18.x reject these definitions at restore time instead of warning, given that a dump from an earlier 18.x release then loses the foreign key?

Counter-check: held. Message #1 and both commit messages confirm that restoring such a key fails on a fixed 18.x, and only pg_upgrade gets a check. The proposal does not weigh the alternative of rejecting these keys only from 19 on.

### 2. Warning code is limited to 18, but 19.0 may already allow these keys

fixable · level 2, rubric item 13 · backpatchability, pg_dump / restore round-trip · `standards-compat-2`

If 19.0 ships before the fix, a user on 19.0 can create such a key. After a minor update to 19.1, the server neither warns nor checks. The key keeps allowing serialization anomalies, and the user finds out only at the next dump or major upgrade.

Message #1 limits the warning to 18 because the fix is expected to land before 19 is released. The CommitFest entry targets 18, it is still in Needs review, and the review date is after the usual major release window. If 19.0 has shipped, the reasoning behind the 18-only split in patch 0002 no longer holds, and the 19 branch would need the warning as well.

Thread: [#1 (Paul A Jungwirth, 2026-09-30)](https://www.postgresql.org/message-id/CA%2BrenyUy%3D%2BoZoOg0qN_wAHTL3MwVEnp4zdfX%2BBu54foz6OXpQw%40mail.gmail.com)

**Question:** If 19.0 ships without the fix, will patch 0002 also go to REL_19_STABLE, or what else warns users with keys created on 19.0?

Counter-check: held. The material does not say whether 19.0 has shipped, so the scenario is conditional. Patch 0002 states that later branches see such keys only from 18.x, and that stops being true if 19.0 shipped without the fix.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | Patch 0001 adds a pg_upgrade check in check.c for temporal foreign keys that reference non-unique indexes. |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | **yes** | The patch narrows which constraints a foreign key with PERIOD may reference; no new syntax. |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | **yes** | A dump from 18.x that contains such a foreign key fails to restore that constraint on fixed versions. |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | Patch 0001 is marked Backpatch-through: 18, and patch 0002 is for REL_18_STABLE only. |
| maintenance burden ("who maintains this in 10 years") | **yes** | The 18 branch carries warning code in ri_triggers.c and tablecmds.c, and pg_upgrade carries a new version-keyed check. |
| performance on the hot path | **yes** | Patch 0002 adds a check when the server builds an RI plan in ri_triggers.c, on 18 only. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Why should 18.x reject these foreign keys at restore time, given that a dump from an earlier 18.x release then restores without the foreign key?
2. Did you consider having 18.x reject only new definitions while still accepting them during restore or binary upgrade, with 19 and later rejecting them outright?
3. If 19.0 ships without the fix, will patch 0002 also go to REL_19_STABLE, or what else warns users with keys created on 19.0?
4. Could pg_dump warn about or flag such a foreign key, so that the user learns of the problem before the restore fails?
5. How often does the 18.x warning fire for one affected foreign key: once per RI plan build in each backend, or less often?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Needs a committer who knows the temporal key and RI trigger code and back-patching policy. Peter Eisentraut is listed as reviewer in the CommitFest entry but has not posted in the visible thread.
- **Liability:** The owner signs for a behaviour change in an 18.x minor release: definitions that 18.x accepted now fail, including at restore. The owner also carries the 18-only warning code until 18 reaches end of life, and the pg_upgrade check for longer.
- **Author follow-up:** The author split the 18-only warning into its own commit, describes the merge conflicts per branch, and offers to add the item to the 19 Open Items list.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** Few users likely have such foreign keys, but those who do have referential integrity that is not reliably enforced. The two patches are small and touch tablecmds.c, ri_triggers.c and pg_upgrade's check.c; the open points are back-branch policy, which a committer can settle in review.

## Appendix

### Refuted in the counter-check

- `replication-ondisk-1` Backpatched error makes 18.x dumps unrestorable on later 18.x minors. Refuted (not specific): Duplicate of standards-compat-1. The logical replication case is the same failed ADD CONSTRAINT during a schema restore.
- `maintenance-fit-1` Warning on RI plan build repeats in every backend in 18.x. Refuted (no failure scenario): The warning reaches only sites that have a foreign key that cannot reliably enforce referential integrity, and it stops once they replace the key. Rubric item 8 concerns cost for non-users, who pay nothing here; warning frequency is an implementation detail.

### Review angles

- prior-attempts: 0 objection(s). No listed precedent rejected this fix or one like it. [P1] shows that the empty-range problem has already caused a revert, and this patch narrows that problem further. [P14] was held back for unrelated reasons (btree_gist in core, index ownership of the constraint), and neither reason applies to this fix.
- standards-compat: 2 objection(s). The restriction itself fits the standard: temporal foreign keys there reference keys declared WITHOUT OVERLAPS. The pg_upgrade check follows existing practice.
- replication-ondisk: 1 objection(s). The patch does not change WAL, the on-disk format or the catalog layout. My only concern is the dump and restore path for catalog state that already exists in 18.x clusters.
- security: 0 objection(s). The proposal adds no privilege or exposure surface: it rejects more foreign key definitions, adds a pg_upgrade check, and adds an 18.x warning naming constraints that pg_constraint already exposes. The warning fires only at RI plan build or partition clone, and RI plans are cached per backend, so an unprivileged user cannot use it to flood the log beyond normal connection logging.
- maintenance-fit: 1 objection(s). From the maintenance angle this is a narrow bugfix. It tightens an existing check, and the pg_upgrade check follows existing practice for version-keyed checks. The only objection is the frequency of the back-branch warning.
- already-exists: 0 objection(s). This is a bug fix that closes a gap in constraint validation. No existing feature, extension or setting stops a temporal foreign key from referencing a plain exclusion constraint, and only core code can enforce that rule. The pg_upgrade check follows existing practice, so this angle finds no duplicated mechanism and nothing that belongs outside core.
- demand: 0 objection(s). This is a bug fix, so demand is not the test. A committer reported a correctness hole (serialization anomalies through temporal foreign keys on plain EXCLUDE constraints), and the restriction matches what WITHOUT OVERLAPS keys already guarantee. The pg_upgrade check and the 18.x warning fire only for affected foreign keys, so their cost falls on the users they protect.

### Precedents retrieved

- P1: [Revert temporal primary keys and foreign keys](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=8aee330af55d8a759b2b73f5a771d9d34a7b887f) (revert, Reverted, 2024-05-16) - used in the review
- P2: [Foreign Key Arrays](https://commitfest.postgresql.org/patch/2966/) (cf, Returned with feedback, 2021-09-14) - judged not relevant
- P3: [Foreign Key Arrays](https://commitfest.postgresql.org/patch/1252/) (cf, Returned with feedback, 2018-10-02) - judged not relevant
- P4: [Temporal foreign key actions](https://commitfest.postgresql.org/patch/7238/) (cf, Withdrawn, 2026-09-17) - judged not relevant; reason summarised from its thread
- P5: [Allow foreign keys to reference a superset of unique columns](https://commitfest.postgresql.org/patch/3678/) (cf, Returned with feedback, 2022-10-12) - judged not relevant; reason summarised from its thread
- P6: [foreign key on virtual generated column](https://commitfest.postgresql.org/patch/5782/) (cf, Withdrawn, 2026-09-23) - judged not relevant; reason summarised from its thread
- P7: [Revert "Use Foreign Key relationships to infer multi-column join selectivity".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=77ba610805e7ef9ba9c9a593ea8b1ca8f98f8bcb) (revert, Reverted, 2016-06-07) - judged not relevant
- P8: [Revert "Fix an O(N^2) problem in foreign key references".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=3d9e8db9e56beb0501d168779ae9644587924061) (revert, Reverted, 2015-09-15) - judged not relevant
- P9: [pg_get_constraintdef: Schema qualify foreign tables unless pretty printing is enabled](https://commitfest.postgresql.org/patch/3811/) (cf, Returned with feedback, 2022-11-30) - judged not relevant; reason summarised from its thread
- P10: [Fix permission tests for views/tables proven empty by constraint exclusion.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=50c137487c96e629e0e5372bb3d1b5f1a2f71a88) (revert, Partially reverted, 2013-05-01) - judged not relevant
- P11: [Reducing memory usage of creating foreign key between partitioned tables](https://commitfest.postgresql.org/patch/2330/) (cf, Rejected, 2020-09-12) - judged not relevant; reason summarised from its thread
- P12: [Add pg_upgrade check for invalid role references in pg_init_privs](https://commitfest.postgresql.org/patch/6856/) (cf, Withdrawn, 2026-06-17) - judged not relevant; reason summarised from its thread
- P13: [pg_upgrade: Support for upgrading to checksums enabled](https://commitfest.postgresql.org/patch/5206/) (cf, Returned with feedback, 2025-04-03) - judged not relevant; reason summarised from its thread
- P14: [SQL:2011 application time](https://commitfest.postgresql.org/patch/2048/) (cf, Returned with feedback, 2022-02-07) - used in the review; reason summarised from its thread
- P15: [pg_upgrade data type check connection overhead reduction](https://commitfest.postgresql.org/patch/4200/) (cf, Returned with feedback, 2024-02-01) - judged not relevant
- P16: [pg_upgrade version and path checking](https://commitfest.postgresql.org/patch/2095/) (cf, Returned with feedback, 2019-09-30) - judged not relevant
- P17: [pg_upgrade check for Unicode update](https://commitfest.postgresql.org/patch/5563/) (cf, Withdrawn, 2025-03-18) - judged not relevant
- P18: [Improve amcheck to check UNIQUE constraint in btree index](https://commitfest.postgresql.org/patch/2976/) (cf, Returned with feedback, 2021-04-08) - judged not relevant
- P19: [Revert "Add basic TAP test setup for pg_upgrade"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=58ffe141eb37c3f027acd25c1fc6b36513bf9380) (revert, Reverted, 2017-09-22) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 13; wall time: 97 s; cost reported by the CLI: $1.06
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 2 of 19 kept
- Tone check: 68 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:58:02.465Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

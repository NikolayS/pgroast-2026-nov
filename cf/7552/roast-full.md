# Design pre-review: Setting the number of digits after the decimal point with a psql's pset variable

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7552](https://commitfest.postgresql.org/patch/7552/) (Needs review); authors: Guillaume Lelarge (gleu)
- Thread: [Setting the number of digits after the decimal point with a psql's pset variable](https://www.postgresql.org/message-id/flat/a1db875a-15dc-4a8a-a1b3-1f32b020b70a%40dalibo.com), started 2026-10-29
- Messages read: 7 of 7
- Patch prose read: 0001-Add-new-n_decimals-pset-variable.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The goal has support in the thread, and 645cb44c5 is cited as existing practice, so level 1b does not stop the proposal. The design is not settled. The commit message says "rounds" while the code truncates (standards-compat-1), no output format is excluded (standards-compat-2), and exponent notation, NaN and Infinity have no rule (maintenance-fit-2). The list needs to agree on these semantics before code review is useful.

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so this review cannot say which output formats the print.c change reaches or how it handles numericlocale. The heap overflow and memory leaks reported in the thread are implementation matters and were not judged.

## The proposal, restated

**Problem.** Ratios and percentages in psql output show more fractional digits than a reader needs. Wrapping each column in round() is slow and error prone in large queries. The author wants a quick display setting for interactive users, for example when reading metrics at a customer site or formatting tables for the mailing lists.

**Proposed design.** psql gets a new \pset variable, n_decimals, that cuts float4, float8 and numeric columns to a fixed number of fractional digits at display time. The default "all" keeps current behaviour, and a negative value means "all". The change sits in the fe_utils print code, so it touches only client output, not query results.

**User-visible surface.**
- New psql \pset option n_decimals, accepting "all" or an integer.
- New status message: Number of decimals is "N".
- Changed output of float4, float8 and numeric columns in psql when n_decimals is set.
- New fields in fe_utils print options (src/include/fe_utils/print.h), visible to other frontend tools using that API.

**What the author claims.**
- Default "all" leaves psql behaviour unchanged.
- Commit message says values are rounded, but the author confirms in message #6 that the code truncates and he prefers it.
- \pset is the right place for this kind of feature.
- Nathan Bossart wants the same feature, and Bertrand Drouvot cites 645cb44c5 as a precedent.

**Thread state.** Nathan Bossart supports the idea, and Bertrand Drouvot accepts it after citing a precedent, then reports a heap overflow, truncation, wrong output for 1e100, memory leaks and lax input parsing. The author fixed parsing, says he probably fixed the overflow, and promises a new patch; none is attached yet.

## 2. Prior attempts

### psql: Add \pset true/false (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/406/](https://commitfest.postgresql.org/patch/406/)
- **Thread:** [https://www.postgresql.org/message-id/flat/56308F56.8060908@joh.to](https://www.postgresql.org/message-id/flat/56308F56.8060908@joh.to)
- **Date:** decided 2015-12-10; started 2015-10-28
- **Outcome:** Rejected (cf)
- **Who objected:** Michael Paquier (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Reviewers held that choosing display values for one data type is not a job for psql, because the user can already format values in the query. Robert Haas also did not want similar options added for other data types later. The final tally showed more -1 than +1 votes. Discussion of a more general output filter, either client-side or in the backend, reached no design, so Michael Paquier marked the patch rejected.
- **Reason (quote):** "FWIW, I am -1 on the concept of enforcing output values for particular datatypes because that's not the job of psql, and it is easy to do that at query level (no comment about the existing \pset NULL stuff)." ([message](https://www.postgresql.org/message-id/CAB7nPqQMOOLpZCGsuxUo9wK0nSRybC%3DhVBCz77Tt2mwtx9oytg%40mail.gmail.com))
- **What is different this time:** P5 set display strings for booleans; this proposal limits fractional digits of float4, float8 and numeric. Messages #2 and #5 cite 645cb44c5 as having since opened the door to per-type display options, and this proposal has a +1 in message #3.


## 3. Objections a reviewer is likely to raise

### 1. Truncation makes psql output disagree with round() and with stored values

fixable · level 2, rubric item 12 · psql UX conventions, SQL-standard conflict or non-standard syntax · `standards-compat-1`

A DBA sets n_decimals 2 and sees 9.99 for a value of 9.999. round(9.999, 2) in the same session returns 10.00, so the screen and the query disagree. A hit ratio of 99.996 shows as 99.99 and looks short of a 100% target that it nearly meets.

Message #6 confirms the code truncates and the author prefers that. The commit message says "rounds", and the docs say neither. A setting that changes how numbers are shown should match the rounding users already get from round() and from numeric-to-text casts, or say plainly that it does not. The docs hunk also has no warning that shown values differ from stored values.

Thread: [#5 (Bertrand Drouvot, 2026-11-02)](https://www.postgresql.org/message-id/auhYSNhQ47Q8DdSJ%40bdtpg), [#6 (Guillaume Lelarge, 2026-11-02)](https://www.postgresql.org/message-id/be6091c8-5995-4635-a78f-4043aa782f06%40dalibo.com)

**Question:** Will n_decimals round like round(), and if it truncates, how do the docs warn users that the shown value is not the stored value?

Counter-check: held. Message #6 confirms truncation, the commit message says "rounds", and the docs hunk says neither. Message #5 expected round() behaviour, so the disagreement is real, and a new patch version can settle it.

### 2. Proposal does not say which output formats n_decimals affects

fixable · level 2, rubric item 13 · psql UX conventions · `standards-compat-2`

A user puts \pset n_decimals 2 in .psqlrc for interactive work. Later a cron job runs psql --csv or -A -t into a file that another tool loads. The exported numbers are cut to two digits with no error, and nobody sees it until reports disagree with the database.

The change sits in the shared print path used by aligned, unaligned, CSV, HTML and other formats, and by \o. The proposal does not say whether machine-readable formats are excluded, or how it treats \gset, \crosstabview and expanded mode. numericlocale has the same reach but only adds separators and drops no digits. A setting that loses data needs a clearer boundary than one that only changes layout.

Thread: [#1 (Guillaume Lelarge, 2026-10-29)](https://www.postgresql.org/message-id/a1db875a-15dc-4a8a-a1b3-1f32b020b70a%40dalibo.com)

**Question:** Does n_decimals apply to csv and unaligned output and to \gset, and if it does, how is a .psqlrc setting kept from changing scripted exports?

Counter-check: held. The proposal and the docs hunk name no output format, and the change sits in the shared fe_utils print code. \gset probably does not use that code, but the csv and unaligned case with a .psqlrc setting is a concrete failure.

### 3. Client re-parses the server's text output for float and numeric

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years"), ABI / extension API, psql UX conventions · `maintenance-fit-2`

A user sets n_decimals 0 and runs a query that returns 1e100::float8, NaN, or Infinity. psql prints 1 for 1e100, as message #5 shows. Users of domains over numeric, or of extension numeric types, see no effect, because the code checks only three built-in OIDs.

The design edits strings in fe_utils by hand, based on assumptions about how the server writes float4, float8 and numeric: where the decimal point sits, exponent notation, special values, and how numericlocale changes the string. Any later change to those output functions, or to extra_float_digits behaviour, must also update print.c, and no test fails if that update is missed. The new fields in print.h become API that other frontend tools inherit.

Thread: [#5 (Bertrand Drouvot, 2026-11-02)](https://www.postgresql.org/message-id/auhYSNhQ47Q8DdSJ%40bdtpg), [#6 (Guillaume Lelarge, 2026-11-02)](https://www.postgresql.org/message-id/be6091c8-5995-4635-a78f-4043aa782f06%40dalibo.com)

**Question:** Which output forms of float4, float8 and numeric will the formatter handle, and what keeps it in step when server output functions change?

Counter-check: held. Message #5 shows 1e100 printed as 1, and the author agrees in message #6 that it needs work. The domain example is weak, because the server reports the base type OID for a domain column, but the string-editing concern stands.

### 4. Per-type display option repeats the case rejected for \pset true/false

fixable · level 1b, rubric item 5 · prior rejection, psql UX conventions, maintenance burden ("who maintains this in 10 years"), doesn't fit what Postgres is about (extension / external tool / already solved) · `maintenance-fit-1`

After n_decimals goes in, the next patches ask for date formats, interval styles, and boolean strings in \pset, each citing this one. Each adds a \pset option, a status message, tab completion, and print.h fields. Reviewers have no principle left for saying no.

[P5] was rejected because picking display values for one data type was held not to be psql's job when the query can do it, and because of worry about follow-on options for other types. Message #2 raised the same worry, then set it aside by citing 645cb44c5. That makes two precedents in favour, and the series gets easier to extend each time. A committer would likely ask for a general rule, or a general formatting hook, before adding another type-specific knob.

**Precedents.**
- [psql: Add \pset true/false](https://commitfest.postgresql.org/patch/406/) (Rejected, 2015-12-10); thread: [link](https://www.postgresql.org/message-id/flat/56308F56.8060908@joh.to)

Thread: [#2 (Bertrand Drouvot, 2026-10-30)](https://www.postgresql.org/message-id/auRYP9odWlyr3yKM%40bdtpg), [#3 (Nathan Bossart, 2026-10-30)](https://www.postgresql.org/message-id/auT0N7oFol0JOVtv%40nathan), [#5 (Bertrand Drouvot, 2026-11-02)](https://www.postgresql.org/message-id/auhYSNhQ47Q8DdSJ%40bdtpg)

**Question:** What rule would decide which later per-type display options belong in \pset, and why does n_decimals fit it while \pset true/false did not?

Counter-check: held. Message #2 raised the follow-on worry and the thread answered it only by citing a commit, with no rule for later per-type options. [P5] must not be used as support, because psql has since accepted boolean display options.

### 5. Demand rests on two users, and round() already covers the need

fixable · level 1b, rubric item 3 · demand, doesn't fit what Postgres is about (extension / external tool / already solved), maintenance burden ("who maintains this in 10 years") · `demand-1`

psql takes on a permanent \pset option, new fe_utils print fields, docs and tests for numeric display formatting. The only users shown are two developers who would otherwise add round() to their queries. Later requests for similar per-type display options cite this option as precedent.

The evidence of demand is the author's own consulting workflow (messages #1, #4) and one +1 from a developer who wanted it "a handful of times" (message #3). The thread cites no -general or -bugs reports and names no client tool that offers this. [P5] rejected per-type display values in psql because "it is easy to do that at query level", and round() is that query-level way here. Message #5 cites 645cb44c5 as precedent, but that shows only that the door is open, not that users want this.

**Precedents.**
- [psql: Add \pset true/false](https://commitfest.postgresql.org/patch/406/) (Rejected, 2015-12-10); thread: [link](https://www.postgresql.org/message-id/flat/56308F56.8060908@joh.to)

Thread: [#1 (Guillaume Lelarge, 2026-10-29)](https://www.postgresql.org/message-id/a1db875a-15dc-4a8a-a1b3-1f32b020b70a%40dalibo.com), [#3 (Nathan Bossart, 2026-10-30)](https://www.postgresql.org/message-id/auT0N7oFol0JOVtv%40nathan), [#4 (Guillaume Lelarge, 2026-10-30)](https://www.postgresql.org/message-id/232d4e56-4d88-40fb-8d1d-dda7994ead3c%40dalibo.com), [#5 (Bertrand Drouvot, 2026-11-02)](https://www.postgresql.org/message-id/auhYSNhQ47Q8DdSJ%40bdtpg)

**Question:** Which users beyond this thread ask for display-time decimal limits, and why does round() or a saved psql variable holding a query not serve them?

Counter-check: held. The thread shows two independent users, and round() can do the job, so the material does not show a common need. That is a "show demand" point, not a rejection, and [P5] must not be used as support.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | The patch adds fields to the print options in src/include/fe_utils/print.h, which other frontend tools use. |
| SQL-standard conflict or non-standard syntax | **yes** | see objection standards-compat-1: Truncation makes psql output disagree with round() and with stored values |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | **yes** | The patch adds \pset n_decimals, a status message, help text and tab completion, and changes how float4, float8 and numeric are shown. |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | print.c edits the server's text output for float4, float8 and numeric, so it must follow any change to those output functions. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | Message #1 says round() in the query already gives the result; the option adds convenience for large queries. |

## 5. Questions the author must answer before anyone reviews code

1. Will n_decimals round as round() does or truncate, and will the commit message, docs and code say the same thing?
2. Does n_decimals apply to csv, unaligned and \o output, and how is a .psqlrc setting kept from changing scripted exports?
3. What does n_decimals show for exponent notation such as 1e100, for NaN and for Infinity?
4. How does n_decimals interact with numericlocale, and does it add trailing zeros so that 1 shows as 1.00?
5. What rule decides which later per-type display options belong in \pset, so that n_decimals is not the argument for each one?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. A committer who knows psql and the fe_utils print code would have to own it. The visible thread shows support for the idea but nobody offering to commit it.
- **Liability:** The owner signs for a permanent \pset option and for client code that edits the server's float and numeric text output. That code needs an update whenever those output functions change.
- **Author follow-up:** The author answered the review, fixed input parsing and promised a new patch version; none is attached yet.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** niche
- **Maintenance burden:** medium
- **Review cost:** low
- **Why:** The patch is about 9 KiB and stays in psql and fe_utils, and the thread shows two interactive users who want it. Rounding, the output formats covered and the special values need a decision before another review round.

## Appendix

### Refuted in the counter-check

- `prior-attempts-1` Per-type display formatting in psql was rejected before for the same reason. Refuted (outdated reason): Messages #2 and #5 say 645cb44c5 already opened the door to per-type options, and psql now has \pset display_true and display_false. So the [P5] outcome no longer holds. The remaining follow-on worry is covered by maintenance-fit-1.
- `prior-attempts-2` Display semantics split reviewers, as in the earlier JSON output attempt. Refuted (not specific): Duplicate of standards-compat-1, which states the truncation and rounding conflict more directly. [P2] concerned a JSON output format and round-trip safety, so it adds no support here.
- `standards-compat-3` Per-type display formatting was rejected before, and corner cases are not decided. Refuted (not specific): Duplicate of maintenance-fit-1 for the per-type part, and of standards-compat-1 and maintenance-fit-2 for the corner cases. It adds no new point.
- `already-exists-1` Query-level round() or a numeric cast already gives this result. Refuted (not specific): Duplicate of demand-1 for the round() argument and of maintenance-fit-1 for the follow-on options. Its [P5] support is outdated.

### Review angles

- prior-attempts: 2 objection(s). Two objections are raised, both rooted in precedents whose reasons the thread has not yet answered.
- standards-compat: 3 objection(s). Objections found.
- replication-ondisk: 0 objection(s). The proposal changes only how psql displays float4, float8 and numeric values, so it creates no on-disk, WAL, catalog, replication or wire-protocol state. The new fields in fe_utils print.h are in a static frontend library, not a server ABI that extensions link against, so they need no backpatch or ABI care.
- security: 0 objection(s). The proposal has no privilege or exposure aspect. n_decimals is a psql display setting that changes only how the client prints values it has already received. It grants no new rights and shows no data the user could not already see. The heap overflow reported in message #5 is an implementation bug, which is out of scope for this review.
- maintenance-fit: 2 objection(s). Hot-path cost looks negligible when the option is unset. Nathan Bossart supports the idea, but no committer has said he would own the print.c changes.
- already-exists: 1 objection(s). One objection only. extra_float_digits covers float4 and float8 but not numeric, so it does not meet the need in message #1.
- demand: 1 objection(s). One objection only. Message #2 raised the risk of more per-type options, and message #5 settled it by pointing to precedent, so this review does not repeat that point.

### Precedents retrieved

- P1: [psql markdown output format](https://commitfest.postgresql.org/patch/1056/) (cf, Returned with feedback, 2017-11-28) - judged not relevant; reason summarised from its thread
- P2: [psql JSON output format](https://commitfest.postgresql.org/patch/4707/) (cf, Returned with feedback, 2024-05-15) - used in the review; reason summarised from its thread
- P3: [csv output format for psql](https://commitfest.postgresql.org/patch/1500/) (cf, Returned with feedback, 2018-10-01) - judged not relevant; reason summarised from its thread
- P4: [make psql backslash commands honor "\pset null"](https://commitfest.postgresql.org/patch/4603/) (cf, Withdrawn, 2023-10-06) - judged not relevant; reason summarised from its thread
- P5: [psql: Add \pset true/false](https://commitfest.postgresql.org/patch/406/) (cf, Rejected, 2015-12-10) - used in the review; reason summarised from its thread
- P6: [psql - possibility to specify where status row should be displayed](https://commitfest.postgresql.org/patch/2536/) (cf, Rejected, 2021-04-08) - judged not relevant; reason summarised from its thread
- P7: [psql \graw execution and output format](https://commitfest.postgresql.org/patch/1254/) (cf, Returned with feedback, 2017-09-07) - judged not relevant; reason summarised from its thread
- P8: [Precision loss casting float to numeric](https://commitfest.postgresql.org/patch/1541/) (cf, Returned with feedback, 2018-03-17) - judged not relevant; reason summarised from its thread
- P9: [PoC - psql - emphases line with table name in verbose output](https://commitfest.postgresql.org/patch/6620/) (cf, Withdrawn, 2026-08-06) - judged not relevant
- P10: [\crosstabview (previously: \rotate) in psql for crosstab-style display](https://commitfest.postgresql.org/patch/372/) (cf, Returned with feedback, 2016-02-08) - judged not relevant
- P11: [Using multicolumn statistics for more precise join selectivity estimatation](https://commitfest.postgresql.org/patch/2386/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P12: [psql \setfileref - new psql variable type - reference to file](https://commitfest.postgresql.org/patch/787/) (cf, Returned with feedback, 2016-12-15) - judged not relevant
- P13: [Revert psql bits to display NOT VALID for FKs](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=135a507991c8ea9eca201cc084294a689ca24ba9) (revert, Reverted, 2011-06-08) - judged not relevant
- P14: [Fix to_number() handling for values not matching the format string](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Character_Formatting) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P15: [Adjust rounding behavior for numeric GUC values](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Configuration_files) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P16: [Revert support for improved tracking of nested queries](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f85f6ab051b7cf6950247e5fa6072c4130613555) (revert, Reverted, 2025-06-12) - judged not relevant
- P17: [Throw an error from to_char() instead of printing a string of "#" when a number doesn't fit in the desired output format.](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Character_Formatting) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P18: [IF (NOT) EXISTS in psql-completion](https://commitfest.postgresql.org/patch/518/) (cf, Returned with feedback, 2016-09-28) - judged not relevant
- P19: [Add and report the new "session_read_only" GUC pseudo-variable.](https://commitfest.postgresql.org/patch/1090/) (cf, Returned with feedback, 2019-02-04) - judged not relevant
- P20: [Revert "Test conversion of NaN between float4 and float8."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=5c4c771dafefa7d0ee80a6bd6955b7a3940452a1) (revert, Reverted, 2018-05-05) - judged not relevant
- P21: [possibility to specify sort columns for psql describe commands in verbose mode](https://commitfest.postgresql.org/patch/1032/) (cf, Rejected, 2018-01-12) - judged not relevant
- P22: [ensure same width of last column](https://commitfest.postgresql.org/patch/2281/) (cf, Withdrawn, 2019-11-12) - judged not relevant
- P23: [format_datum debugging function](https://commitfest.postgresql.org/patch/5165/) (cf, Returned with feedback, 2024-10-07) - judged not relevant
- P24: [Fix formatting of Interval output](https://commitfest.postgresql.org/patch/3548/) (cf, Withdrawn, 2022-02-19) - judged not relevant

### Warnings

- review angle "standards-compat" returned prior attempts; ignored (only the prior-attempts angle may)

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 16; wall time: 153 s; cost reported by the CLI: $1.61
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 5 from cache, 3 summarised now, 0 failed; quotes: 3 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 93 text field(s) checked, 1 finding(s), 1 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T04:09:02.741Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

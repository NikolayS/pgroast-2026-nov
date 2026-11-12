# Design pre-review: Setting the number of digits after the decimal point with a psql's pset variable

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7552](https://commitfest.postgresql.org/patch/7552/); [thread](https://www.postgresql.org/message-id/flat/a1db875a-15dc-4a8a-a1b3-1f32b020b70a%40dalibo.com), 7 of 7 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The goal has support in the thread, and 645cb44c5 is cited as existing practice, so level 1b does not stop the proposal. The design is not settled. The commit message says "rounds" while the code truncates (standards-compat-1), no output format is excluded (standards-compat-2), and exponent notation, NaN and Infinity have no rule (maintenance-fit-2). The list needs to agree on these semantics before code review is useful.

Not checked: The code was not read, so this review cannot say which output formats the print.c change reaches or how it handles numericlocale. The heap overflow and memory leaks reported in the thread are implementation matters and were not judged.

## Prior attempts

- [psql: Add \pset true/false](https://www.postgresql.org/message-id/flat/56308F56.8060908@joh.to) (similar idea; Rejected, 2015-12-10). "FWIW, I am -1 on the concept of enforcing output values for particular datatypes because that's not the job of psql, and it is easy to do that at query level (no comment about the existing \pset NULL stuff)." ([message](https://www.postgresql.org/message-id/CAB7nPqQMOOLpZCGsuxUo9wK0nSRybC%3DhVBCz77Tt2mwtx9oytg%40mail.gmail.com)) Different now: P5 set display strings for booleans; this proposal limits fractional digits of float4, float8 and numeric. Messages #2 and #5 cite 645cb44c5 as having since opened the door to per-type display options, and this proposal has a +1 in message #3.

## Objections a reviewer is likely to raise

### 1. Truncation makes psql output disagree with round() and with stored values

A DBA sets n_decimals 2 and sees 9.99 for a value of 9.999. round(9.999, 2) in the same session returns 10.00, so the screen and the query disagree. A hit ratio of 99.996 shows as 99.99 and looks short of a 100% target that it nearly meets.

Message #6 confirms the code truncates and the author prefers that. The commit message says "rounds", and the docs say neither. A setting that changes how numbers are shown should match the rounding users already get from round() and from numeric-to-text casts, or say plainly that it does not. The docs hunk also has no warning that shown values differ from stored values.

**Question:** Will n_decimals round like round(), and if it truncates, how do the docs warn users that the shown value is not the stored value?

<sub>fixable, level 2, psql UX conventions; messages [#5](https://www.postgresql.org/message-id/auhYSNhQ47Q8DdSJ%40bdtpg), [#6](https://www.postgresql.org/message-id/be6091c8-5995-4635-a78f-4043aa782f06%40dalibo.com); `standards-compat-1`</sub>

### 2. Proposal does not say which output formats n_decimals affects

A user puts \pset n_decimals 2 in .psqlrc for interactive work. Later a cron job runs psql --csv or -A -t into a file that another tool loads. The exported numbers are cut to two digits with no error, and nobody sees it until reports disagree with the database.

The change sits in the shared print path used by aligned, unaligned, CSV, HTML and other formats, and by \o. The proposal does not say whether machine-readable formats are excluded, or how it treats \gset, \crosstabview and expanded mode. numericlocale has the same reach but only adds separators and drops no digits. A setting that loses data needs a clearer boundary than one that only changes layout.

**Question:** Does n_decimals apply to csv and unaligned output and to \gset, and if it does, how is a .psqlrc setting kept from changing scripted exports?

<sub>fixable, level 2, psql UX conventions; messages [#1](https://www.postgresql.org/message-id/a1db875a-15dc-4a8a-a1b3-1f32b020b70a%40dalibo.com); `standards-compat-2`</sub>

### 3. Client re-parses the server's text output for float and numeric

A user sets n_decimals 0 and runs a query that returns 1e100::float8, NaN, or Infinity. psql prints 1 for 1e100, as message #5 shows. Users of domains over numeric, or of extension numeric types, see no effect, because the code checks only three built-in OIDs.

The design edits strings in fe_utils by hand, based on assumptions about how the server writes float4, float8 and numeric: where the decimal point sits, exponent notation, special values, and how numericlocale changes the string. Any later change to those output functions, or to extra_float_digits behaviour, must also update print.c, and no test fails if that update is missed. The new fields in print.h become API that other frontend tools inherit.

**Question:** Which output forms of float4, float8 and numeric will the formatter handle, and what keeps it in step when server output functions change?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#5](https://www.postgresql.org/message-id/auhYSNhQ47Q8DdSJ%40bdtpg), [#6](https://www.postgresql.org/message-id/be6091c8-5995-4635-a78f-4043aa782f06%40dalibo.com); `maintenance-fit-2`</sub>

### 4. Per-type display option repeats the case rejected for \pset true/false

After n_decimals goes in, the next patches ask for date formats, interval styles, and boolean strings in \pset, each citing this one. Each adds a \pset option, a status message, tab completion, and print.h fields. Reviewers have no principle left for saying no.

[P5] was rejected because picking display values for one data type was held not to be psql's job when the query can do it, and because of worry about follow-on options for other types. Message #2 raised the same worry, then set it aside by citing 645cb44c5. That makes two precedents in favour, and the series gets easier to extend each time. A committer would likely ask for a general rule, or a general formatting hook, before adding another type-specific knob.

Precedent: [psql: Add \pset true/false](https://www.postgresql.org/message-id/flat/56308F56.8060908@joh.to) (Rejected, 2015-12-10)

**Question:** What rule would decide which later per-type display options belong in \pset, and why does n_decimals fit it while \pset true/false did not?

<sub>fixable, level 1b, prior rejection; messages [#2](https://www.postgresql.org/message-id/auRYP9odWlyr3yKM%40bdtpg), [#3](https://www.postgresql.org/message-id/auT0N7oFol0JOVtv%40nathan), [#5](https://www.postgresql.org/message-id/auhYSNhQ47Q8DdSJ%40bdtpg); `maintenance-fit-1`</sub>

### 5. Demand rests on two users, and round() already covers the need

psql takes on a permanent \pset option, new fe_utils print fields, docs and tests for numeric display formatting. The only users shown are two developers who would otherwise add round() to their queries. Later requests for similar per-type display options cite this option as precedent.

The evidence of demand is the author's own consulting workflow (messages #1, #4) and one +1 from a developer who wanted it "a handful of times" (message #3). The thread cites no -general or -bugs reports and names no client tool that offers this. [P5] rejected per-type display values in psql because "it is easy to do that at query level", and round() is that query-level way here. Message #5 cites 645cb44c5 as precedent, but that shows only that the door is open, not that users want this.

Precedent: [psql: Add \pset true/false](https://www.postgresql.org/message-id/flat/56308F56.8060908@joh.to) (Rejected, 2015-12-10)

**Question:** Which users beyond this thread ask for display-time decimal limits, and why does round() or a saved psql variable holding a query not serve them?

<sub>fixable, level 1b, demand; messages [#1](https://www.postgresql.org/message-id/a1db875a-15dc-4a8a-a1b3-1f32b020b70a%40dalibo.com), [#3](https://www.postgresql.org/message-id/auT0N7oFol0JOVtv%40nathan), [#4](https://www.postgresql.org/message-id/232d4e56-4d88-40fb-8d1d-dda7994ead3c%40dalibo.com), [#5](https://www.postgresql.org/message-id/auhYSNhQ47Q8DdSJ%40bdtpg); `demand-1`</sub>

## Questions for the author

1. Will n_decimals round as round() does or truncate, and will the commit message, docs and code say the same thing?
2. Does n_decimals apply to csv, unaligned and \o output, and how is a .psqlrc setting kept from changing scripted exports?
3. What does n_decimals show for exponent notation such as 1e100, for NaN and for Infinity?
4. How does n_decimals interact with numericlocale, and does it add trailing zeros so that 1 shows as 1.00?
5. What rule decides which later per-type display options belong in \pset, so that n_decimals is not the argument for each one?

## Touch points

- **ABI / extension API:** The patch adds fields to the print options in src/include/fe_utils/print.h, which other frontend tools use.
- **SQL-standard conflict or non-standard syntax:** see objection standards-compat-1: Truncation makes psql output disagree with round() and with stored values
- **psql UX conventions:** The patch adds \pset n_decimals, a status message, help text and tab completion, and changes how float4, float8 and numeric are shown.
- **maintenance burden ("who maintains this in 10 years"):** print.c edits the server's text output for float4, float8 and numeric, so it must follow any change to those output functions.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** Message #1 says round() in the query already gives the result; the option adds convenience for large queries.

## Who would commit it, and when

- Owner: silent. A committer who knows psql and the fe_utils print code would have to own it. The visible thread shows support for the idea but nobody offering to commit it.
- Priority: fix design first; audience niche; review cost low. The patch is about 9 KiB and stays in psql and fe_utils, and the thread shows two interactive users who want it. Rounding, the output formats covered and the special values need a decision before another review round.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 4 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

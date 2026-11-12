# Design pre-review: Implement row pattern recognition feature

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 4460](https://commitfest.postgresql.org/patch/4460/) (Needs review); authors: Tatsuo Ishii (ishii), Jian He (stayreal), Henson Choi (assam258.gmail.com)
- Thread: [Row pattern recognition](https://www.postgresql.org/message-id/flat/20230625.210509.1276733411677577841.t-ishii%40sranhm.sra.co.jp), started 2023-06-25
- Messages read: 549 of 549
- Patch prose read: v20-0001-Row-pattern-recognition-patch-for-raw-parser.patch, v20-0002-Row-pattern-recognition-patch-parse-analysis.patch, v20-0003-Row-pattern-recognition-patch-rewriter.patch, v20-0004-Row-pattern-recognition-patch-planner.patch, v20-0005-Row-pattern-recognition-patch-executor.patch, v20-0006-Row-pattern-recognition-patch-docs.patch, v20-0007-Row-pattern-recognition-patch-tests.patch, v20-0008-Allow-to-print-raw-parse-tree.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No surviving objection is fatal, and none fails level 1a or 1b: the problem is stated and demand is shown. The surviving level 2 objections are fixable design points: the open frame redesign (maintenance-fit-3), no stated NFA memory bound (security-1), DEFINE deparse round-trip (replication-ondisk-1) and the planner burden of defineClause (maintenance-fit-1). They need a committer who knows the planner, ruleutils.c and nodeWindowAgg.c to judge them.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Only v20 patch prose and the first three messages were read, while the current version is v54. NFA memory behaviour, the frame code and the planner handling of defineClause could not be checked without the code.

## The proposal, restated

**Problem.** Searching an ordered series of rows for a regular-expression-like shape, such as a V-shaped price curve, is complex and slow to write in Postgres today. The author targets analytic users and cites Oracle, Snowflake, Trino and others that offer row pattern recognition.

**Proposed design.** The patch adds the SQL:2016 R020 row pattern syntax to a window definition: PATTERN, DEFINE, INITIAL and AFTER MATCH SKIP. The pattern match narrows each row's full window frame to a reduced frame, so existing window functions and aggregates read only matched rows. An NFA in the executor does the matching, DEFINE is planned like havingQual, and navigation functions read other rows.

**User-visible surface.**
- Window clause syntax: AFTER MATCH SKIP PAST LAST ROW | TO NEXT ROW, INITIAL, PATTERN (...), DEFINE var AS expr
- PATTERN operators: +, *, ?, |, grouping, {n}, {n,}, {n,m}, {,m}, reluctant quantifiers
- Row pattern navigation functions PREV, NEXT, FIRST, LAST, with offsets and compound forms, allowed only in DEFINE
- New EXPLAIN output for RPR windows: Pattern line with absorption markers (~, #), NFA counters under ANALYZE
- pg_get_viewdef() output changes, including some queries without RPR (TABLEFUNC alias lists, column name reservation)

**What the author claims.**
- The design is a subset of R020 that can grow later without changing behaviour, and the default skip mode now follows the standard.
- Patch 0003 makes match results agree with the standard and Perl, and was cross-checked against Oracle 23ai and Trino 471.
- The DEFINE memory leak fix cuts memory from 94MB to 18MB on 300k rows.
- Tests find no remaining wrong answers apart from listed known issues; run condition pushdown and frame replacement are disabled for RPR windows.
- The large test suite is needed as black-box coverage, and its query count is in line with existing features.

**Thread state.** Tatsuo Ishii posted v54 on 2026-10-20, built from Henson Choi's ten patches; Henson Choi and jian he keep exchanging refactorings. Open items include a proposed redesign that makes the reduced frame the frame itself, the EXPLAIN marker format, test suite size, and whether R020 should be marked YES in sql_features.txt.

## 2. Prior attempts

No precedent found (corpus searched for items before 2023-06-25; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Reduced-frame design is still open and would change the window API

fixable · level 2, rubric item 11 · ABI / extension API, maintenance burden ("who maintains this in 10 years"), performance on the hot path · `maintenance-fit-3`

The current overlay design ships, and extension window functions start using it. The proposed redesign in message #547 then changes what frameheadpos and frametailpos mean, including that T may decrease and H may move backward. Those extensions, and the core readers that work out the end from frameOptions, need changes in a later release.

Message #543 and message #547 say the current reduced frame cannot express SEEK. They also say it restarts aggregates on every row, which is quadratic, and that it spools the partition for last_value(). Message #547 lists six frame contracts (C1-C6) that the readers rely on, and it notes that one change touches the contract of transition functions. A committer would likely ask that this contract be settled before WinGetFuncArgInFrame() and related code take on RPR-specific behaviour.

Thread: [#542 (jian he, 2026-10-21)](https://www.postgresql.org/message-id/CACJufxFEwTRc4JEbRoYVWPYE-KM6B%3D7qWYtg6SpsZHfA2U3fWw%40mail.gmail.com), [#543 (Henson Choi, 2026-10-22)](https://www.postgresql.org/message-id/CAAAe_zCqiC5iqoYq5uaTWUCpsUsv-7pOjScwfm66wu28OYWOZA%40mail.gmail.com), [#547 (Henson Choi, 2026-11-04)](https://www.postgresql.org/message-id/CAAAe_zDM87k_d_UnkYVeG-XP%2Bz_aBHWQFoe8osTN7DYcb_32cA%40mail.gmail.com)

**Question:** Will the frame contract change of message #547 land before commit, and if not, which window-function and transition-function behaviour would change later?

Counter-check: held. An author proposes in #543 and #547 to rethink the frame representation, and the thread does not say whether that lands before commit. The extension scenario is not supported: #547 says results are right today and readers keep reading the frame positions as before.

### 2. No stated bound on NFA state memory for patterns any user can write

fixable · level 2, rubric item 7 · security / privileges · `security-1`

A user who can only run SELECT writes a nested, nullable, alternating pattern over a large partition. The matcher keeps growing its NFA state set in backend memory until the host runs out of memory, and the OOM killer or a crash restart hits every session.

RPR adds a new regex-like engine that runs on input any user controls, but the material gives no state cap, no work_mem accounting and no error for an oversized match. Message #540 lists state explosion as still open, with fixes not yet ready. The interrupt checks added in messages #291 and #502 let statement_timeout cancel a runaway query, but they do not limit how much memory it takes before the timeout fires. The existing regex code in Postgres stops with an error when a regex grows too large, so a committer would likely ask for the same kind of limit here.

Thread: [#291 (Henson Choi, 2026-03-29)](https://www.postgresql.org/message-id/CAAAe_zBCF3dwSjStmG0kJqw_y1z8QD73Rf1G58QTKEvd9tScwA%40mail.gmail.com), [#502 (Henson Choi, 2026-09-09)](https://www.postgresql.org/message-id/CAAAe_zCuex4qU44vxA5gmGTieh9GQn2xZxT2o4-bfeOWaBLV%3Dw%40mail.gmail.com), [#514 (Henson Choi, 2026-09-19)](https://www.postgresql.org/message-id/CAAAe_zCN0iy1-keB5mxqA2LKU2omAdFcpi8rap5zGjUjJh9PcA%40mail.gmail.com), [#539 (Tatsuo Ishii, 2026-10-19)](https://www.postgresql.org/message-id/20261019.095259.1988929700676866808.ishii%40postgresql.org), [#540 (Henson Choi, 2026-10-19)](https://www.postgresql.org/message-id/CAAAe_zCb9847vcoaH72wvk61O-xOH_OSzYHt40TH37xjz%3DKmKg%40mail.gmail.com)

**Question:** What limit stops one unprivileged query from growing NFA state without bound, and what error or memory figures show for a worst-case pattern?

Counter-check: held. Message #540 confirms state explosion is open, and the material shows no cap or memory figures. A SELECT-only user can already exhaust memory in other ways, so this is a robustness question more than a new privilege problem.

### 3. View deparse of DEFINE keeps failing round-trip, which breaks pg_upgrade

fixable · level 2, rubric item 9 · pg_dump / restore round-trip, on-disk format / pg_upgrade, maintenance burden ("who maintains this in 10 years") · `replication-ondisk-1`

A user creates a view or a SQL function with BEGIN ATOMIC whose DEFINE reads a bare column, then later adds or renames a column in a joined table. At the next major upgrade, pg_upgrade's schema dump emits text that does not re-parse. The upgrade stops at restore time, long after the view was created.

DEFINE prints columns without a qualifier, because the qualifier slot belongs to the pattern variable. So every bare name must resolve the same way when the text is read back. The thread found this failing for ADD COLUMN, RENAME COLUMN, USING merges and FULL JOIN COALESCE (message #530), and for function parameters printed as f.threshold (message #532). Message #527 still lists a window-name deparse case as a known issue. The fix reserves names case by case in ruleutils.c, so nothing fails loudly when a later ruleutils change or a new feature such as qualified pattern variables does not know about DEFINE.

Thread: [#527 (Henson Choi, 2026-10-08)](https://www.postgresql.org/message-id/CAAAe_zDsYugq506ou49PU%2BOk%2B4Umn5n59Qs5wYofvKyfEpvZJQ%40mail.gmail.com), [#530 (Henson Choi, 2026-10-09)](https://www.postgresql.org/message-id/CAAAe_zBSWRe2uxFVs5bwx%2BkutCqXYv801S3Vg%2B9wP_cWjJcnvQ%40mail.gmail.com), [#532 (Henson Choi, 2026-10-09)](https://www.postgresql.org/message-id/CAAAe_zBno3aWzkJ6iu-Qsj-wniF%2Bj%2Bi3crZRs9DX9UCZZjG_Xg%40mail.gmail.com), [#537 (Tatsuo Ishii, 2026-10-14)](https://www.postgresql.org/message-id/20261014.105043.724886930578991800.ishii%40postgresql.org)

**Question:** What test checks that every RPR view and function body in the regression suite survives pg_dump and restore under pg_upgrade, and what remains open in the window-name case?

Counter-check: held. Message #253 keeps RPR test objects for the pg_upgrade and pg_dump tests, which answers part of the question. But message #532 reports a function body that still does not restore, and message #527 lists an open window-name deparse case.

### 4. DEFINE adds an expression site every planner rewrite must remember

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years"), planner behaviour change · `maintenance-fit-1`

A later patch adds or changes a planner pass that rewrites expressions, such as pull-up, grouping or alias expansion, and does not handle WindowClause.defineClause. RPR queries then fail with "variable not found in subplan target list" or crash the backend. Users find out in a minor release, since nothing in the build or the core tests fails.

Message #529 says every planner step that rewrites expressions "has to treat DEFINE the same way, or its copy drifts out of step". The thread shows the cost of missing one: grouping (#529), subquery pull-up (#511), a setrefs crash (#513), a volatility check bypassed through GROUP BY (#513), and a defect found after the fix was cut (#532). Each defect was found by RPR-specific black-box tests, not by a check that fails when a pass forgets the clause.

Thread: [#511 (Henson Choi, 2026-09-18)](https://www.postgresql.org/message-id/CAAAe_zBufaHrGyJK4zzQ-U9YZLVmhpBOjBuLugzqdtwE_qf7Nw%40mail.gmail.com), [#513 (Henson Choi, 2026-09-18)](https://www.postgresql.org/message-id/CAAAe_zDwXWsb_ATce%3D1yBgioFDTaTBDOGUvEy0NaDXcfN3ZTEw%40mail.gmail.com), [#529 (Henson Choi, 2026-10-09)](https://www.postgresql.org/message-id/CAAAe_zBr-wZyE%2BDU3uK8OATdMJsSLE2nHEeHcyQA0FD3KpJJaw%40mail.gmail.com), [#532 (Henson Choi, 2026-10-09)](https://www.postgresql.org/message-id/CAAAe_zBno3aWzkJ6iu-Qsj-wniF%2Bj%2Bi3crZRs9DX9UCZZjG_Xg%40mail.gmail.com)

**Question:** What mechanism makes a new planner expression pass fail loudly if it does not visit defineClause, beyond the RPR regression files?

Counter-check: held. havingQual and frame offsets carry the same burden, and the tree walkers now visit defineClause (#259, #335), which lowers the risk. The rule that a navigation argument must not be folded is new, and the thread shows five defects of this kind.

### 5. Catalog names prev, next, first, last may capture existing user functions

fixable · level 2, rubric item 12 · SQL-standard conflict or non-standard syntax, catalog changes, pg_dump / restore round-trip · `standards-compat-1`

A site has its own aggregate public.last(anyelement) or uses an extension that defines first() and last(). After upgrade, a query calls last(x) outside a window. If it resolves to the new pg_catalog entry, the query fails because navigation is allowed only in DEFINE, and the user finds out in production.

The patch adds navigation entries to pg_proc.dat (v20-0005, message #312), and message #222 restricts PREV and NEXT to DEFINE. pg_catalog is searched first, so a built-in with the same name and signature can shadow a user function. First and last aggregates are a common user and extension pattern. The proposal does not say how these names resolve outside DEFINE, or inside DEFINE when the user has an object with the same name.

Thread: [#222 (Tatsuo Ishii, 2026-02-19)](https://www.postgresql.org/message-id/20260219.134751.1095180149902968829.ishii%40postgresql.org), [#312 (Henson Choi, 2026-05-01)](https://www.postgresql.org/message-id/CAAAe_zBdwAUDNs_WFdLkFF%3DewhkDkv-AqizVEVzhsfremGFb4w%40mail.gmail.com), [#405 (Henson Choi, 2026-06-21)](https://www.postgresql.org/message-id/CAAAe_zCsaf%3DWedELLjqLe3BV_8dWiO1DPDGA9sXj4qhe%2B%3D-XXw%40mail.gmail.com)

**Question:** With a user-defined public.first(anyelement) or public.last(anyelement) aggregate installed, do existing queries outside DEFINE behave the same after upgrade, and what does DEFINE resolve to?

Counter-check: held. The series adds navigation entries to pg_proc.dat, and pg_catalog is searched before user schemas, so a same-signature user first() or last() can be shadowed. The thread does not say how these names resolve outside DEFINE.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | Stored views and SQL function bodies with DEFINE must restore under pg_upgrade; message #532 reports one that does not. |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | **yes** | The series adds navigation function entries to pg_proc.dat. |
| ABI / extension API | **yes** | The reduced frame changes what window functions and aggregates read through the window API in nodeWindowAgg.c. |
| SQL-standard conflict or non-standard syntax | **yes** | The patch adds PATTERN, DEFINE, INITIAL and AFTER MATCH SKIP to the window clause as a subset of SQL R020, with new keywords. |
| new GUC | no |  |
| planner behaviour change | **yes** | DEFINE is planned like havingQual, and run condition pushdown and frame replacement are disabled for RPR windows. |
| pg_dump / restore round-trip | **yes** | ruleutils.c deparses DEFINE, and pg_get_viewdef() output changes for some queries without RPR. |
| psql UX conventions | no |  |
| security / privileges | **yes** | Any user who can run SELECT can write a pattern that drives NFA state growth in backend memory. |
| backpatchability | **yes** | The series carries a TABLEFUNC alias list deparse fix for a bug that message #530 says exists back to 10. |
| maintenance burden ("who maintains this in 10 years") | **yes** | A new NFA matcher in the executor, a new expression site in the planner, and name reservation in ruleutils.c need lasting care. |
| performance on the hot path | **yes** | The series changes nodeWindowAgg.c, which every window query runs; the review notes say the cost appears limited to RPR windows. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Will the frame contract change proposed in message #547 land before commit, and if not, which window-function and transition-function behaviour would change later?
2. What limit stops one query from growing NFA state without bound, and what are the memory and run time figures for a worst-case pattern?
3. Which test checks that every RPR view and SQL function body survives pg_dump and restore under pg_upgrade, and what remains open in the window-name case?
4. What makes a new planner expression pass fail loudly if it does not visit defineClause, beyond the RPR regression files?
5. With a user-defined public.first(anyelement) or public.last(anyelement) installed, how do calls outside DEFINE and inside DEFINE resolve after upgrade?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. It needs a committer who knows the planner (setrefs.c, subquery pull-up), ruleutils.c deparse and the nodeWindowAgg.c frame code. Tatsuo Ishii is an author, but writes in message #539 that he still needs to learn more about the planner, and no such reviewer replied to #529 or #530.
- **Liability:** The owner signs for a new NFA matcher in the executor, a new expression site that every planner rewrite must handle, DEFINE deparse that must round-trip for ever, and standard syntax that cannot be withdrawn once released.
- **Author follow-up:** Strong sign that the authors stay: 17 CommitFests, v54 posted on 2026-10-20, and three authors still exchanging fixes and refactorings.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs champion
- **Audience:** a sizeable minority
- **Maintenance burden:** high
- **Review cost:** high
- **Why:** Analytic users gain a SQL-standard feature that several other systems offer, and the authors are active. The series spans parser, planner, executor and ruleutils.c, and the visible thread shows no planner or deparse committer outside the author group reviewing those parts.

## Appendix

### Held up in the counter-check, not listed above

- `standards-compat-2` (fixable, level 2) DEFINE uses the general expression grammar, which may block RUNNING and MEASURES later. Not listed: not among the strongest objections.
- `maintenance-fit-2` (fixable, level 2) Shared planner and ruleutils changes lack an engaged owner. Not listed: not among the strongest objections.

### Refuted in the counter-check

- `standards-compat-3` Deparse changes for views without row pattern recognition ride in the RPR series. Refuted (already handled): Messages #530 and #540 commit to moving the generic ruleutils.c fixes into a prerequisite patch with its own thread and CF entry. The changed output replaces view text that did not re-parse, so the scenario describes a bug fix.
- `standards-compat-4` Marking R020 YES overstates conformance while MEASURES and qualified variables are missing. Refuted (no failure scenario): The thread proposes YES with a note, and sql_features shows that note in the same row, so the user in the scenario sees the gap. The entry is a one-line choice the authors list as open, and it does not affect the design.
- `replication-ondisk-2` Navigation functions in pg_proc can shadow extension functions named first or last. Refuted (not specific): Duplicate of standards-compat-1.
- `maintenance-fit-4` Shipped DEFINE grammar may block RUNNING, MEASURES and qualifiers. Refuted (not specific): Duplicate of standards-compat-2.
- `demand-1` Demand cited is for MATCH_RECOGNIZE features this subset leaves out. Refuted (wrong facts): Message #519 speaks only of queries that use RUNNING, not of migrated queries in general. Trino has the window form, Vik Fearing and Jacob Champion asked for this feature in #2 and #9, and the subset runs the V-shape query of message #1 with existing window functions.

### Review angles

- prior-attempts: 0 objection(s). No precedent found: the precedent list has no earlier rejected, returned or reverted proposal of row pattern recognition or a close relative. The thread does not point to an earlier attempt at this idea either. Message #1 says only Oracle and some warehouse products implement row pattern recognition, not that Postgres tried it before.
- standards-compat: 4 objection(s). Objections found; see list.
- replication-ondisk: 2 objection(s). The feature adds no WAL records, no on-disk relation format and no wire protocol change. Persistent state is limited to stored query trees and the catalog, and both are covered above.
- security: 1 objection(s). RLS and security barrier quals apply below the WindowAgg, volatile DEFINE expressions are rejected, and run condition pushdown is disabled, so the material shows no way past row filtering. The only concern from this angle is resource exhaustion.
- maintenance-fit: 4 objection(s). Objections raised; hot-path cost appears limited to RPR windows, since run-condition and frame optimizations are disabled only when defineClause is set.
- already-exists: 0 objection(s). Postgres has no existing feature that does row pattern matching. lag()/lead() with recursive CTEs cannot express general quantifiers, alternation or reluctant matching at reasonable effort, and this SQL-standard window syntax needs grammar changes that an extension cannot add.
- demand: 1 objection(s). Demand for row pattern matching in general is shown: it is a SQL standard feature, several other systems have it, and independent reviewers asked for it. Cost falls mostly on queries that use RPR. The only open point is whether this subset serves the users that this evidence describes.

### Precedents retrieved

- P1: [Recognize D part for to_date() and to_timestamp() in 'WW-D' pattern](https://commitfest.postgresql.org/patch/2389/) (cf, Returned with feedback, 2020-07-10) - judged not relevant
- P2: [Fix to_char() behaviour with 'WW-D' pattern](https://commitfest.postgresql.org/patch/2388/) (cf, Returned with feedback, 2020-07-10) - judged not relevant
- P3: [Pattern based listeners for asynchronous messaging (LISTEN/NOTIFY)](https://commitfest.postgresql.org/patch/1218/) (cf, Returned with feedback, 2017-11-30) - judged not relevant
- P4: [Revert "Fix under-quoted filename pattern in pgbench TAP test."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f4593bd2ff0376fbc56dd1b384c7d51e1a7062f0) (revert, Reverted, 2019-01-21) - judged not relevant
- P5: [Add regexp_positions(string text, pattern text, flags text) → setof record (starts integer\[\], lengths integer\[\])](https://commitfest.postgresql.org/patch/3027/) (cf, Returned with feedback, 2021-12-03) - judged not relevant; reason summarised from its thread
- P6: [Make selectivity example match wording](https://commitfest.postgresql.org/patch/3695/) (cf, Returned with feedback, 2022-11-30) - judged not relevant; reason summarised from its thread
- P7: [basebackup: support zstd long distance matching](https://commitfest.postgresql.org/patch/4111/) (cf, Withdrawn, 2023-04-04) - judged not relevant; reason summarised from its thread
- P8: [Document the limitations of row-level security](https://commitfest.postgresql.org/patch/2027/) (cf, Returned with feedback, 2019-04-04) - judged not relevant; reason summarised from its thread
- P9: [Reduce wakeup on idle for bgwriter & walwriter for >5s](https://commitfest.postgresql.org/patch/4029/) (cf, Withdrawn, 2023-03-30) - judged not relevant; reason summarised from its thread
- P10: [use regular expressions to simplify less_greater and not_equals](https://commitfest.postgresql.org/patch/3281/) (cf, Rejected, 2021-09-05) - judged not relevant; reason summarised from its thread
- P11: [Recognizing superuser in pg_hba.conf](https://commitfest.postgresql.org/patch/2397/) (cf, Returned with feedback, 2020-07-03) - judged not relevant; reason summarised from its thread
- P12: [Implement NULL-related checks in object address functions to prevent cache lookup errors](https://commitfest.postgresql.org/patch/1204/) (cf, Returned with feedback, 2018-11-29) - judged not relevant; reason summarised from its thread
- P13: [Make executor's Range Table an array instead of a List](https://commitfest.postgresql.org/patch/1771/) (cf, Returned with feedback, 2018-08-23) - judged not relevant
- P14: [Revert "MERGE SQL Command following SQL:2016"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=aa5877bb26347c58a34aee4e460eb1e1123bb096) (revert, Reverted, 2018-04-02) - judged not relevant
- P15: [extension patch to add OR REPLACE clause to CREATE TRIGGER](https://commitfest.postgresql.org/patch/2033/) (cf, Withdrawn, 2019-03-01) - judged not relevant
- P16: [Support for grabbing multiple consecutive values with nextval()](https://commitfest.postgresql.org/patch/3577/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P17: [Revert "Common SQL/JSON clauses"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1460fc5942591fdb6bee0bc8342019ede31ff3b6) (revert, Reverted, 2022-03-22) - judged not relevant
- P18: [Write Amplification Reduction Method (WARM)](https://commitfest.postgresql.org/patch/775/) (cf, Returned with feedback, 2017-10-02) - judged not relevant
- P19: [Reduce the time required for a database recovery from archive.](https://commitfest.postgresql.org/patch/2732/) (cf, Returned with feedback, 2020-12-01) - judged not relevant
- P20: [iteration over record in plpgsql](https://commitfest.postgresql.org/patch/124/) (cf, Returned with feedback, 2015-07-07) - judged not relevant
- P21: [Reduce the time required for a database recovery from archive.](https://commitfest.postgresql.org/patch/2935/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P22: [Revert "Add libpq function PQhostaddr()."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2cda889984a6f5ad405318f9e91202d258383b66) (revert, Reverted, 2014-11-29) - judged not relevant
- P23: [loop over unbound cursor](https://commitfest.postgresql.org/patch/2558/) (cf, Withdrawn, 2020-08-10) - judged not relevant
- P24: [Revert SQL/JSON features](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2f2b18bd3f554e96a8cc885b177211be12288e4a) (revert, Reverted, 2022-09-01) - judged not relevant

### Warnings

- attachment limit (8) reached; v1-0001-Row-pattern-recognition-patch-for-raw-parser.patch not read
- attachment limit (8) reached; v1-0002-Row-pattern-recognition-patch-parse-analysis.patch not read
- attachment limit (8) reached; v1-0003-Row-pattern-recognition-patch-planner.patch not read
- attachment limit (8) reached; v1-0004-Row-pattern-recognition-patch-executor.patch not read
- attachment limit (8) reached; v1-0005-Row-pattern-recognition-patch-docs.patch not read
- attachment limit (8) reached; v1-0006-Row-pattern-recognition-patch-tests.patch not read
- 498 long messages in the middle of the thread were left out of the prompts

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-fable-5-2, synthesis claude-fable-5-2; no tools given to any of them
- LLM calls: 18; wall time: 223 s; cost reported by the CLI: $11.45
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 2 from cache, 6 summarised now, 0 failed; quotes: 4 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 110 text field(s) checked, 1 finding(s), 1 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T00:54:21.743Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

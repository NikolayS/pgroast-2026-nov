# Design pre-review: Implement row pattern recognition feature

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 4460](https://commitfest.postgresql.org/patch/4460/); [thread](https://www.postgresql.org/message-id/flat/20230625.210509.1276733411677577841.t-ishii%40sranhm.sra.co.jp), 549 of 549 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No surviving objection is fatal, and none fails level 1a or 1b: the problem is stated and demand is shown. The surviving level 2 objections are fixable design points: the open frame redesign (maintenance-fit-3), no stated NFA memory bound (security-1), DEFINE deparse round-trip (replication-ondisk-1) and the planner burden of defineClause (maintenance-fit-1). They need a committer who knows the planner, ruleutils.c and nodeWindowAgg.c to judge them.

Not checked: Only v20 patch prose and the first three messages were read, while the current version is v54. NFA memory behaviour, the frame code and the planner handling of defineClause could not be checked without the code.

## Prior attempts

None found before 2023-06-25.

## Objections a reviewer is likely to raise

### 1. Reduced-frame design is still open and would change the window API

The current overlay design ships, and extension window functions start using it. The proposed redesign in message #547 then changes what frameheadpos and frametailpos mean, including that T may decrease and H may move backward. Those extensions, and the core readers that work out the end from frameOptions, need changes in a later release.

Message #543 and message #547 say the current reduced frame cannot express SEEK. They also say it restarts aggregates on every row, which is quadratic, and that it spools the partition for last_value(). Message #547 lists six frame contracts (C1-C6) that the readers rely on, and it notes that one change touches the contract of transition functions. A committer would likely ask that this contract be settled before WinGetFuncArgInFrame() and related code take on RPR-specific behaviour.

**Question:** Will the frame contract change of message #547 land before commit, and if not, which window-function and transition-function behaviour would change later?

<sub>fixable, level 2, ABI / extension API; messages [#542](https://www.postgresql.org/message-id/CACJufxFEwTRc4JEbRoYVWPYE-KM6B%3D7qWYtg6SpsZHfA2U3fWw%40mail.gmail.com), [#543](https://www.postgresql.org/message-id/CAAAe_zCqiC5iqoYq5uaTWUCpsUsv-7pOjScwfm66wu28OYWOZA%40mail.gmail.com), [#547](https://www.postgresql.org/message-id/CAAAe_zDM87k_d_UnkYVeG-XP%2Bz_aBHWQFoe8osTN7DYcb_32cA%40mail.gmail.com); `maintenance-fit-3`</sub>

### 2. No stated bound on NFA state memory for patterns any user can write

A user who can only run SELECT writes a nested, nullable, alternating pattern over a large partition. The matcher keeps growing its NFA state set in backend memory until the host runs out of memory, and the OOM killer or a crash restart hits every session.

RPR adds a new regex-like engine that runs on input any user controls, but the material gives no state cap, no work_mem accounting and no error for an oversized match. Message #540 lists state explosion as still open, with fixes not yet ready. The interrupt checks added in messages #291 and #502 let statement_timeout cancel a runaway query, but they do not limit how much memory it takes before the timeout fires. The existing regex code in Postgres stops with an error when a regex grows too large, so a committer would likely ask for the same kind of limit here.

**Question:** What limit stops one unprivileged query from growing NFA state without bound, and what error or memory figures show for a worst-case pattern?

<sub>fixable, level 2, security / privileges; messages [#291](https://www.postgresql.org/message-id/CAAAe_zBCF3dwSjStmG0kJqw_y1z8QD73Rf1G58QTKEvd9tScwA%40mail.gmail.com), [#502](https://www.postgresql.org/message-id/CAAAe_zCuex4qU44vxA5gmGTieh9GQn2xZxT2o4-bfeOWaBLV%3Dw%40mail.gmail.com), [#514](https://www.postgresql.org/message-id/CAAAe_zCN0iy1-keB5mxqA2LKU2omAdFcpi8rap5zGjUjJh9PcA%40mail.gmail.com), [#539](https://www.postgresql.org/message-id/20261019.095259.1988929700676866808.ishii%40postgresql.org), [#540](https://www.postgresql.org/message-id/CAAAe_zCb9847vcoaH72wvk61O-xOH_OSzYHt40TH37xjz%3DKmKg%40mail.gmail.com); `security-1`</sub>

### 3. View deparse of DEFINE keeps failing round-trip, which breaks pg_upgrade

A user creates a view or a SQL function with BEGIN ATOMIC whose DEFINE reads a bare column, then later adds or renames a column in a joined table. At the next major upgrade, pg_upgrade's schema dump emits text that does not re-parse. The upgrade stops at restore time, long after the view was created.

DEFINE prints columns without a qualifier, because the qualifier slot belongs to the pattern variable. So every bare name must resolve the same way when the text is read back. The thread found this failing for ADD COLUMN, RENAME COLUMN, USING merges and FULL JOIN COALESCE (message #530), and for function parameters printed as f.threshold (message #532). Message #527 still lists a window-name deparse case as a known issue. The fix reserves names case by case in ruleutils.c, so nothing fails loudly when a later ruleutils change or a new feature such as qualified pattern variables does not know about DEFINE.

**Question:** What test checks that every RPR view and function body in the regression suite survives pg_dump and restore under pg_upgrade, and what remains open in the window-name case?

<sub>fixable, level 2, pg_dump / restore round-trip; messages [#527](https://www.postgresql.org/message-id/CAAAe_zDsYugq506ou49PU%2BOk%2B4Umn5n59Qs5wYofvKyfEpvZJQ%40mail.gmail.com), [#530](https://www.postgresql.org/message-id/CAAAe_zBSWRe2uxFVs5bwx%2BkutCqXYv801S3Vg%2B9wP_cWjJcnvQ%40mail.gmail.com), [#532](https://www.postgresql.org/message-id/CAAAe_zBno3aWzkJ6iu-Qsj-wniF%2Bj%2Bi3crZRs9DX9UCZZjG_Xg%40mail.gmail.com), [#537](https://www.postgresql.org/message-id/20261014.105043.724886930578991800.ishii%40postgresql.org); `replication-ondisk-1`</sub>

### 4. DEFINE adds an expression site every planner rewrite must remember

A later patch adds or changes a planner pass that rewrites expressions, such as pull-up, grouping or alias expansion, and does not handle WindowClause.defineClause. RPR queries then fail with "variable not found in subplan target list" or crash the backend. Users find out in a minor release, since nothing in the build or the core tests fails.

Message #529 says every planner step that rewrites expressions "has to treat DEFINE the same way, or its copy drifts out of step". The thread shows the cost of missing one: grouping (#529), subquery pull-up (#511), a setrefs crash (#513), a volatility check bypassed through GROUP BY (#513), and a defect found after the fix was cut (#532). Each defect was found by RPR-specific black-box tests, not by a check that fails when a pass forgets the clause.

**Question:** What mechanism makes a new planner expression pass fail loudly if it does not visit defineClause, beyond the RPR regression files?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#511](https://www.postgresql.org/message-id/CAAAe_zBufaHrGyJK4zzQ-U9YZLVmhpBOjBuLugzqdtwE_qf7Nw%40mail.gmail.com), [#513](https://www.postgresql.org/message-id/CAAAe_zDwXWsb_ATce%3D1yBgioFDTaTBDOGUvEy0NaDXcfN3ZTEw%40mail.gmail.com), [#529](https://www.postgresql.org/message-id/CAAAe_zBr-wZyE%2BDU3uK8OATdMJsSLE2nHEeHcyQA0FD3KpJJaw%40mail.gmail.com), [#532](https://www.postgresql.org/message-id/CAAAe_zBno3aWzkJ6iu-Qsj-wniF%2Bj%2Bi3crZRs9DX9UCZZjG_Xg%40mail.gmail.com); `maintenance-fit-1`</sub>

### 5. Catalog names prev, next, first, last may capture existing user functions

A site has its own aggregate public.last(anyelement) or uses an extension that defines first() and last(). After upgrade, a query calls last(x) outside a window. If it resolves to the new pg_catalog entry, the query fails because navigation is allowed only in DEFINE, and the user finds out in production.

The patch adds navigation entries to pg_proc.dat (v20-0005, message #312), and message #222 restricts PREV and NEXT to DEFINE. pg_catalog is searched first, so a built-in with the same name and signature can shadow a user function. First and last aggregates are a common user and extension pattern. The proposal does not say how these names resolve outside DEFINE, or inside DEFINE when the user has an object with the same name.

**Question:** With a user-defined public.first(anyelement) or public.last(anyelement) aggregate installed, do existing queries outside DEFINE behave the same after upgrade, and what does DEFINE resolve to?

<sub>fixable, level 2, SQL-standard conflict or non-standard syntax; messages [#222](https://www.postgresql.org/message-id/20260219.134751.1095180149902968829.ishii%40postgresql.org), [#312](https://www.postgresql.org/message-id/CAAAe_zBdwAUDNs_WFdLkFF%3DewhkDkv-AqizVEVzhsfremGFb4w%40mail.gmail.com), [#405](https://www.postgresql.org/message-id/CAAAe_zCsaf%3DWedELLjqLe3BV_8dWiO1DPDGA9sXj4qhe%2B%3D-XXw%40mail.gmail.com); `standards-compat-1`</sub>

## Questions for the author

1. Will the frame contract change proposed in message #547 land before commit, and if not, which window-function and transition-function behaviour would change later?
2. What limit stops one query from growing NFA state without bound, and what are the memory and run time figures for a worst-case pattern?
3. Which test checks that every RPR view and SQL function body survives pg_dump and restore under pg_upgrade, and what remains open in the window-name case?
4. What makes a new planner expression pass fail loudly if it does not visit defineClause, beyond the RPR regression files?
5. With a user-defined public.first(anyelement) or public.last(anyelement) installed, how do calls outside DEFINE and inside DEFINE resolve after upgrade?

## Touch points

- **on-disk format / pg_upgrade:** Stored views and SQL function bodies with DEFINE must restore under pg_upgrade; message #532 reports one that does not.
- **catalog changes:** The series adds navigation function entries to pg_proc.dat.
- **ABI / extension API:** The reduced frame changes what window functions and aggregates read through the window API in nodeWindowAgg.c.
- **SQL-standard conflict or non-standard syntax:** The patch adds PATTERN, DEFINE, INITIAL and AFTER MATCH SKIP to the window clause as a subset of SQL R020, with new keywords.
- **planner behaviour change:** DEFINE is planned like havingQual, and run condition pushdown and frame replacement are disabled for RPR windows.
- **pg_dump / restore round-trip:** ruleutils.c deparses DEFINE, and pg_get_viewdef() output changes for some queries without RPR.
- **security / privileges:** Any user who can run SELECT can write a pattern that drives NFA state growth in backend memory.
- **backpatchability:** The series carries a TABLEFUNC alias list deparse fix for a bug that message #530 says exists back to 10.
- **maintenance burden ("who maintains this in 10 years"):** A new NFA matcher in the executor, a new expression site in the planner, and name reservation in ruleutils.c need lasting care.
- **performance on the hot path:** The series changes nodeWindowAgg.c, which every window query runs; the review notes say the cost appears limited to RPR windows.

## Who would commit it, and when

- Owner: silent. It needs a committer who knows the planner (setrefs.c, subquery pull-up), ruleutils.c deparse and the nodeWindowAgg.c frame code. Tatsuo Ishii is an author, but writes in message #539 that he still needs to learn more about the planner, and no such reviewer replied to #529 or #530.
- Priority: needs champion; audience a sizeable minority; review cost high. Analytic users gain a SQL-standard feature that several other systems offer, and the authors are active. The series spans parser, planner, executor and ruleutils.c, and the visible thread shows no planner or deparse committer outside the author group reviewing those parts.

---

[roast-full.md](roast-full.md) has 2 more objection(s) that held, 5 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 7.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

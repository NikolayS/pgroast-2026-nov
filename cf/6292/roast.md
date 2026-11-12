# Design pre-review: Proposal to allow setting cursor options on Portals

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6292](https://commitfest.postgresql.org/patch/6292/); [thread](https://www.postgresql.org/message-id/flat/CADK3HHKe1PA1U6aB5-7tWBQ0yZGgNvY7H%3DECDD9955Pas_zx_Q%40mail.gmail.com), 54 of 54 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The goal has support in the thread, and the choice of a protocol extension over a version bump is settled. The design of v5 is not: standards-compat-1 shows that negotiating _pq_.cursor changes every Bind and Execute, and the PortalHold alternative from message #52 has no answer. standards-compat-3 and maintenance-fit-2 show that Bind with cursor options and DECLARE already behave differently. already-exists-1 questions whether the Execute fields in 0002 belong in this patch.

Not checked: Without the code, the crash fixes in 0004 and 0005 and the reported isTopLevel difference for a directional Execute could not be checked. The review saw only 3 of 54 messages in full, so later thread facts come from the panel's reports.

## Prior attempts

- [cursors with prepared statements](https://www.postgresql.org/message-id/flat/762cc764-74f0-13fb-77ed-16f91c90f40d@2ndquadrant.com) (similar idea; Returned with feedback, 2018-12-01). "I hope it's not another abandoned patch, but due to lack of response I'm marking it as returned with feedback." ([message](https://www.postgresql.org/message-id/CA%2Bq6zcWJifdjoXiqPqiRAqw%3DEE%3DDVLksr4kFz%3DVevwyrU_iqTA%40mail.gmail.com)) Different now: P1 added SQL syntax to declare a cursor over a prepared statement. This proposal leaves SQL alone and adds cursor options to Bind and fetch fields to Execute through a negotiated protocol extension.

## Objections a reviewer is likely to raise

### 1. Negotiating _pq_.cursor changes every Bind and Execute on the connection

An application sets protocol_cursor=1 behind PgBouncer 1.22.1 with ignore_startup_parameters = _pq_.cursor. libpq gets no NegotiateProtocolVersion, assumes the option was accepted, and adds the new fields. From then on every extended query on that connection fails with "invalid message format", including queries that never use a cursor.

Message #52 reports this failure and shows that a pooler cannot share a server connection between clients that did and did not negotiate the option. In v5 the fields are mandatory once negotiated, so the cost falls on every Bind and Execute rather than only on messages that use the feature. Message #52 proposes a separate PortalHold message that leaves Bind and Execute unchanged, and the thread has no answer to it. The author's reason for mandatory fields, that length sniffing cannot carry a second extension (message #48), holds just as well for a new message.

**Question:** Why must holdability travel inside Bind instead of in a separate message, given that only PreCommit_Portals reads CURSOR_OPT_HOLD?

<sub>fixable, level 2, wire protocol; messages [#48](https://www.postgresql.org/message-id/CADK3HH%2BpdutF6dQG9-fDPtJri%2B%3D1-o9YKJQLeO9u6gj7cisU0A%40mail.gmail.com), [#52](https://www.postgresql.org/message-id/CAB%3DJe-HDihfXy%2BXRQtukPkyft%3Dx2%2BC660wroyfRh7g9b4iiV5g%40mail.gmail.com), [#28](https://www.postgresql.org/message-id/CAGECzQSHKR2LtBSnDT_bjdpE1%3DcVzZLRcb2k5d_K%3DBMvJCFihg%40mail.gmail.com), [#30](https://www.postgresql.org/message-id/CAOYmi%2BkkQqRw2ajt5e-CX5YchrMwS_qJ4%2B94Yjxh%3DdHGo%2B8jPA%40mail.gmail.com); `standards-compat-1`</sub>

### 2. Protocol cursors do not follow the rules DECLARE applies

A session with plan_cache_mode = force_generic_plan binds a prepared GROUP BY query with $1 = 0. Without SCROLL it returns 10 rows. With SCROLL the replan folds the parameter in as a constant and Bind fails with division by zero. The same query through DECLARE SCROLL is planned under DECLARE's rules, so the two paths give different results.

v5 crashes on SCROLL for plans that cannot scan backward and on HOLD for non-SELECT portals (message #49). The 0004 fixes then make Bind differ from DECLARE in other ways: it accepts FOR UPDATE with a data-modifying WITH that DECLARE refuses, and it rejects FOR UPDATE in subqueries, CTEs and views that DECLARE accepts (message #51). The documentation says "as for DECLARE", so drivers will rely on a contract that is not met. The material does not show that the author has adopted 0004 or chosen which behaviour is correct.

**Question:** For which queries and settings does a Bind with SCROLL or HOLD behave the same as DECLARE with the same options, and where does it differ on purpose?

<sub>fixable, level 2, planner behaviour change; messages [#49](https://www.postgresql.org/message-id/CAH7T-aqbVB8t1ZPiqVMhYvwU8Q3AbABBUgSaDvafAWY7xbeS3g%40mail.gmail.com), [#51](https://www.postgresql.org/message-id/CAB%3DJe-HjGipRGuxP98BnqO1jhTTrSJx5z7m9000ugc0%2B3F1rKg%40mail.gmail.com); `standards-compat-3`</sub>

### 3. Second cursor planning and checking path beside DECLARE

A future patch adds a cursor restriction or planning flag to DECLARE but not to Bind. Clients that use Bind with SCROLL or HOLD then get plans or checks that differ from DECLARE, or hit an executor assertion. Nothing fails at build or test time when the two paths drift.

Message #49 found crashes because Bind reused cached plans not built for backward scans, and allowed HOLD on non-SELECT portals. The fix in 0004 re-plans outside the plan cache. Message #51 shows it still differs from DECLARE in four ways: data-modifying WITH, force_generic_plan, a stricter FOR UPDATE check, and no CURSOR_OPT_FAST_PLAN. Each cursor rule now lives in two places.

**Question:** Can Bind with cursor options share DECLARE's planning and validation code, so that one rule change covers both paths?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#49](https://www.postgresql.org/message-id/CAH7T-aqbVB8t1ZPiqVMhYvwU8Q3AbABBUgSaDvafAWY7xbeS3g%40mail.gmail.com), [#51](https://www.postgresql.org/message-id/CAB%3DJe-HjGipRGuxP98BnqO1jhTTrSJx5z7m9000ugc0%2B3F1rKg%40mail.gmail.com); `maintenance-fit-2`</sub>

### 4. Execute fetch fields duplicate SQL FETCH and MOVE on named portals

Under _pq_.cursor, every driver and pooler must send and parse 12 extra bytes on every Execute to gain what FETCH and MOVE already do. A second fetch path then drifts from SQL FETCH. One case is known: a CALL with an INOUT parameter that commits works through a plain Execute and fails with 2D000 through a directional Execute.

SQL FETCH and MOVE already read a named portal created by Bind in any direction, as message #52 points out, and the author says in message #54 that everything the patch enables is possible through SQL today. The 0002 commit message justifies the fields because only SQL FETCH could scroll, so they save a text query and nothing more. Message #52 lists further defects that exist only because of the second path, such as direction checks running after log_statement.

**Question:** Which driver task needs a directional Execute that "FETCH BACKWARD n FROM portal" sent over the extended protocol cannot do, apart from saving one query string?

<sub>fixable, level 1b, doesn't fit what Postgres is about (extension / external tool / already solved); messages [#52](https://www.postgresql.org/message-id/CAB%3DJe-HDihfXy%2BXRQtukPkyft%3Dx2%2BC660wroyfRh7g9b4iiV5g%40mail.gmail.com), [#54](https://www.postgresql.org/message-id/CADK3HHKM4%2BhTWdLbg%2BEB0pAtWh%3D7dyzrnC0EZGoCYwMWKMknNg%40mail.gmail.com), [#48](https://www.postgresql.org/message-id/CADK3HH%2BpdutF6dQG9-fDPtJri%2B%3D1-o9YKJQLeO9u6gj7cisU0A%40mail.gmail.com); `already-exists-1`</sub>

### 5. Bind-time HOLD now covers only what DECLARE WITH HOLD already does

After 0004, a driver can set HOLD only on a single SELECT, which DECLARE ... WITH HOLD with $n parameters and binary FETCH results already supports. The case SQL cannot do, holding the result of INSERT ... RETURNING across COMMIT, is still rejected. Every Bind on the connection changes format for a gain that the driver could get with SQL.

Message #52 notes that holding a portal at the protocol level is strongest for RETURNING and data-modifying WITH, and that 0004 excludes exactly those. The author's case in messages #1 and #10 is that rewriting queries into DECLARE is cumbersome, which is a matter of convenience rather than a missing capability. The out-of-memory report in message #37 comes from pgjdbc ignoring the fetch size for holdable result sets; DECLARE WITH HOLD plus FETCH avoids that today.

**Question:** What does a held SELECT portal created by Bind give pgjdbc that DECLARE ... WITH HOLD with parameters and binary FETCH cannot, enough to justify changing every Bind?

<sub>fixable, level 1b, desirability; messages [#1](https://www.postgresql.org/message-id/CADK3HHKe1PA1U6aB5-7tWBQ0yZGgNvY7H%3DECDD9955Pas_zx_Q%40mail.gmail.com), [#10](https://www.postgresql.org/message-id/CADK3HHJVX_H%2B8ge%2BvMzU35CzbUSCw08FJ_1G39m2-SLwM3V6pQ%40mail.gmail.com), [#37](https://www.postgresql.org/message-id/2b3d5da9-34ba-49ee-a052-b6dd7233c524%40gmail.com), [#49](https://www.postgresql.org/message-id/CAH7T-aqbVB8t1ZPiqVMhYvwU8Q3AbABBUgSaDvafAWY7xbeS3g%40mail.gmail.com), [#52](https://www.postgresql.org/message-id/CAB%3DJe-HDihfXy%2BXRQtukPkyft%3Dx2%2BC660wroyfRh7g9b4iiV5g%40mail.gmail.com), [#54](https://www.postgresql.org/message-id/CADK3HHKM4%2BhTWdLbg%2BEB0pAtWh%3D7dyzrnC0EZGoCYwMWKMknNg%40mail.gmail.com); `already-exists-2`</sub>

## Questions for the author

1. Why must holdability travel inside Bind, and not in a separate message such as the PortalHold proposed in message #52, which leaves Bind and Execute unchanged?
2. Which driver task needs a directional Execute that SQL FETCH or MOVE on the same named portal cannot do, apart from saving one query string?
3. Will Sehrope's 0004 and 0005 be folded into the next version, and if so, which of the remaining differences from DECLARE listed in message #51 are intended?
4. Can Bind with SCROLL or HOLD share DECLARE's planning and validation code, so that one rule change covers both paths?
5. How does a transaction-mode pooler learn that a held portal exists or is gone, or will protocol.sgml state that HOLD is unsupported there?

## Touch points

- **wire protocol:** _pq_.cursor adds a mandatory Int32 to every Bind and an Int32 plus Int64 to every Execute once negotiated.
- **ABI / extension API:** exports.txt gains four libpq functions, and libpq-fe.h gains the PQ_FETCH_* constants.
- **SQL-standard conflict or non-standard syntax:** see objection standards-compat-3: Protocol cursors do not follow the rules DECLARE applies
- **planner behaviour change:** A Bind with SCROLL needs a plan that can scan backward, so it cannot reuse a cached plan as is (messages #49, #51).
- **security / privileges:** A held portal outlives its transaction, so behind a transaction-mode pooler another client can reach it (message #52).
- **maintenance burden ("who maintains this in 10 years"):** The patch adds a permanent wire format, four libpq exports, and a second cursor checking path beside DECLARE.
- **performance on the hot path:** exec_bind_message and the Execute path in postgres.c read extra fields, but only on connections that negotiated the option.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** The author states in message #54 that SQL DECLARE, FETCH and MOVE already provide everything the patch enables.

## Who would commit it, and when

- Owner: silent. It needs a committer who knows the extended query protocol, portal and plan cache code in tcop, and libpq. Tom Lane, Robert Haas and Jacob Champion joined only the negotiation debate, not the v5 design.
- Priority: fix design first; audience a sizeable minority; review cost high. Demand comes from JDBC holdable result sets, postgres_fdw interest and one out-of-memory field report. The series is about 100 KiB across postgres.c, backend_startup.c and libpq, and it fixes a wire format, so the open design question from message #52 should be settled before code review.

---

[roast-full.md](roast-full.md) has 1 more objection(s) that held, 9 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

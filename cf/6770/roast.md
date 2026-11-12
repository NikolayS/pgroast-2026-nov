# Design pre-review: [BUG] DELETE/UPDATE more than one row in partitioned foreign table

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6770](https://commitfest.postgresql.org/patch/6770/); [thread](https://www.postgresql.org/message-id/flat/20250718175314.4513c00a%40karst), 37 of 37 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). The goal is not in question: the bug silently changes wrong rows on the remote side, and messages #1 and #6 report production hits. The surviving objections are all fixable level 2 points. The decisive one is maintenance-fit-1: v5-0001 passes an attribute number outside min_attr..max_attr through the planner, and no planner committer has reviewed it. prior-attempts-1 leaves the stable-branch fix open, which the list must settle before commit.

Not checked: Without the code, I could not check how v5-0001 handles every planner array indexed by attno, or whether it extends the existing row-identity path or duplicates it. I also could not check the ABI effect on planner structs.

## Prior attempts

None found before 2025-07-18.

## Objections a reviewer is likely to raise

### 1. Out-of-range pseudo attribute number adds a silent planner invariant

A later planner patch indexes per-attribute arrays such as attr_needed or attr_widths by attno, assuming the range min_attr..max_attr. It does not handle RemoteTableOidAttributeNumber. On a production build, postgres_fdw UPDATE/DELETE on a remote partitioned table then reads out of bounds or plans wrongly, and nobody notices until a user reports it.

v5-0001 touches costsize.c, initsplan.c, setrefs.c, relnode.c and ruleutils.c so that one attribute number outside the normal range can pass through. Message #36 turns some existing Asserts into runtime errors, but only at places that already had a check. Message #27 asks for a planner expert and message #34 says the reviewer can no longer follow the edge cases. No planner committer has reviewed v5-0001 in the thread, although message #29 says Robert Haas suggested the approach off-list.

**Question:** Which planner code paths assume attno lies within min_attr..max_attr, and what makes a future patch that misses the pseudo-column fail in tests instead of in the field?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#27](https://www.postgresql.org/message-id/CAKZiRmwr4zEZ2rFWv54GVh7FfAstQTdbA0HAv2M4E4dpLsX1-Q%40mail.gmail.com), [#29](https://www.postgresql.org/message-id/CAER375NQEqXLQ5eqFJ8QpnpnYKCHeyi8YhG6CdAm6m%3DQGc6unw%40mail.gmail.com), [#34](https://www.postgresql.org/message-id/CAKZiRmzpABcNuk%2BZrhmV3%3DGtaA2S%3DC3%3DvKw_ggXCUwQUv8D4ag%40mail.gmail.com), [#36](https://www.postgresql.org/message-id/CAER375NOcbQyDGxfRDM0wuRmJx8j5TBmkXtC0QY0j2Doye%3DHSA%40mail.gmail.com); `maintenance-fit-1`</sub>

### 2. Same non-back-patchable shape that reportedly stalled the earlier fix

A user on Postgres 16 to 19 runs a non-direct UPDATE on a foreign table over a remote partitioned table. Rows with a shared ctid in other partitions are silently changed. The user finds out later from wrong data, because v5 changes the planner and only lands in 20.

Message #1 reports that the 2018 fixes lost attention partly because they needed new FDW features and could not be backported. Message #9 says this line of work builds on that earlier design and is not back-patchable. v5 keeps that shape: planner changes that message #34 calls invasive, and varnullingrels use that limits it to 16 and later (message #27). Etsuro's remotely_inherited patch (messages #9, #10, #14) is the stable-branch answer, but the thread does not say whether it is still planned alongside v5.

**Question:** What fix goes to stable branches, and is the remotely_inherited patch still planned to be committed alongside v5?

<sub>fixable, level 2, backpatchability; messages [#1](https://www.postgresql.org/message-id/20250718175314.4513c00a%40karst), [#9](https://www.postgresql.org/message-id/CAPmGK166P%2Bngd2ehady%3D_f-L4MePgBdBNxN5gi5_gSAfmV82QA%40mail.gmail.com), [#13](https://www.postgresql.org/message-id/aij4NSMQYpQxtRMF%40paquier.xyz), [#14](https://www.postgresql.org/message-id/CAPmGK15vOW11uodFuSEDcYaUxLX4egOgwGq4BZasgG_pNOa_7Q%40mail.gmail.com), [#27](https://www.postgresql.org/message-id/CAKZiRmwr4zEZ2rFWv54GVh7FfAstQTdbA0HAv2M4E4dpLsX1-Q%40mail.gmail.com), [#34](https://www.postgresql.org/message-id/CAKZiRmzpABcNuk%2BZrhmV3%3DGtaA2S%3DC3%3DvKw_ggXCUwQUv8D4ag%40mail.gmail.com); `prior-attempts-1`</sub>

### 3. Chained foreign tables keep silent wrong-row updates behind a protective-looking plan

An administrator points a foreign table at a remote foreign table that sits on a partitioned table. EXPLAIN VERBOSE shows "WHERE ctid = $1 AND tableoid = $2", so the administrator trusts the result. Non-direct UPDATE then changes rows in other partitions, and nothing reports it until someone audits the data.

Message #28 shows this case still corrupts data under v5, and that the new condition makes the plan look protected. Message #29 declines a remote relkind check because of an extra round trip per query and documents the gap instead. A documentation note does not reach users who never read it, and the rest of the patch changes this behaviour from silently wrong to correct. The relkind could be checked only on the non-direct UPDATE/DELETE path, once per foreign table and connection, at much lower cost than the patch assumes.

**Question:** Why could postgres_fdw not check the remote relkind once per foreign table and connection, only for non-direct UPDATE/DELETE, and refuse when it is a foreign table?

<sub>fixable, level 2, design; messages [#28](https://www.postgresql.org/message-id/CA%2BbCEdCo-w5_34AAm6BTZ0PucNg3sycpM8-7L6Qn_KCwxSR30Q%40mail.gmail.com), [#29](https://www.postgresql.org/message-id/CAER375NQEqXLQ5eqFJ8QpnpnYKCHeyi8YhG6CdAm6m%3DQGc6unw%40mail.gmail.com), [#30](https://www.postgresql.org/message-id/179008834802.2666470.8606898268993687598%40gmail.com), [#37](https://www.postgresql.org/message-id/179371008812.1245613.18345043610354071644%40gmail.com); `standards-compat-2`</sub>

### 4. New FDW planner API is shaped for postgres_fdw and undocumented

An author of another FDW, for example one for a remote sharded store, sees RemoteTableOidAttributeNumber and uses it for a different row identity. fdwhandler.sgml says nothing about its rules, so the FDW gets whole-row or setrefs errors like those in message #34. Core then has to keep the undocumented behaviour stable in every later release.

The material shows one fixed attribute number named for the remote tableoid, plus the rule that the scan must go through fdw_scan_tlist (message #21). The proposal does not say whether a second pseudo-column, or an FDW other than postgres_fdw, is supported. It also does not add any documentation for FDW authors. Once released, this becomes extension-facing behaviour that the project must keep working.

**Question:** Is v5-0001 meant as a general FDW API, and if so, where are its rules documented and how does an FDW add more than one row-identity column?

<sub>fixable, level 2, ABI / extension API; messages [#21](https://www.postgresql.org/message-id/CAER375N8yyZhdvJMMU1fx2ZUcQmMbgj%3D8Vf9Qj3Qv_KFeczFZQ%40mail.gmail.com), [#34](https://www.postgresql.org/message-id/CAKZiRmzpABcNuk%2BZrhmV3%3DGtaA2S%3DC3%3DvKw_ggXCUwQUv8D4ag%40mail.gmail.com), [#36](https://www.postgresql.org/message-id/CAER375NOcbQyDGxfRDM0wuRmJx8j5TBmkXtC0QY0j2Doye%3DHSA%40mail.gmail.com); `maintenance-fit-2`</sub>

## Questions for the author

1. Which planner code paths index per-attribute arrays by attno within min_attr..max_attr, and what makes a later patch that misses the pseudo-column fail in tests?
2. What fix goes to stable branches, and is the remotely_inherited option still planned alongside v5?
3. Why can postgres_fdw not check the remote relkind once per foreign table and connection, only for non-direct UPDATE/DELETE, and refuse a remote foreign table?
4. Is v5-0001 meant as a general FDW API, and if so, where are its rules documented and can an FDW add more than one row-identity column?
5. Has a committer who knows the planner agreed to review v5-0001, given that message #29 says the approach was suggested off-list?

## Touch points

- **ABI / extension API:** v5-0001 adds planner support for FDW row-identity pseudo-columns and the attribute number RemoteTableOidAttributeNumber.
- **planner behaviour change:** v5-0001 changes costsize.c, initsplan.c, setrefs.c and relnode.c, and turns planner Asserts into runtime errors.
- **backpatchability:** v5 relies on varnullingrels, so it cannot go below Postgres 16, and the thread calls its planner changes too invasive to back-patch.
- **maintenance burden ("who maintains this in 10 years"):** Later planner patches must handle one attribute number outside min_attr..max_attr, and nothing shown fails loudly if they miss it.
- **performance on the hot path:** Every non-direct UPDATE/DELETE in postgres_fdw gains a tableoid fetch and condition; with 100 remote partitions per-row commands cost about ten times more.

## Who would commit it, and when

- Owner: silent. v5-0001 needs a committer who knows the planner's per-attribute bookkeeping and setrefs; none has reviewed it in the thread. Etsuro Fujita takes part and knows postgres_fdw, but favoured the remotely_inherited option.
- Priority: needs champion; audience a sizeable minority; review cost high. The bug silently corrupts remote data and has two independent production reports, so it is worth attention now. The entry is Ready for Committer by a tester, but v5-0001 changes several planner files and no planner committer has reviewed it.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 2 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

# Design pre-review: Use a bulk-write buffer access strategy for TOAST inserts

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7470](https://commitfest.postgresql.org/patch/7470/); [thread](https://www.postgresql.org/message-id/flat/atHWFmxAz1KuKzqm%40bdtpg), 2 of 2 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). demand-1 is decisive: the thread shows one earlier report and no numbers for the eviction today or for the effect of the patch. In message #2 a committer who knows the buffer code says TOAST inserts should not use strategies, because later reads of the evicted TOAST data are random IO. That claim has no reply yet. The list has to agree that the change is wanted before the design of the four patches is reviewed.

Not checked: The code was not read, so how the one-relation rule is enforced and how the TOAST state is cleaned up on error could not be checked. The design level was not evaluated, because the review stopped at level 1b.

## Prior attempts

- [INSERT SELECT: BulkInsertState and table_multi_insert](https://www.postgresql.org/message-id/flat/20200508072545.GA9701@telsasoft.com) (related; Returned with feedback, 2021-11-24). "Since the thread has stalled, maybe that's the best course of action here." ([message](https://www.postgresql.org/message-id/AF742DB8-2992-4826-B85B-1FC36BBEF3A9%40yesql.se)) Different now: P2 extended BulkInsertState rings to INSERT and INSERT SELECT. This proposal adds a ring for TOAST inserts in paths that already use a bulk state.

## Objections a reviewer is likely to raise

### 1. No measurement shows the TOAST eviction problem in practice

A committer weighs a four-patch series that changes GetBulkInsertState() for every caller. The only evidence of harm is one 2022 report with no numbers. If the eviction is rare or small, the project takes on the API churn and the new strategy path for little gain.

Message #1 cites a single earlier report from Justin Pryzby and gives no workload, buffer-hit figures or pg_stat_io numbers. No other user in the thread asks for the change. Using BAS_BULKWRITE for the main relation is established practice, so the idea is plausible. But the gain for TOAST-heavy loads has not been shown against a workload where it matters.

**Question:** On a realistic COPY or VACUUM FULL of TOAST-heavy data, how much of shared_buffers is evicted today, and how much does the patch change it?

<sub>fixable, level 1b, demand; messages [#1](https://www.postgresql.org/message-id/atHWFmxAz1KuKzqm%40bdtpg); `demand-1`</sub>

## Questions for the author

1. On a COPY or VACUUM FULL of TOAST-heavy data, how much of shared_buffers is evicted today, and how much does the patch change that?
2. For a load-then-query workload, do the later random reads of TOAST data cost more than the cache the ring saves for other sessions?
3. Are 0001 and 0002 a step toward the tableam-owned bulk state proposed in message #2, or would that design undo this API change?
4. Is the series meant for master only, given that the CommitFest entry is tagged Bugfix?
5. How must out-of-core table AMs and extensions that call GetBulkInsertState() adapt to the caller-owned strategy and the one-relation rule?

## Touch points

- **ABI / extension API:** 0001 and 0002 change the GetBulkInsertState() signature in heapam.h and bind each state to one relation.
- **backpatchability:** The CommitFest entry is tagged Bugfix, but the series changes exported functions and targets version 20.
- **maintenance burden ("who maintains this in 10 years"):** Each BulkInsertState gains a TOAST state, and heap rewrites create a ring and state for TOAST on first use.
- **performance on the hot path:** TOAST writes in COPY, CREATE TABLE AS, matview refresh, table rewrites and VACUUM FULL move to a small ring, with no switch.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** see objection demand-1: No measurement shows the TOAST eviction problem in practice

## Who would commit it, and when

- Owner: qualified committer engaged. Andres Freund takes part in message #2 and disagrees with both the TOAST ring and the API direction. The owner must know buffer access strategies and the tableam insert API.
- Not now, show demand first: A benchmark of a TOAST-heavy bulk load that shows the eviction of other sessions' buffers today, the effect of the patch, and the cost of reading the loaded TOAST data afterwards. More than one field report of the eviction would also help.
- Priority: needs design rethink; audience a sizeable minority; review cost medium. Bulk loads of wide values are common, but the benefit is not measured and message #2 disputes it. The four patches, about 60 KiB, touch heapam, COPY, the executor and TOAST code, and message #2 prefers a tableam-owned bulk state.

---

[roast-full.md](roast-full.md) has 2 more objection(s) that held, 6 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

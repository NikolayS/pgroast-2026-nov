# Design pre-review: pgstat: Flush some statistics within running transactions, take 2

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6781](https://commitfest.postgresql.org/patch/6781/); [thread](https://www.postgresql.org/message-id/flat/CAA5RZ0uA-4qcD3%2B2hjcE_-zQUBhvWf5foPM2vzYneFKrJLsBDQ%40mail.gmail.com), 44 of 44 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No surviving objection is fatal, and none is at level 1. Two fixable level 2 points remain. replication-ondisk-1: a custom stats callback ported with the smallest change double counts without any error. standards-compat-2: the existing pg_stat_force_next_flush() takes on immediate-flush behaviour under a name that says "next". A committer who knows pgstat has to decide both; neither stops the idea.

Not checked: The code was not read. The retained-entry and baseline logic, the requeue of entries during a pass, and error safety of a flush inside a transaction could not be judged from the prose.

## Prior attempts

- [Flush some statistics within running transactions](https://www.postgresql.org/message-id/flat/aWTVEycKj7Qh%2FSXH@ip-10-97-1-34.eu-west-3.compute.internal) (similar idea; Returned with feedback, 2026-03-25). "This design requires enabling a new signal with a signal handler in a lot of processes that did not do that. Enabling timeouts in a bunch of new processes, while claiming it is fine to do, sounds like something we should rather be careful about." ([message](https://www.postgresql.org/message-id/aZ-_AkG3cmeTbdFn%40paquier.xyz)) Different now: P1 used a periodic timeout and later a stats_flush_interval GUC across backends and auxiliary processes. This proposal has no timer, no new signal handler and no GUC: the calling backend flushes on demand, which is the direction the P1 objector suggested according to message #1.

## Objections a reviewer is likely to raise

### 1. Custom stats callbacks can silently double count after the API change

An extension with a custom variable-numbered kind is ported by adding the xact_boundary argument and returning PGSTAT_FLUSH_DONE. When a user calls pg_stat_force_next_flush() inside a transaction, the retained pending entry is merged again at the next flush or at commit. The inflated counters reach shared memory and the stats file written at shutdown, and nothing reports an error.

The patch keeps pending entries allocated mid-transaction (message #4), so every callback must now clear or baseline what it published. The rule exists only as a comment on flush_pending_cb (message #12). The signature change makes the build fail, but the minimal port that fixes the build is the one that double counts. Message #3 reordered the enum to avoid silent misbehavior in callbacks, but the retained-pending rule has no such guard.

**Question:** What makes a ported custom callback that does not clear or baseline its pending data fail visibly, for example in an assert-enabled build, rather than double count?

<sub>fixable, level 2, ABI / extension API; messages [#3](https://www.postgresql.org/message-id/amyuOVEGvwOs5J0Y%40bdtpg), [#4](https://www.postgresql.org/message-id/CAA5RZ0v8REO_RHHBcTPdmoO6a2fkT%3Dxp4iKG-%3DLh_FvmG%3DSzgw%40mail.gmail.com), [#9](https://www.postgresql.org/message-id/anSCUOyzesTT%2FqwR%40bdtpg), [#12](https://www.postgresql.org/message-id/CAA5RZ0u%2Bmb3H7in2%3DZjZE4P6Aujm%3Doph1KOTQLtoOZD2phyOvw%40mail.gmail.com); `replication-ondisk-1`</sub>

### 2. Existing function and C API get new semantics under the old name

An extension or test harness already calls pgstat_force_next_flush() or pg_stat_force_next_flush() inside a transaction, from a hook or from a function, expecting only a deferred flush. After upgrade, the call runs every flush callback on the spot, takes shared stats locks, and can raise an ERROR there. The caller sees this only when it fails.

v1 added a separate pg_stat_report_anytime(). v2 moved the immediate flush into the existing function, whose name still says the flush happens at the next occasion (message #2). Michael documented the function as end-of-transaction and test-oriented in message #36, and v18 changes that documented contract in the same cycle. The function also changes from PARALLEL RESTRICTED to PARALLEL UNSAFE, which can change plans for queries that already call it.

**Question:** Why should the existing deferred-flush function take on immediate-flush semantics rather than a new function or an option, given current C and SQL callers?

<sub>fixable, level 2, design; messages [#1](https://www.postgresql.org/message-id/CAA5RZ0uA-4qcD3%2B2hjcE_-zQUBhvWf5foPM2vzYneFKrJLsBDQ%40mail.gmail.com), [#2](https://www.postgresql.org/message-id/CAA5RZ0u84eMFeFWMaEo0D84ed3jF_RY0%3DRWD8tDwQWsNsU1qvA%40mail.gmail.com), [#8](https://www.postgresql.org/message-id/CAA5RZ0uqJYn9SXeZuA_pmr1E5xpTV45Sw7TX8pqHj44o_0BY4w%40mail.gmail.com), [#36](https://www.postgresql.org/message-id/aqEONVuLhgBiqHW6%40paquier.xyz), [#39](https://www.postgresql.org/message-id/aqIxIveLyADWOwlb%40paquier.xyz); `standards-compat-2`</sub>

## Questions for the author

1. What makes a ported custom callback that returns PGSTAT_FLUSH_DONE without clearing or baselining its pending data fail visibly, for example in an assert-enabled build?
2. Why does the existing pg_stat_force_next_flush() take on the immediate flush, instead of a new function or an option as in v1?
3. Where is the rule that custom flush callbacks must avoid cyclic dependencies documented for extension authors?
4. What does one transaction cost when it forces many flushes over hundreds of retained entries?
5. Is dropping the flush of another backend by PID, including auxiliary processes, final for this patch?

## Touch points

- **catalog changes:** pg_proc.dat changes pg_stat_force_next_flush() to PARALLEL UNSAFE and needs a catversion bump.
- **ABI / extension API:** flush_pending_cb gets an xact_boundary argument and an enum return type; flush_static_cb gets a transaction-boundary flag.
- **SQL-standard conflict or non-standard syntax:** see objection standards-compat-2: Existing function and C API get new semantics under the old name
- **planner behaviour change:** The PARALLEL UNSAFE marking stops parallel plans for queries that call pg_stat_force_next_flush().
- **maintenance burden ("who maintains this in 10 years"):** Relation, index and function stats keep a flushed baseline with delta logic, and custom kinds carry a new no-double-merge rule.
- **performance on the hot path:** The patch touches the pending-stats flush path that every backend runs at transaction end.

## Who would commit it, and when

- Owner: qualified committer engaged. Michael Paquier committed the counter split and the documentation of pg_stat_force_next_flush() from this thread. The thread does not say whether he accepts the main patch design.
- Priority: proceed; audience a sizeable minority; review cost high. The patch is a prerequisite for the pg_stat_statements scalability work and serves users and tests that want current numbers from a long transaction. The main patch is about 98 KiB across all pgstat kinds and has had no new review since early September, so it needs a committer's design decision on the two open points.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 5 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

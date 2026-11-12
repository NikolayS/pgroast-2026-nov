# Design pre-review: Don't use the deprecated and insecure PQcancel in our frontend tools anymore

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6314](https://commitfest.postgresql.org/patch/6314/); [thread](https://www.postgresql.org/message-id/flat/DEY0N7FS8NCU.1F7QXGG37MQ09%40jeltef.nl), 32 of 32 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No surviving objection questions the goal or the overall design. The only surviving objection, prior-attempts-1, is a fixable portability point: v13-0003 uses pselect() behind an existence check, and [P12] records that some platforms shipped a pselect() that did not work. A committer who knows the frontend signal and portability code can settle that, and the 1 second select() fallback already exists.

Not checked: Without the code, the review could not check thread safety of siglongjmp() out of input waits, the wait in SetCancelConn() and ResetCancelConn() for an in-flight cancel, or the pselect() signal masking. Only 3 of 32 messages were visible, so later answers may exist.

## Prior attempts

- [Revert "Use pselect(2) not select(2), if available, to wait in postmaster's loop."](https://www.postgresql.org/message-id/flat/9696.1493072081@sss.pgh.pa.us) (related; Reverted, 2017-04-24). "Buildfarm results suggest that some platforms have versions of pselect(2) that are not merely non-atomic, but flat out non-functional." Different now: This uses pselect() only in frontend wait_on_slots(), not in the postmaster, and falls back to select() with a 1 second timeout. The thread does not discuss how a broken pselect() would be detected.
- [pgbench: allow to cancel queries during benchmark](https://www.postgresql.org/message-id/flat/20230626224628.61f372f25cde3747f164a4c6@sraoss.co.jp) (related; Returned with feedback, 2024-10-07). "Due to commit 61461a300c1c, this patch needs to be reworked." ([message](https://www.postgresql.org/message-id/202403301335.jcac6f3tkf2f%40alvherre.pgsql)) Different now: That patch added Ctrl+C cancel for all pgbench clients and stopped because it had to move to the new cancel API. This proposal makes that move for fe_utils users but keeps one global cancel connection.

## Objections a reviewer is likely to raise

### 1. pselect() was reverted before because some platforms ship a broken one

On a platform whose pselect() exists but does not work, configure detects it and wait_on_slots() uses it. Then vacuumdb -j or reindexdb -j can miss Ctrl+C or fail while it waits on its slots. The buildfarm or a user on that platform finds out after commit.

[P12] records that pselect() in the postmaster loop was reverted because some platforms' versions "are not merely non-atomic, but flat out non-functional". v13-0003 makes the first use of pselect() since then, behind a plain configure check (message #22 notes it is unused today). The thread does not say whether those platforms are still supported or how a broken pselect() would be detected. The 1 second select() fallback already exists, so a narrower or opt-in use is a possible way out.

Precedent: [Revert "Use pselect(2) not select(2), if available, to wait in postmaster's loop."](https://www.postgresql.org/message-id/flat/9696.1493072081@sss.pgh.pa.us) (Reverted, 2017-04-24)

**Question:** Which platforms had the broken pselect() behind the earlier revert, and are any of them still supported or in the buildfarm today?

<sub>fixable, level 2, prior rejection; messages [#22](https://www.postgresql.org/message-id/a4c6998e-d026-4eed-bb90-c9ad121aaed0%40iki.fi), [#27](https://www.postgresql.org/message-id/DL519QREJ4P1.2Z9UAEYVCTNQ2%40jeltef.nl); `prior-attempts-1`</sub>

## Questions for the author

1. Which platforms had the broken pselect() behind the revert in [P12], and are any of them still supported or in the buildfarm today?
2. What do the tools do when the cancel thread cannot be created, or on a platform without threads?
3. Is a second call to setup_cancel_handler() now guarded against, as raised in review?
4. Has cancel through a bad network been tested for tools other than psql, such as pg_dump with parallel workers?
5. Is the plan master-only, or is a backport to PG17 and PG18 still wanted?

## Touch points

- **wire protocol:** The tools now send cancel requests over an encrypted connection through the existing libpq cancel API; the protocol itself does not change.
- **ABI / extension API:** The fe_utils cancel.h API changes, including a new setup_cancel_handler() with two callbacks, and pthread-win32.c moves to src/port.
- **psql UX conventions:** psql -f scripts exit with code 3 on Ctrl+C at input prompts, and the prompt can stay blocked on a stuck cancel connection.
- **security / privileges:** The cancel key no longer travels in clear text when the session uses SSL or GSS.
- **backpatchability:** Message #1 asks about a backport to PG17 and PG18 as a security fix; the thread has not settled it.
- **maintenance burden ("who maintains this in 10 years"):** Every frontend tool gains a cancel thread, a self-pipe and a new configure and meson check for pselect().

## Who would commit it, and when

- Owner: qualified committer engaged. Peter Eisentraut committed v13-0001 and is listed as reviewer; Heikki Linnakangas reviewed the design in depth. The owner needs to know frontend signal handling, threads and platform portability.
- Priority: proceed; audience most users; review cost high. Every SSL or GSS user of the frontend tools is affected, and committers are already engaged with one part committed. The remaining series changes signal and thread handling in many tools, so review is costly and the new cancel path needs ongoing care.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 5 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

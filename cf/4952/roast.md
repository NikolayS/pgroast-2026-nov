# Design pre-review: Use WALReadFromBuffers in more places

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 4952](https://commitfest.postgresql.org/patch/4952/); [thread](https://www.postgresql.org/message-id/flat/CALj2ACVfF2Uj9NoFy-5m98HNtjHpuD17EDE9twVeJng-jTAe7A%40mail.gmail.com), 31 of 31 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No objection survived at level 1: the problem is stated, and several people tested or reproduced the gains. Two fixable design points remain at level 2. replication-ondisk-1 asks what ties a page read from WAL buffers to the requested timeline, and the seg.ws_tli fix is not yet posted. maintenance-fit-1 asks for one shared helper instead of four copies of the buffer-then-file logic. A committer who knows the WAL reader code can settle both.

Not checked: The code was not read, so the segment-close rule, the planned seg.ws_tli fix and the behaviour when the WAL writer reuses pages during a read could not be checked. The timeline question rests on the thread, not on the patch.

## Prior attempts

None found before 2024-04-24.

## Objections a reviewer is likely to raise

### 1. Buffer-only reads leave seg.ws_tli unset, and the timeline invariant is not stated

A logical slot is synced to a standby, the standby is promoted, and decoding crosses the timeline switch. Pages served from WAL buffers leave seg.ws_tli at 0 or stale, so a later file read or a page check uses the wrong timeline. The user sees a wrong error or bad decoding only after failover.

Message #30 shows that the page_read contract in xlogreader.h requires the callback to set seg.ws_tli. Message #31 agrees the contract is broken and says it only affects error reporting and validation, but no fixed version is posted. For read_local_xlog_page() callers, XLogReadDetermineTimeline() picks the timeline per page, yet the proposal does not say why a buffer read cannot return pages from a timeline other than the one the reader asked for. That gap matters now that logical decoding runs across promotion.

**Question:** With seg.ws_tli fixed, what guarantees that a buffer-served page belongs to the timeline the reader requested, including reads that straddle a promotion?

<sub>fixable, level 2, WAL / replication / standby; messages [#30](https://www.postgresql.org/message-id/OS7PR01MB183178A50F430AC19E5CEDC60F5602%40OS7PR01MB18317.jpnprd01.prod.outlook.com), [#31](https://www.postgresql.org/message-id/CALj2ACV3mL9BP0VPP%2BqA%3DU_y%3DD8vhZ_1%3DDgxr%3DYvQwvf%3DdmUAQ%40mail.gmail.com); `replication-ondisk-1`</sub>

### 2. Buffer-then-file read logic copied into four page_read callbacks

A later patch changes how WALRead() tracks segments or sets seg.ws_tli, and updates only some of the physical walsender, logical walsender, read_local_xlog_page_guts() and summarizer copies. A copy that was missed then reuses a stale descriptor or reports a wrong TLI. Users find out through an "unexpected pageaddr" error during decoding, or a wrong error message, once tests stop reaching the file-read fallback.

Each caller gets its own WALReadFromBuffers() plus WALRead() sequence, its own segment-close rule and its own ws_tli handling, and the author declined a shared helper (message #5, #20). The thread has already shown misses of exactly this kind: an unconditional close that Michael flagged (message #21), the dropped retry in the physical path (message #20), and the unset ws_tli (message #30, #31). Message #25 shows that tests stop exercising the file path once buffers serve every read, so a missed update would not fail loudly.

**Question:** Can the buffer read, the segment close and the ws_tli rule live in one xlogreader-level helper, so that future WALRead() changes need one edit and fail loudly if missed?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#5](https://www.postgresql.org/message-id/CALj2ACXa-2eEHHaNRwjcF1k9rtH%3DEJrWvbGJkucdSOD3zP-OUw%40mail.gmail.com), [#20](https://www.postgresql.org/message-id/CALj2ACUo2Kqy8ovFwSr1Cr7hkTLGQo4U8QFDQF9nW4d9ADGfXg%40mail.gmail.com), [#21](https://www.postgresql.org/message-id/appZ6saaV5wV5UA4%40paquier.xyz), [#25](https://www.postgresql.org/message-id/CALj2ACUMtx%2B3hd7mO8Gb5kewpvwQ9OLEPPSC1fC2r500k5_CaQ%40mail.gmail.com), [#30](https://www.postgresql.org/message-id/OS7PR01MB183178A50F430AC19E5CEDC60F5602%40OS7PR01MB18317.jpnprd01.prod.outlook.com), [#31](https://www.postgresql.org/message-id/CALj2ACV3mL9BP0VPP%2BqA%3DU_y%3DD8vhZ_1%3DDgxr%3DYvQwvf%3DdmUAQ%40mail.gmail.com); `maintenance-fit-1`</sub>

## Questions for the author

1. With seg.ws_tli set, what guarantees that a page read from WAL buffers belongs to the timeline the reader requested, including reads that straddle a promotion?
2. Can the buffer read, the segment close and the seg.ws_tli rule live in one helper shared by the logical walsender, read_local_xlog_page() and the WAL summarizer?
3. What does the extra WALReadFromBuffers() attempt cost when a walsender lags and almost every read misses WAL buffers, as Hayato Kuroda asked?
4. Is there a measurement with the default wal_buffers and WAL direct IO off that shows no regression?
5. Which test still reaches the WALRead() file fallback and the segment-close path once WAL buffers serve most reads?

## Touch points

- **WAL / replication / standby:** Logical walsenders, read_local_xlog_page() callers and the WAL summarizer change how they read WAL, including across timeline switches.
- **ABI / extension API:** read_local_xlog_page() is a page_read callback that extensions such as pg_walinspect use, and its seg.ws_tli behaviour changes.
- **maintenance burden ("who maintains this in 10 years"):** Four page_read callbacks each carry their own buffer read, segment-close rule and seg.ws_tli handling.
- **performance on the hot path:** Every WAL page read by walsenders and local readers now tries WAL buffers first, including when a lagging reader mostly misses.

## Who would commit it, and when

- Owner: qualified committer engaged. Michael Paquier committed the pg_stat_io part and reviewed the rest; the owner must know xlogreader page_read callbacks and timeline handling in walsenders.
- Priority: proceed; audience niche; review cost medium. The patches are small and a qualified committer is engaged, but the throughput gain is measured only with WAL direct IO, a developer setting today. The duplicated read logic and the timeline question need care in the WAL reader code, which sets the burden and review cost at medium.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

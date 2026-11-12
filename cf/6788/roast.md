# Design pre-review: COPY TO BLACKHOLE / pg_dump -j -Fb

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6788](https://commitfest.postgresql.org/patch/6788/); [thread](https://www.postgresql.org/message-id/flat/CAKZiRmzorOYZH-CvyN98%2BKz3aGdJW5Kj-MirtM0tmjYO3PtASQ%40mail.gmail.com), 6 of 6 messages read; no code read.

## Verdict

**LIKELY-REJECT** (confidence: medium). The proposal fails at level 1b on maintenance-fit-1 and already-exists-1. COPY TO '/dev/null' already runs the same check, and the thread gives no reason for a permanent COPY keyword beyond 11-14% less CPU. The author accepts in message #6 that pg_dump is the wrong place and considers a client-only pg_amcheck option, which would be a different proposal. Confidence is medium because v1 has no documentation and the pg_amcheck alternative has no design.

Not checked: The level 2 objections on syntax, privileges, progress reporting and per-row cost were not evaluated because level 1b fails. The code was not read, so the behaviour for file and program targets and the size of the CPU saving are unverified.

## Prior attempts

None found before 2026-05-20.

## Objections a reviewer is likely to raise

### 1. A COPY destination for corruption checks does not belong in COPY

Postgres adds BLACKHOLE as a COPY keyword and destination to gain about 14% over COPY TO '/dev/null'. Users then rely on COPY as a supported corruption checker. If amcheck later gains detoast checks, the syntax still has to stay for compatibility.

Message #2 says a COPY destination that moves no data is counter-intuitive. Message #4 notes that '/dev/null' or a hashtext query already does the same work, and in message #5 Michael Paquier asks for better corruption tools instead. In message #6 the author suggests a client-only pg_amcheck option that would not touch core COPY, which fits existing practice better.

**Question:** What does COPY TO BLACKHOLE offer that a pg_amcheck option running COPY TO '/dev/null' or a full-row query in parallel cannot offer?

<sub>fatal, level 1b, doesn't fit what Postgres is about (extension / external tool / already solved); messages [#2](https://www.postgresql.org/message-id/CAFTkQVK0uaGV%3DtL2EtjWrCY3Rz7FRHLUegn4EdG4VKRN6Yv%3DYQ%40mail.gmail.com), [#4](https://www.postgresql.org/message-id/CAMT0RQR5_CQ3NsVvmHePMqF2etQoJSeFrTiMcCvXVkMHT8rh%3Dw%40mail.gmail.com), [#5](https://www.postgresql.org/message-id/amwMzxnv4I9VoSPc%40paquier.xyz), [#6](https://www.postgresql.org/message-id/CAKZiRmz6Nfwj0BS%3DdizC1%3DjRoGt6Yu990fz8T4f0bxPJmEn_SA%40mail.gmail.com); `maintenance-fit-1`</sub>

### 2. COPY TO '/dev/null' already gives the same check

Postgres adds a second COPY TO destination that forms every row and discards it, as COPY TO '/dev/null' already does. Every later change to the COPY TO row path has to keep the BLACKHOLE branch working. The only gain is about 11-14% on a check that users run rarely.

Message #1 shows that COPY to '/dev/null' runs the same check, and message #6's pglz reproducer shows it catches the same TOAST corruption that BLACKHOLE targets. Message #4 makes the same point, and the thread never answers it. Users without pg_write_server_files can run COPY TO STDOUT with output sent to /dev/null on the client. Parallel runs need no core change, because N sessions or a pg_amcheck-style client driver can each run COPY to '/dev/null', and message #6 itself names the client-only option "--copy-to-null".

**Question:** Which check can BLACKHOLE do that COPY TO '/dev/null' or COPY TO STDOUT from a parallel client cannot, beyond about 12% less CPU?

<sub>fixable, level 1b, doesn't fit what Postgres is about (extension / external tool / already solved); messages [#1](https://www.postgresql.org/message-id/CAKZiRmzorOYZH-CvyN98%2BKz3aGdJW5Kj-MirtM0tmjYO3PtASQ%40mail.gmail.com), [#4](https://www.postgresql.org/message-id/CAMT0RQR5_CQ3NsVvmHePMqF2etQoJSeFrTiMcCvXVkMHT8rh%3Dw%40mail.gmail.com), [#6](https://www.postgresql.org/message-id/CAKZiRmz6Nfwj0BS%3DdizC1%3DjRoGt6Yu990fz8T4f0bxPJmEn_SA%40mail.gmail.com); `already-exists-1`</sub>

## Questions for the author

1. What does COPY TO BLACKHOLE offer that a pg_amcheck option running COPY TO '/dev/null' or COPY TO STDOUT in parallel sessions cannot, beyond 11-14% less CPU?
2. Where do the hot-cache savings come from, given that COPY to '/dev/null' already discards the output?
3. If the pg_amcheck route is chosen, what is the option name, and does it need any change to core COPY?
4. Which corruption does a COPY-based check find that verify_heapam() with check_toast, index checks and pg_catcheck do not cover?
5. What does the new per-row branch cost normal COPY TO STDOUT and COPY TO file, in text and binary format, on narrow and wide tables?

## Touch points

- **wire protocol:** COPY TO BLACKHOLE sends no row data to the client, and the proposal does not say which COPY messages the client receives.
- **ABI / extension API:** Patch 0001 changes copy.h and parsenodes.h and touches the test_copy_callbacks module.
- **SQL-standard conflict or non-standard syntax:** Patch 0001 adds BLACKHOLE to kwlist.h and to the COPY TO grammar.
- **pg_dump / restore round-trip:** Patch 0002 adds an archive format, -Fb, that writes nothing and cannot be restored.
- **security / privileges:** The proposal does not say which privilege COPY TO BLACKHOLE needs, while COPY to '/dev/null' needs pg_write_server_files.
- **maintenance burden ("who maintains this in 10 years"):** A new keyword, a new COPY destination and a new pg_dump archive format must be kept working after commit.
- **performance on the hot path:** The patch adds a branch to the per-row path of every COPY TO, with no benchmark for normal COPY TO.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** COPY to '/dev/null' from parallel client sessions already gives the same check with no core change.

## Who would commit it, and when

- Owner: silent. It needs someone who knows the COPY TO row path and, for the alternative, pg_amcheck. Michael Paquier raised the COPY path cost in message #5, but nobody has offered to take the patch.
- Priority: withdraw and resubmit; audience niche; review cost low. The goal of a parallel logical check with no disk use is valid for large databases, and the two patches are small. Reviewers and the author agree the COPY and pg_dump placement is wrong, so a client-only pg_amcheck option needs its own proposal.

---

[roast-full.md](roast-full.md) has 3 more objection(s) that held, 3 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

# Design pre-review: Halve memory allocation for toast_flatten_tuple

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7362](https://commitfest.postgresql.org/patch/7362/); [thread](https://www.postgresql.org/message-id/flat/CALT9ZEG8W0-xVCYQmxebYQ55fE-SJmKOTrBqgMbcVekhGNMaAw%40mail.gmail.com), 3 of 3 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). No surviving objection is fatal, and no level fails. The problem is stated, and message #2 independently confirms the memory cut. Two fixable design points remain: the stub header must predict what detoast_external_attr() returns (maintenance-fit-1), and every heap_form_tuple() caller now takes the _attr path without CPU numbers (maintenance-fit-2). demand-1 is open but does not block review, because the thread shows no field report either way.

Not checked: Without the code, nobody can confirm that the stub header always matches the detoasted value, or that a mismatch raises an error before the write. The compiled cost of the extra arguments in the shared fill path is also unknown.

## Prior attempts

None found before 2026-09-21.

## Objections a reviewer is likely to raise

### 1. No field report shows this memory peak hurts anyone

A committer reworks heap_form_tuple(), fill_val() and heap_fill_tuple() into _attr variants for a narrow workload: REPLICA IDENTITY FULL tables with several large toasted columns. Users of that workload saw no failure before. Everyone else carries the reshaped core tuple-forming code and its new arguments in later patches and back-patches.

The only numbers come from one synthetic single-row test in message #2, and none come from a production system. Message #1 calls the 2x peak a strain but cites no out-of-memory report, -bugs thread or user complaint. The gain needs two or more large external columns: with one column the cut is 0%, and backend RSS drops 6 to 19% at a 96 MB payload. The cost lands in heaptuple.c, which every heap_form_tuple() caller uses, so non-users carry the maintenance.

**Question:** Which production systems using REPLICA IDENTITY FULL hit memory limits or OOM during toast_flatten_tuple(), and how large were their old tuples?

<sub>fixable, level 1b, demand; messages [#1](https://www.postgresql.org/message-id/CALT9ZEG8W0-xVCYQmxebYQ55fE-SJmKOTrBqgMbcVekhGNMaAw%40mail.gmail.com), [#2](https://www.postgresql.org/message-id/CAHYEV2tW%2BogNGXC7tnVL17Q6J26S-2R7%2BvBCkFEsd_VTwUiWiA%40mail.gmail.com); `demand-1`</sub>

### 2. Stub headers must track detoast output with no loud failure

A later patch adds a compression method or a new external varlena kind and changes what detoast_external_attr() returns. The stub logic in heaptoast.c is not updated, so the layout reserves a different size than the detoasted value needs. The write overruns the result tuple or leaves invalid data in it, and users find out from corrupted rows on a subscriber.

The design needs the stub header to predict the size and compression tag that detoast_external_attr() will return. That is a second copy of detoast format knowledge, kept in a different file. The commit message already treats INDIRECT as a case the stub cannot predict, which shows the prediction is fragile. The prose does not mention a runtime check that compares the detoasted length with the reserved space before the copy.

**Question:** Does v2 check, in production builds, that each detoasted value's length equals the reserved stub size and raise an error before it writes, not after?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#1](https://www.postgresql.org/message-id/CALT9ZEG8W0-xVCYQmxebYQ55fE-SJmKOTrBqgMbcVekhGNMaAw%40mail.gmail.com), [#3](https://www.postgresql.org/message-id/CALT9ZEEiDGAjG65B%2B15-RcHf1g3DN2uqf3K0SbTE5QgLstnwVw%40mail.gmail.com); `maintenance-fit-1`</sub>

### 3. Every heap_form_tuple() call now goes through the new _attr path

heap_form_tuple() and heap_fill_tuple() run for nearly every tuple that executor nodes, COPY and catalog code build. After v2, each call goes through the _attr variants, which take extra skip_copy and attr_data arguments. If those are not fully compiled away, a bulk INSERT ... SELECT or COPY into a wide table gets slower for users who never flatten a toasted tuple.

Measurements in message #2 cover only toast_flatten_tuple() memory, not the CPU cost of the shared fill path. The new arguments exist for one caller, but the cost of the shared path falls on all callers. A committer would likely ask for a micro-benchmark of heap_form_tuple() on many-column tuples before and after.

**Question:** What is the measured cost of heap_form_tuple() and heap_fill_tuple() on a wide, many-attribute tuple before and after v2, for example in a COPY benchmark?

<sub>fixable, level 2, performance on the hot path; messages [#2](https://www.postgresql.org/message-id/CAHYEV2tW%2BogNGXC7tnVL17Q6J26S-2R7%2BvBCkFEsd_VTwUiWiA%40mail.gmail.com), [#3](https://www.postgresql.org/message-id/CALT9ZEEiDGAjG65B%2B15-RcHf1g3DN2uqf3K0SbTE5QgLstnwVw%40mail.gmail.com); `maintenance-fit-2`</sub>

## Questions for the author

1. Does v2 check in production builds that each detoasted value's length equals the space reserved by its stub, and raise an error before it writes?
2. What does heap_form_tuple() cost on a wide, many-attribute tuple before and after v2, for example in a COPY benchmark?
3. Which production systems with REPLICA IDENTITY FULL hit memory limits during toast_flatten_tuple(), and how large were their old tuples?
4. What is the CPU cost of v2 when a tuple has many small external attributes, where the per-attribute stub work is not hidden by large toast fetches?
5. Is heap_form_tuple_attr() meant as an API for extensions, or as an internal helper for toast_flatten_tuple() only?

## Touch points

- **WAL / replication / standby:** The flattened old tuple for REPLICA IDENTITY FULL is written to WAL and decoded by logical replication.
- **ABI / extension API:** htup_details.h declares the new heap_form_tuple_attr() and the _attr variants of heap_fill_tuple() and fill_val().
- **backpatchability:** The patch reshapes fill_val() and heap_fill_tuple() in heaptuple.c, so later fixes there apply differently to older branches.
- **maintenance burden ("who maintains this in 10 years"):** The stub logic in heaptoast.c is a second copy of what detoast_external_attr() returns and must stay in step with it.
- **performance on the hot path:** heap_form_tuple() and heap_fill_tuple() become wrappers around the _attr variants, so every tuple-forming caller takes the new path.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** see objection demand-1: No field report shows this memory peak hurts anyone

## Who would commit it, and when

- Owner: silent. It needs a committer who knows the heap tuple forming code in heaptuple.c and the TOAST code. The visible thread has the author and one reviewer, and no such committer has spoken.
- Priority: proceed; audience niche; review cost medium. The gain needs two or more large external columns and mainly helps REPLICA IDENTITY FULL tables, but message #2 measured up to 28% lower backend peak RSS. The patch is about 19 KiB, adds no user-visible surface, and changes shared code in heaptuple.c, so it needs a rebase and the two design answers.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

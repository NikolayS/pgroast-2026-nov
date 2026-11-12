# Design pre-review: Fix double-counting of pending relation stats by VACUUM and ANALYZE

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7460](https://commitfest.postgresql.org/patch/7460/); [thread](https://www.postgresql.org/message-id/flat/1DC9D815-8754-4A76-B2B1-99C1A614A55A%40localhost.localdomain), 12 of 12 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). No surviving objection is fatal, and no level fails. The thread shows a deterministic single-session reproduction on 14 through master and measured extra autovacuum runs. Two fixable points remain. prior-attempts-1 asks why an earlier thread judged a similar case not a bug. standards-compat-1 asks whether 0002 may change a pgstat.h struct in stable branches. A committer who knows the cumulative stats code has to settle both.

Not checked: Without the code, nobody can check that dropping pending deltas is correct with subtransactions, 2PC and concurrent sessions. The same holds for whether the 0002 struct change is ABI-safe on 14 through 18.

## Prior attempts

None found before 2026-10-14.

## Objections a reviewer is likely to raise

### 1. Same cause as an earlier case judged not a bug

A committer who took part in the earlier thread sees the same root cause raised again and declines 0001 for the same reason. Only the TRUNCATE fix goes in. The VACUUM and ANALYZE double count stays, and users running ETL scripts keep seeing extra autovacuum and autoanalyze runs.

Message #10 points to an earlier thread where a similar case was not treated as a bug. Message #11 agrees the VACUUM example there has the same cause. The thread does not report why that thread decided as it did. Message #11 reads the earlier reason as a wish to keep stats batched and says the patch keeps batching. If the earlier reason was that these counters are approximate by design, the fix is still partial: other backends' pending deltas, including those seen by autovacuum's own VACUUM, still inflate the counters, as message #11 concedes.

**Question:** What reason did the earlier thread give for calling this not a bug, and why does that reason not cover the same-backend case fixed here?

<sub>fixable, level 1b, prior rejection; messages [#10](https://www.postgresql.org/message-id/auwYtm6qM3fPltNF%40bdtpg), [#11](https://www.postgresql.org/message-id/2C6AA9AE-4348-4952-B971-E0D63B2B1A53%40localhost.localdomain); `prior-attempts-1`</sub>

### 2. Back-patching 0002 changes a public pgstat.h struct in stable branches

A packager ships a 14-18 minor release with 0002, and an extension built against the old pgstat.h keeps running unrebuilt. If that extension embeds, copies or sizes the pending table stats struct, it now reads or allocates the wrong layout. The breakage would show up as wrong counters or memory errors after the minor update.

0002 adds ins_before_vacuum to a struct in pgstat.h that C extensions can see. Message #2 and message #5 argue that appending the field keeps the inline pgstat_count_* offsets stable, so a back-patch stays possible. That covers offsets inside the struct. It does not cover the struct's size, or any containing struct that holds it ahead of other members, and nobody in the thread ran an ABI check on 14-18. 0001 touches only pgstat_relation.c, so it can be back-patched alone, with 0002 kept for master only.

**Question:** Which stable branches would receive 0002, and does an ABI comparison of 14-18 show no size or layout change in any struct visible to extensions?

<sub>fixable, level 2, backpatchability; messages [#2](https://www.postgresql.org/message-id/179200362437.252424.7000490135467740876%40gmail.com), [#5](https://www.postgresql.org/message-id/1CBF745F-CB0D-4D35-9E25-D6B5208335E4%40localhost.localdomain), [#12](https://www.postgresql.org/message-id/CADMs5ONUqqmPcNWnqobGy8wWujFbYHusZjwdJsehShGH3SsS1Q%40mail.gmail.com); `standards-compat-1`</sub>

## Questions for the author

1. What reason did the earlier thread cited in message #10 give for calling the similar case not a bug, and why does it not cover the same-backend case?
2. Which stable branches should receive 0002, and does an ABI comparison on 14 through 18 show no size or layout change visible to extensions?
3. Should this patch or the in-transaction stats flush patch (v17) go first, and how does the reset look with flushed_xact baselines?
4. Do 0001 alone and 0001 plus 0002 each pass check-world, and what do the counters show when another session modifies the table concurrently?

## Touch points

- **ABI / extension API:** 0002 appends ins_before_vacuum to the pending table stats struct in pgstat.h, which C extensions can see.
- **backpatchability:** The bug reproduces on 14 through 18, and the thread offers back-branch versions, including the struct change in 0002.
- **maintenance burden ("who maintains this in 10 years"):** The reset of pending deltas conflicts with the in-transaction stats flush patch (v17), and the VACUUM test is best-effort.

## Who would commit it, and when

- Owner: silent. Someone who knows the cumulative stats code in pgstat_relation.c and how autovacuum reads these counters; no committer has replied in the thread.
- Priority: proceed; audience a sizeable minority; review cost low. The patches are small and touch one source file plus one header field. They fix a deterministic miscount that causes extra autovacuum runs after a bulk load or delete followed by ANALYZE or VACUUM in the same session. A committer must answer the "not a bug" question and the back-patch question.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 2 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

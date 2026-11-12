# Design pre-review: Fall back to serializing when parallel apply runs out of DSM segments

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7548](https://commitfest.postgresql.org/patch/7548/); [thread](https://www.postgresql.org/message-id/flat/CA%2Bfm-RMyb6EoMuk%2Ba4_hZou9cdTHfB6mj1UV-7yb_YuTWLug1g%40mail.gmail.com), 5 of 5 messages read; no code read.

## Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium). No objection survived at levels 1a, 1b or 2. Both patches pass a flag that InitializeParallelDSM() already passes, so existing NULL checks and the documented streaming = parallel fallback become reachable. Message #1 gives a reproduction, which P4 and P5 lacked. Confidence is medium because the code was not read and the callers of make_new_segment() were not checked.

Not checked: The code was not read. Whether the leader's fallback and every caller of make_new_segment() behave correctly on the newly reachable NULL path could not be judged.

## Prior attempts

- [Improve logicalrep_worker_launch() logic](https://www.postgresql.org/message-id/flat/d9e208a0-1bdf-42e3-b6c6-49bca1904060@oss.nttdata.com) (related; Withdrawn, 2026-01-08). "No, the idea was that if we are changing the code, it is better to test it via a testcase instead of changing based on assumptions." ([message](https://www.postgresql.org/message-id/CAA4eK1J17Le_Xx4BmGXa-KubmuBaw7xRhvok7WqzsY_5heyK%2BA%40mail.gmail.com)) Different now: P4 changed worker-slot cleanup based on reading the code. This proposal makes an existing fallback reachable and reports a reproduction in message #1.
- [logicalrep_worker_launch - counting/checking the type limits](https://www.postgresql.org/message-id/flat/CAHut+Pu-7DiCwMdbBf5vzUL1t8H=_NpyFR4RvpZbpMGXV=+QLQ@mail.gmail.com) (related; Withdrawn, 2024-04-23). "Did you try to reproduce this condition, if not, can you please try it once? I wonder if the leader worker crashed, won't it lead to a restart of the server?" ([message](https://www.postgresql.org/message-id/CAA4eK1LtFyiMV6e9%2BRr66pe5e-MX7Pk6N3iHd4JgcBW1X4kS6Q%40mail.gmail.com)) Different now: P5 changed per-type worker limit checks without a reproduction. This proposal reproduced the failure, which is a leader apply worker restart loop.

## Objections a reviewer is likely to raise

No objection held up in the counter-check.

## Questions for the author

1. Does every path that reaches make_new_segment() handle a NULL segment correctly now that 0002 makes that return reachable?
2. Can the out-of-segments path get an in-tree regression test, since the reproduction in message #1 used an external test extension?
3. Do other dsm_create() callers have the same unreachable NULL check, and should this series fix them too?
4. Will the commit messages drop the Backpatch-through lines if the fix goes to HEAD only?
5. What is pg_stash_advice, and where does the code that 0002's commit message names live?

## Touch points

- **WAL / replication / standby:** 0001 changes what the leader apply worker does at STREAM START when no DSM segment is free.
- **ABI / extension API:** 0002 makes dsa_allocate_extended() with DSA_ALLOC_NO_OOM return InvalidDsaPointer where it raised an ERROR, which extension callers see.
- **backpatchability:** The commit messages say backpatch to 16 and 14, while the thread now leans to HEAD only and leaves the choice to a committer.

## Who would commit it, and when

- Owner: silent. 0001 needs a committer who knows the logical replication apply code; 0002 needs one who knows DSA and DSM. The brief names two reviewers who approve but does not show a committer in the thread.
- Priority: proceed; audience niche; review cost low. Each patch is under 2 KiB, touches one file and adds no new user-visible surface. Only subscribers that run out of DSM segments with streaming = parallel benefit, and two reviewers already approve.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

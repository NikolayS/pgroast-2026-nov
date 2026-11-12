# Design pre-review: Fix use-after-free after failed pg_checksum_init

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7257](https://commitfest.postgresql.org/patch/7257/); [thread](https://www.postgresql.org/message-id/flat/9d69505e8d4948048e8401165ca4c3b8%40localhost.localdomain), 6 of 6 messages read; no code read.

## Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium). No objection survived the critic. The only one raised, maintenance-fit-1 on the linker-substituted test mock, was refuted because both failure modes are loud and the point is test-level. The fix clears a dangling pointer, frees a leaked context, and makes pg_combinebackup exit on checksum errors as basebackup and pg_verifybackup do. A committer agreed with the goal in message #2 and the brief reports v3 as mostly ready.

Not checked: The code was not read, so this review could not check that every fallible pg_checksum_init() and pg_checksum_final() call in pg_combinebackup is covered, or that no caller frees the context a second time. Only three of six messages were visible.

## Prior attempts

None found before 2026-08-17.

## Objections a reviewer is likely to raise

No objection held up in the counter-check.

## Questions for the author

1. Is there a v4 that answers the review comments of 2026-11-06 on the if/else with an Assert, the Makefile rule name, and one helper call per test case?
2. Should the fix go to 17 and later, and if so, does the back-branch version include the new test program or only the code change?
3. Does test_checksum_helper link and run with the mock cryptohash functions on Windows and macOS, under both meson and autoconf?
4. Does any in-tree or known out-of-tree caller of pg_checksum_final() use the SHA context after the call and so depend on the old cleanup behaviour?

## Touch points

- **ABI / extension API:** pg_checksum_final() in src/common now always frees and clears the SHA context, which changes the cleanup contract for callers.
- **backpatchability:** Message #1 asks for a back-patch to 17; message #2 calls the risk/reward unfavorable, and the question is still open.
- **maintenance burden ("who maintains this in 10 years"):** The patch adds test_checksum_helper, a test program that links checksum_helper.c against mock pg_cryptohash_* functions.

## Who would commit it, and when

- Owner: qualified committer engaged. Daniel Gustafsson reviewed the patch in message #2 and, per the brief, calls v3 mostly ready. The owner needs to know the frontend cryptohash and OpenSSL code and the backup tools.
- Priority: proceed; audience niche; review cost low. The error path needs an OpenSSL init or final failure, which message #2 calls unlikely, so few users will ever hit it. The v3 patch is about 16 KiB over nine files in pg_combinebackup and one src/common file, and a committer is already reviewing it.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 1 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

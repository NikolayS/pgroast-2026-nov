# Design pre-review: Refactor pgbench - pure mechanical modularization into 10 module pairs (stats, variable, poller, random, expr, script, commands, init, engine, report)

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7387](https://commitfest.postgresql.org/patch/7387/); [thread](https://www.postgresql.org/message-id/flat/CAMT0RQT1BMSKqgO31QJw4razSDruW_HX6G-R_Xr7yXsfMM_H2g%40mail.gmail.com), 14 of 14 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The surviving objections demand-1 and maintenance-fit-1 both fail level 1b. The split has no user-visible effect, and the only named consumer is the author's own later work, which message #12 still lists as a follow-up. Messages #13 and #14 ask to start from one feature with only the refactoring it needs, and the thread has no answer to that yet. The goal may be fine, but the list has to agree on the approach before code review makes sense.

Not checked: No one has confirmed that v7 is move-only; that claim and the amount of coupling through exported globals need the code. The commit messages for 0009 and 0010 were not in the material.

## Prior attempts

- [split tablecmds.c](https://www.postgresql.org/message-id/flat/aS3PeXHLe7ir_gPc@nathan) (similar idea; Withdrawn, 2026-08-24). reason not recorded Different now: P14 split a backend file, tablecmds.c; this splits a client program, pgbench.c. The reason for P14's withdrawal is not recorded, so it gives no lesson here.
- [Split xlog.c](https://www.postgresql.org/message-id/flat/b3b71061-4919-e882-4857-27e370ab134a@iki.fi) (similar idea; Returned with feedback, 2021-11-15). reason not recorded Different now: P15 split xlog.c in the backend; this splits pgbench.c into ten modules. The reason P15 was returned is not recorded.

## Objections a reviewer is likely to raise

### 1. No concrete feature or user shown to need this module layout

After the split is committed, every open pgbench patch must be rebased onto ten new files, and each back-patched pgbench fix must be adapted by hand. If the feature that motivated the split never lands, or needs different boundaries, the project pays that cost and may move the code a second time.

The only stated beneficiary is the author's own planned work: contrib/pgbench and TPC-like features (messages #3, #12). No other user or developer asks for the split. Dolgov calls the motivation weak and asks for one feature with only the refactoring it needs (message #13); Shibagaki withdraws support for split-only review for the same reason (message #14). Message #12 says the split helps "anyone wanting to improve pgbench" but names no such person or patch, and that argument would fit any large source file.

Precedent: [Docs: make source file path references consistent](https://www.postgresql.org/message-id/flat/CAEoWx2n=X2vvkJqsNDENgqkyw_adG=7wSHG-og-dLbyM4G_7SA@mail.gmail.com) (Withdrawn, 2026-01-08)

**Question:** Which concrete pgbench feature, wanted by someone besides the author, cannot reasonably be built on today's pgbench.c, and which of the ten modules does it need?

<sub>fixable, level 1b, demand; messages [#3](https://www.postgresql.org/message-id/CAMT0RQSQiLuOWw6R750Dn3hPC3qy_mJrBEst-Xrw68tqSkYWHQ%40mail.gmail.com), [#12](https://www.postgresql.org/message-id/CAMT0RQTKqefLQJropNPKsSo1i%3DahxjbGRFHJUT6A%2BUn37q4wwQ%40mail.gmail.com), [#13](https://www.postgresql.org/message-id/autS5tAQoZWpZQCL%40ddolgov-thinkpadt14sgen1.rmtde.csb), [#14](https://www.postgresql.org/message-id/TY5PR01MB1844582E13D20621BE09CA1DCFA772%40TY5PR01MB18445.jpnprd01.prod.outlook.com); `demand-1`</sub>

### 2. Module boundaries chosen before any feature that would use them

The split goes in, then the planned distribution or contrib/pgbench work needs random.c in src/common and a different expr/commands interface. A second large reshuffle follows. Back-branch fixes and git blame in pgbench pay for both rounds.

v7 makes about 66 functions and many globals external, including throttle_delay, timer_exceeded, use_quiet and scale, without any encapsulation. The new files are therefore coupled through shared globals, so the split adds files but not real interfaces. Messages #13 and #14 ask for one feature first, so that the boundaries are judged against a real consumer. Message #12 still names that feature only as a later follow-up.

**Question:** Which part of the distribution-generator or contrib/pgbench work needs exactly this ten-module layout, and would random.c move again for it?

<sub>fixable, level 1b, design; messages [#12](https://www.postgresql.org/message-id/CAMT0RQTKqefLQJropNPKsSo1i%3DahxjbGRFHJUT6A%2BUn37q4wwQ%40mail.gmail.com), [#13](https://www.postgresql.org/message-id/autS5tAQoZWpZQCL%40ddolgov-thinkpadt14sgen1.rmtde.csb), [#14](https://www.postgresql.org/message-id/TY5PR01MB1844582E13D20621BE09CA1DCFA772%40TY5PR01MB18445.jpnprd01.prod.outlook.com); `maintenance-fit-1`</sub>

## Questions for the author

1. Which concrete pgbench feature needs these module boundaries, and which of the ten modules does that feature use?
2. Does anyone besides the author have a pgbench patch or plan that today's single pgbench.c blocks or makes harder?
3. Will random.c move again to src/common for the contrib/pgbench work, and if so, why not move it there in one step?
4. Can the series be reduced to the distribution-generator feature plus only the code moves it needs, as messages #13 and #14 ask?
5. Where is the promised table of which function goes to which module, and what do patches 0009 and 0010 move?

## Touch points

- **backpatchability:** Fixes to code moved out of pgbench.c need manual ports to older branches; message #2 counts about four such commits per year.
- **maintenance burden ("who maintains this in 10 years"):** The series adds ten module pairs and makes about 66 functions and many globals external inside src/bin/pgbench.

## Who would commit it, and when

- Owner: silent. A committer who knows pgbench and its back-branch history would have to own it. The material shows reviewers in the thread but does not identify any of them as a committer.
- Not now, show demand first: A posted feature patch, such as SQL-callable distribution generators, that uses specific module boundaries from this series, or pgbench patches from other developers that the single file demonstrably hinders.
- Priority: needs design rethink; audience unclear; review cost high. The split changes nothing for users and its only named consumer is unposted follow-up work, while verifying about 7,000 moved lines as move-only is costly. Reviewers in messages #13 and #14 ask for a feature-first series with only the refactoring that feature needs.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 2 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 9.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

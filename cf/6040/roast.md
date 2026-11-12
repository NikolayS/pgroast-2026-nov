# Design pre-review: Fix read/write races with pg_control file in case when postgres was compiled with DEXEC_BACKEND option

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6040](https://commitfest.postgresql.org/patch/6040/); [thread](https://www.postgresql.org/message-id/flat/f59335a4-83ff-438a-a30e-7cf2200276b6%40postgrespro.ru), 6 of 6 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). No surviving objection questions the bug or either fix. The race is reproduced in message #3, and both designs reuse existing mechanisms. The one open point is prior-attempts-1: message #5 names another thread with its own fix and CommitFest entry, and nobody has said which patch is the candidate. That is a coordination question to settle before commit, not a flaw in the idea.

Not checked: The BackendParameters patch from message #4 and messages #4 to #6 were not shown, so its handling of a stale postmaster copy could not be checked. The fix in the thread linked in message #5 was not seen either.

## Prior attempts

- [Fixes for non-atomic read of read of control file on ext4 + ntfs](https://www.postgresql.org/message-id/flat/20221123014224.xisi44byq3cf5psi@awork3.anarazel.de) (similar idea; Returned with feedback, 2024-02-01). "I have changed the status of this to "Waiting on Author" as Robert's comments have not yet been handled." ([message](https://www.postgresql.org/message-id/CALDaNm01LGGbQ0DJATqwtmg4xx62dQTbrRjg%3DfFhObGCJ2EzqA%40mail.gmail.com)) Different now: P1 fixed torn pg_control reads in frontend programs with a retry loop, and that part was committed. This proposal targets EXEC_BACKEND child processes, and the thread now prefers passing the postmaster's copy through BackendParameters. P1 was returned only for unhandled review comments on its backup patches, which does not apply here.

## Objections a reviewer is likely to raise

### 1. Overlap with the fix in the thread linked in message #5 is unresolved

Two CommitFest entries carry different fixes for the same EXEC_BACKEND torn read. A committer reviews this entry while the other fix is being reviewed or committed elsewhere. Reviewer time is split, and the second fix to land either conflicts with the first or duplicates its work.

Message #5 reports that the same issue was discussed in another thread, which has its own fix and its own CommitFest entry. No later message says how the two fixes compare or which one should go forward. Message #6 approves Korotkov's BackendParameters patch, but the entry still names only the v1 author and needs a rebase. A committer would likely ask which patch, in which entry, is the candidate.

**Question:** Which of the two fixes, this entry's BackendParameters patch or the one in the thread from message #5, should be committed, and should the other entry be closed?

<sub>fixable, level 1a, prior rejection; messages [#4](https://www.postgresql.org/message-id/CAPpHfdsXkEWUeLUG4zh9q%3DhjpsOCMgsbN_XZh-6JL0z1NaNMqQ%40mail.gmail.com), [#5](https://www.postgresql.org/message-id/033028ce-8ef7-421e-92dc-bcd8a421c4c3%40gmail.com), [#6](https://www.postgresql.org/message-id/4ff7bf45-3a95-471a-8dc2-84a8a3b3924a%40postgrespro.ru); `prior-attempts-1`</sub>

## Questions for the author

1. Which fix should be committed, the BackendParameters patch here or the one in the thread linked in message #5, and should the other CommitFest entry be closed?
2. Does CommitFest entry 6040 now track v1 or the BackendParameters patch, and can a rebased version be posted?
3. Can a child process need a newer pg_control than the postmaster's copy at the time it starts?
4. Should the fix be back-patched, and to which branches?
5. Do Windows production builds hit this race, or only test builds compiled with -DEXEC_BACKEND?

## Touch points

- **backpatchability:** The entry is tagged Bugfix, and the thread does not say which branches should get the fix.
- **maintenance burden ("who maintains this in 10 years"):** Two threads carry different fixes for the same race, and the change touches control file handling in xlog.c.

## Who would commit it, and when

- Owner: silent. Someone who knows postmaster child startup under EXEC_BACKEND and the control file code in xlog.c. Korotkov posted the alternative patch, but the material does not show that he intends to commit it.
- Priority: proceed; audience a sizeable minority; review cost low. The bug is reproduced and affects builds that start children with exec, which the demand notes say includes Windows. v1 is 2.3 KB in one file. First decide which of the two entries goes forward, then post a rebased patch.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 1 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

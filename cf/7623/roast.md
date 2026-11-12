# Design pre-review: Injection point: fix race condition between attach and cleanup

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7623](https://commitfest.postgresql.org/patch/7623/); [thread](https://www.postgresql.org/message-id/flat/OS7PR01MB183175FA935E5DC8734FF2CD6F5712%40OS7PR01MB18317.jpnprd01.prod.outlook.com), 3 of 3 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). No surviving objection stops the proposal at levels 1 or 2. The race is real and the 0001 spec reproduces it; the test-only fix leaves it in the module. demand-1 and standards-compat-1 are both fixable: the thread names one failing test as the consumer of a new core API, and names no branch for either fix. A committer who knows injection points must choose between the two fixes, which the author asks for in message #1.

Not checked: Without the code, this review cannot check whether the shared-memory slot layout changes, how the generation counter handles slot reuse, or whether the handle check and the detach happen under one lock.

## Prior attempts

- [Disable installcheck for tap and regress tests for injection_points module](https://www.postgresql.org/message-id/flat/CANtu0oiTgFW47QgpTwrMOVm3Bq4N0Y5bjvTy5sP0gYWLQuVgjw@mail.gmail.com) (related; Withdrawn, 2025-02-21). "Disabling the test hides the problem, it does not fix it." ([message](https://www.postgresql.org/message-id/Z0lL3GKCyijL8HcI%40paquier.xyz)) Different now: P4 turned off installcheck for the injection_points tests to avoid a crash in shared memory setup during abort. This proposal fixes a different race, in the cleanup of local points, and addresses its cause in the module.

## Objections a reviewer is likely to raise

### 1. New core API has one consumer while a test-only fix exists

The project adds InjectionPointAttachWithHandle() and InjectionPointDetachByHandle() to core injection_point.c, and only the injection_points test module ever calls them. Later work on injection_point.c must keep the slot and generation rules behind the handle intact, though only one cleanup path depends on them.

The material shows one concrete failure: 010_index_concurrently_upsert fails at times (message #1). Message #1 also offers a test-only fix that waits for cleanup, and the author asks reviewers to choose between the two. No other test, extension or user is named as needing to detach by handle, and the docs present it as a general API. A committer would likely ask whether more than one test is exposed before taking the larger fix.

**Question:** Which tests or extensions besides 010_index_concurrently_upsert use injection_points_set_local() in a way that can hit this race, and how would the test-only fix leave them exposed?

<sub>fixable, level 1b, demand; messages [#1](https://www.postgresql.org/message-id/OS7PR01MB183175FA935E5DC8734FF2CD6F5712%40OS7PR01MB18317.jpnprd01.prod.outlook.com); `demand-1`</sub>

### 2. Core API addition does not say which branches get the fix

010_index_concurrently_upsert keeps failing at times on the buildfarm in a stable branch. A committer must then either back-patch a new exported type and two functions in injection_point.h, or apply the test-only fix there. Master and the back branches then clean up local points in different ways.

The fix adds a new public C type and two functions to backend code to fix a race in a test module. The thread does not say which branches carry the failing test or the local-point cleanup, so the cost of the core change in stable branches is unknown. The author offers a test-only alternative but does not say whether it is meant for the back branches, which would mean two fixes for one race. A committer would likely ask for this plan before taking on new API in core.

**Question:** Which branches have this race, and which fix, the handle API or the nocfbot test change, goes into each of them?

<sub>fixable, level 2, backpatchability; messages [#1](https://www.postgresql.org/message-id/OS7PR01MB183175FA935E5DC8734FF2CD6F5712%40OS7PR01MB18317.jpnprd01.prod.outlook.com), [#2](https://www.postgresql.org/message-id/OS7PR01MB18317388DDFE410B1B93562F3F5712%40OS7PR01MB18317.jpnprd01.prod.outlook.com); `standards-compat-1`</sub>

## Questions for the author

1. Which tests or extensions besides 010_index_concurrently_upsert use injection_points_set_local() in a way that can hit this race?
2. Which branches have this race, and which fix, the handle API or the nocfbot test change, goes into each of them?
3. Is the handle API meant for extensions outside the injection_points test module, or only for that module?
4. How does the generation counter behave when a slot is reused many times, including on wraparound?
5. Does the same race exist for points that are not local, when two backends detach and attach the same name?

## Touch points

- **ABI / extension API:** The patch adds InjectionPointHandle and two exported functions to injection_point.h and documents them in xfunc.sgml.
- **backpatchability:** The thread does not say which branches have the race, and a back-patch would add a new type and two functions.
- **maintenance burden ("who maintains this in 10 years"):** Later changes to injection_point.c must keep the slot and generation rules that the handle depends on.

## Who would commit it, and when

- Owner: silent. A committer who knows the injection point code in injection_point.c and the injection_points test module. The author copied two people from the original thread; neither has replied.
- Priority: proceed; audience niche; review cost low. The patch fixes a reproducible race that makes a test fail at times, and it benefits developers who write tests with local injection points. It is about 13 KiB, touches one core file, and keeps the name-based interfaces unchanged.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

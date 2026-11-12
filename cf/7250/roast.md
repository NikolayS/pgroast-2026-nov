# Design pre-review: glist: _Generic wrapper for selective dlist/dclist usage

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7250](https://commitfest.postgresql.org/patch/7250/); [thread](https://www.postgresql.org/message-id/flat/CAEze2WjUs6P1QCUHij9WOnkS7CX%2BtvVqRiFiGdXoyevh417h4A%40mail.gmail.com), 8 of 8 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The idea is clear, and one reviewer found the macros correct, but the proposal fails at desirability as a standalone patch. It touches only ilist.h and adds no caller, so no build compiles the _Generic dispatch (maintenance-fit-1). The thread names one prospective user, in an uncommitted series, and no second one (demand-1). The proposal gives no number for the memory cost of using dclist in all builds, so the existing type may be enough (already-exists-1).

Not checked: Without the code, the set of wrapped functions and the behaviour of the _Generic dispatch on all supported compilers and in C++ header checks could not be judged. One reviewer found the macros correct in message #4.

## Prior attempts

None found before 2026-09-02.

## Objections a reviewer is likely to raise

### 1. New header API with no caller cannot be checked or maintained

A later patch changes a dclist_* signature or adds a dlist_* variant and updates every caller. The glist_* macros in ilist.h expand only where they are used, so with no user the build stays green. The first developer who tries glist finds it broken or out of step, possibly years later.

The patch touches only src/include/lib/ilist.h and adds no user. Japin Li could not name an existing place to use it (message #8), and David Rowley asked for a caller in the same series (message #5). The author answers that a separate thread gathers feedback on _Generic as an API style (message #7). That is a reason to discuss the idea, but a committer would likely not take ownership of macros that nothing compiles.

**Question:** Will the Proxy memory context series carry this patch as its first step, with a caller that compiles the glist_* macros in normal and MEMORY_CONTEXT_CHECKING builds?

<sub>fixable, level 1b, maintenance burden ("who maintains this in 10 years"); messages [#5](https://www.postgresql.org/message-id/CAApHDvo437%2BLn-3C7%2BEDNW5k3dMpSKb-z6PCr_0quPpf%3DYqboA%40mail.gmail.com), [#7](https://www.postgresql.org/message-id/CAEze2WhB_cofsSuT113CFqwJBp0gpDQckCdRzbFcsuiqX138Rg%40mail.gmail.com), [#8](https://www.postgresql.org/message-id/SY7PR01MB10921344A12E4017BB4D76B28B6B62%40SY7PR01MB10921.ausprd01.prod.outlook.com); `maintenance-fit-1`</sub>

### 2. New API with no caller and one prospective user

A committer pushes glist_* into ilist.h as a standalone patch. The Proxy memory context series then changes direction, or drops its debug-only count after review. ilist.h is left with _Generic macros that nothing calls, and every later change to dlist or dclist must keep them in sync, with no caller to show breakage.

The only use named is the author's own Proxy memory context series (message #1). Message #3 says glist is not meant for code that is always dlist or always dclist, and Japin Li could name no existing place where it fits (message #8). David Rowley questions whether that one use is worth having (message #5); message #7 answers him, but nobody else has agreed yet. The author wants a separate thread to collect feedback on _Generic in headers, and so far that has produced no second user.

**Question:** Which callers besides the Proxy memory context series need a list that is dlist in one build and dclist in another, and could glist ship as a patch inside that series?

<sub>fixable, level 1b, demand; messages [#1](https://www.postgresql.org/message-id/CAEze2WjUs6P1QCUHij9WOnkS7CX%2BtvVqRiFiGdXoyevh417h4A%40mail.gmail.com), [#3](https://www.postgresql.org/message-id/CAEze2Wj7GJjfPm8HLtTnaS2EWFCj8fOcRXv8uWebDrC9ZTc5YQ%40mail.gmail.com), [#5](https://www.postgresql.org/message-id/CAApHDvo437%2BLn-3C7%2BEDNW5k3dMpSKb-z6PCr_0quPpf%3DYqboA%40mail.gmail.com), [#7](https://www.postgresql.org/message-id/CAEze2WhB_cofsSuT113CFqwJBp0gpDQckCdRzbFcsuiqX138Rg%40mail.gmail.com), [#8](https://www.postgresql.org/message-id/SY7PR01MB10921344A12E4017BB4D76B28B6B62%40SY7PR01MB10921.ausprd01.prod.outlook.com); `demand-1`</sub>

### 3. Plain dclist or a local wrapper already covers the one known user

The Proxy memory context code can declare its list as a dclist in all builds, or keep a small #ifdef MEMORY_CONTEXT_CHECKING wrapper next to its only use. If glist goes into ilist.h, every later change to the dlist or dclist API also needs a matching glist macro, for a single caller in a different patch series.

dclist already keeps the element count, and using it in every build costs one count field per list head plus an increment per insert or delete. The author calls this a waste of memory (message #1) but does not say how many such list heads a proxy context has. The point that hand-written wrappers would be duplicated for each user (message #1) does not hold yet, because the thread names no second user and Japin Li found none (message #8).

**Question:** How much memory does an unconditional dclist cost the Proxy memory contexts in production builds, and which second caller would need the same build-dependent list type?

<sub>fixable, level 1b, doesn't fit what Postgres is about (extension / external tool / already solved); messages [#1](https://www.postgresql.org/message-id/CAEze2WjUs6P1QCUHij9WOnkS7CX%2BtvVqRiFiGdXoyevh417h4A%40mail.gmail.com), [#3](https://www.postgresql.org/message-id/CAEze2Wj7GJjfPm8HLtTnaS2EWFCj8fOcRXv8uWebDrC9ZTc5YQ%40mail.gmail.com), [#5](https://www.postgresql.org/message-id/CAApHDvo437%2BLn-3C7%2BEDNW5k3dMpSKb-z6PCr_0quPpf%3DYqboA%40mail.gmail.com), [#8](https://www.postgresql.org/message-id/SY7PR01MB10921344A12E4017BB4D76B28B6B62%40SY7PR01MB10921.ausprd01.prod.outlook.com); `already-exists-1`</sub>

## Questions for the author

1. Will the Proxy memory context series carry this patch as its first step, with a caller that compiles the glist_* macros in normal and MEMORY_CONTEXT_CHECKING builds?
2. How much memory does a dclist in all builds cost the Proxy memory contexts in production, counted per list head and per context?
3. Which caller besides the Proxy memory context series needs a field that is a dlist in one build and a dclist in another?
4. What does the count kept only in checking builds verify, given David Rowley's point that it checks only its own tracking?
5. Which dlist_* and dclist_* functions get glist_* wrappers, and how do the macros behave with compilers or build modes that lack _Generic?

## Touch points

- **ABI / extension API:** The patch adds glist_* macros to src/include/lib/ilist.h, a header that backend and extension code includes.
- **maintenance burden ("who maintains this in 10 years"):** The glist_* macros mirror dlist_* and dclist_* functions and have no caller that would show breakage.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** A dclist in all builds, or a local #ifdef wrapper, may already cover the one named user.

## Who would commit it, and when

- Owner: silent. The material does not identify any thread participant as a committer. The owner would be someone who knows ilist.h and the memory context code, and who accepts _Generic as an API style in headers.
- Not now, show demand first: A patch in the same series that calls the glist_* macros in normal and MEMORY_CONTEXT_CHECKING builds, plus either a number for the memory that a dclist in all builds would cost or a second caller with the same need.
- Priority: bundle; audience niche; review cost low. The patch is about 5 KiB in one header and serves one prospective caller, the Proxy memory context series. Two reviewers asked for it to be part of that series, where a caller would compile and justify it.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 2 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

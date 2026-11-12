# Design pre-review: Fix PGTYPESdate_fmt_asc overflow when a year does not fit "yyyy"

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7408](https://commitfest.postgresql.org/patch/7408/); [thread](https://www.postgresql.org/message-id/flat/CAB8bMiuv0T175FkbcFi_jGN1kp_0S_cYLyHXFiL7RvyuqqQrjg%40mail.gmail.com), 7 of 7 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No objection survived at levels 1a or 1b: the overflow is reproducible with ASan and the goal is a bug fix. The one surviving objection, standards-compat-1, is a fixable level 2 point: the patch targets stable branches but does not state the return value or the contents of outbuf on the new failure. A committer who knows ECPG must decide between an error and wider output, and how far back the fix goes.

Not checked: Without the code, the return value and the state of outbuf on failure could not be checked. Whether the length check covers every token, including negative years printed with "%04u", is also unverified.

## Prior attempts

None found before 2026-08-11.

## Objections a reviewer is likely to raise

### 1. Back-branch change to the PGTYPESdate_fmt_asc() failure contract is not defined

An ECPG application on a minor release formats far-future dates with a trailing "yyyy" into a large buffer, then prints outbuf without checking the return value. After the upgrade the call fails, and outbuf holds whatever the patch leaves there. The user finds out from wrong report output, not from an error.

The patch targets stable branches and turns a case that had no error into a new error, with errno set to PGTYPES_DATE_BAD_DATE. The commit message does not say what the function returns or what outbuf holds on this failure, and no documentation hunk lists the case. For a library contract changed in a minor release, a committer would likely ask for both to be stated. A committer would also likely ask whether a caller with a large enough buffer ever got a correct five-digit year before the patch.

**Question:** What do the return value and outbuf look like on the new failure, and did any caller with a large buffer get correct output before the patch?

<sub>fixable, level 2, backpatchability; messages [#1](https://www.postgresql.org/message-id/CAB8bMiuv0T175FkbcFi_jGN1kp_0S_cYLyHXFiL7RvyuqqQrjg%40mail.gmail.com), [#4](https://www.postgresql.org/message-id/CAB8bMiuOxjEJMVg%3DrWDE92AFDUsOjHCiWjRgg0JvFnEhcPZn5Q%40mail.gmail.com); `standards-compat-1`</sub>

## Questions for the author

1. What does PGTYPESdate_fmt_asc() return, and what does outbuf hold, when the year does not fit "yyyy"?
2. Did a caller with a buffer larger than strlen(fmtstring)+1 get a correct five-digit year before the patch, and does that caller now get an error?
3. Can any other token, such as a month or day name, produce a replacement longer than the token, and does the length check cover it?
4. Which stable branches should receive the fix?
5. Does the ECPG documentation for PGTYPESdate_fmt_asc() need to list the new failure case?

## Touch points

- **ABI / extension API:** PGTYPESdate_fmt_asc() in libpgtypes gains a new failure case for callers; the signature does not change.
- **security / privileges:** The patch closes a client-side heap buffer overflow in libpgtypes.
- **backpatchability:** The CommitFest entry targets stable branches, and the fix turns a case with no error into an error.

## Who would commit it, and when

- Owner: silent. Someone who knows ECPG and pgtypeslib, and who can judge a behavior change in a client library on stable branches. No such committer has reviewed the design in the visible thread.
- Priority: needs champion; audience niche; review cost low. The patch is about 6 KiB, touches one pgtypeslib file plus dt_test, and fixes an out-of-bounds write that only ECPG programs with years of 10000 or more reach. It waits on a committer who knows ECPG to settle the failure contract and the back-patch range.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

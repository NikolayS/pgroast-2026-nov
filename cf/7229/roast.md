# Design pre-review: pageinspect: validate line pointers before using them

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7229](https://commitfest.postgresql.org/patch/7229/); [thread](https://www.postgresql.org/message-id/flat/CAGRkXqRWDJR7Ma%3D589-cOFaOK2pG-XxUgXzYvje9eMxxSiOm8g%40mail.gmail.com), 6 of 6 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No surviving objection questions the goal: the thread shows reproducible crashes and out-of-bounds reads, and the cost falls only on pageinspect callers. The remaining points are fixable design questions at level 2. standards-compat-1 asks which header checks are needed for memory safety, maintenance-fit-2 asks for a stated scope, and maintenance-fit-1 asks for a shared helper. None of them would stop a committer from reading further.

Not checked: Without the code, the exact fields verify_page_header() checks and which functions call it could not be confirmed. Whether value decoding can still read past a stored value also depends on the code.

## Prior attempts

None found before 2026-08-31.

## Objections a reviewer is likely to raise

### 1. Whole-page header check takes item listings away from forensic users

A DBA investigating a damaged heap block runs heap_page_items() on a page where only pd_flags has a flipped bit, or pd_special is wrong. Before the patch the call returned the line pointers and tuples, which are still in bounds. After the patch it raises an error, and the DBA loses the data needed to salvage rows.

Message #2 adds verify_page_header() with the same checks PageIsVerified() makes and applies it to heap_page_items(), brin_page_items(), GIN and hash. Some of those checks, such as pd_flags or pd_special alignment, have no bearing on whether reading the line pointer array is safe. heap_page_items() already guards each line pointer (message #1), so this is a behaviour change for the main users of the function, who inspect corrupt pages on purpose. A committer would likely ask for clamping pd_lower and checking only fields that bound memory access, and would weigh this more heavily if the patch is back-patched, which the thread does not settle.

**Question:** Which header checks are needed to keep reads inside the page image, and why should a page that fails only the other checks raise an error instead of returning its items?

<sub>fixable, level 2, design; messages [#1](https://www.postgresql.org/message-id/CAGRkXqRWDJR7Ma%3D589-cOFaOK2pG-XxUgXzYvje9eMxxSiOm8g%40mail.gmail.com), [#2](https://www.postgresql.org/message-id/CAGRkXqQ%2BNvzXW%2B2AjkT1ACR-W%2By-cDoy%2BmzLXriMAV2ovVRA5g%40mail.gmail.com); `standards-compat-1`</sub>

### 2. The hardening contract for raw-page input has no stated limit

A user hands brin_page_items() or gist_page_items() a page whose stored values fit inside the tuple but hold a crafted varlena header or opclass datum. The type output function reads past the value and the backend crashes. Each such report then arrives as a bug against a fix that looked complete, and someone has to decide whether to back-patch it.

Message #6 says v4 checks that values fit inside the tuple but not what they contain. Messages #3 and #5 each found a new crash in a version that looked complete, which suggests the cases go further than the patch covers. The proposal does not say where the boundary lies: which functions must never crash on any bytea, and which stay best-effort. Without that boundary, every new pageinspect function and every new on-page format must add checks, and no test fails when one is missing.

**Question:** Which pageinspect functions does the patch promise are safe on arbitrary bytea, and how is value decoding through type output functions covered or excluded from that promise?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#3](https://www.postgresql.org/message-id/G1mth6ql6rdb8blD-g98kuDstGPcjBeGKlJdI6zcqI1jVS3ZXnBJMduZu_pQ8_QlyU5u1KyGDHft_rTr3nfzRLYSAUBUBbSWM7glRXpRmK0%3D%40protonmail.com), [#5](https://www.postgresql.org/message-id/6b27b31e.4e13.1a15d51e4c8.Coremail.jiezhilove%40126.com), [#6](https://www.postgresql.org/message-id/CAGRkXqTV5wo%3D1O-2VKy5jR8bfXujvVJg_ix3%3Dp6kWhvt12he4g%40mail.gmail.com); `maintenance-fit-2`</sub>

### 3. Duplicated header checks can drift from PageIsVerified()

A later commit tightens or changes the header checks in PageIsVerified(), for example a new pd_flags bit or a new rule for all-zero pages. Nothing makes verify_page_header() in pageinspect change with it. pageinspect then rejects valid pages or accepts pages the server treats as corrupt, and someone finds out from a user report.

Message #2 says verify_page_header() repeats the header checks of PageIsVerified() without the checksum. It keeps a second copy in contrib. The author offers a shared helper in bufpage.c, and message #2 got no answer on it. With a shared helper, a change to the rules in one place covers both callers, and a reviewer has no second copy to miss.

**Question:** Would a shared header-check helper in bufpage.c, used by both PageIsVerified() and pageinspect, replace verify_page_header() without changing what either caller accepts?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#2](https://www.postgresql.org/message-id/CAGRkXqQ%2BNvzXW%2B2AjkT1ACR-W%2By-cDoy%2BmzLXriMAV2ovVRA5g%40mail.gmail.com); `maintenance-fit-1`</sub>

## Questions for the author

1. Which header checks in verify_page_header() are needed to keep reads inside the page image, and why should a page failing only the others raise an error?
2. Which pageinspect functions does the series promise are safe on any bytea, and is value decoding through type output functions inside or outside that promise?
3. Is the series meant for back-patching, and if so, how do supported branches handle pages that returned rows before and now raise an error?
4. Does a new all-zero page still return the same result from heap_page_items() and the other item functions as before the patch?
5. Do bt_page_stats(), bt_metap() and hash_metapage_info() get the same header checks, or are they safe without them?

## Touch points

- **security / privileges:** The patch hardens superuser-only functions against crafted bytea input; it adds no function or privilege.
- **backpatchability:** The entry is tagged Bugfix, and the new errors on damaged pages would change behaviour in supported branches if back-patched.
- **maintenance burden ("who maintains this in 10 years"):** verify_page_header() repeats the header checks of PageIsVerified(), and each new on-page format needs its own checks.

## Who would commit it, and when

- Owner: silent. Someone who knows the page layout code in bufpage.c and the index access method page formats that pageinspect decodes.
- Priority: proceed; audience niche; review cost medium. The series fixes reproducible crashes in a tool used by a small group of superusers who inspect damaged pages, and it stays inside contrib/pageinspect. The two patches are about 36 KiB across six access method files, and each check needs someone who knows that page format.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 1 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

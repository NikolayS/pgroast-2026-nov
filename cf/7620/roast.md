# Design pre-review: Fix interval input at minimum signed field values

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7620](https://commitfest.postgresql.org/patch/7620/); [thread](https://www.postgresql.org/message-id/flat/CAEPgAbE%2BV7Wgx-1BYap276v_Ohc7Zk2HV3-U8Wp520Jz84NRmg%40mail.gmail.com), 2 of 2 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). The goal is a reported bug fix (BUG #19746) and nothing at levels 1a or 1b is in question. The one surviving objection, standards-compat-1, is fixable: message #2 lists four inputs that master accepts and v2 rejects when the minimum field comes last. A fix that may go to back branches must not reject input accepted before, so a revised version is needed before commit.

Not checked: The code was not read, so the cause of the field-order regression rests on message #2. Whether other field combinations or input styles regress cannot be judged without the code.

## Prior attempts

None found before 2026-10-20.

## Objections a reviewer is likely to raise

### 1. v2 breaks interval inputs that master accepts when the minimum field comes last

An application or a stored view default uses an interval literal such as '1 week -2147483648 days ago'. Master accepts it. After a minor release with v2 it fails with "interval field value out of range". The user finds out when a query, a restore or a CHECK constraint errors after upgrade.

Message #2 shows four inputs that work on unpatched master and fail with v2. In each, DecodeInterval negates the minimum field first because it reads right to left. A bugfix meant for back branches must not reject input that minor releases accepted before, because back-branch users do not expect behaviour changes. The commit message promises that cancellation is preserved, but v2 keeps it for only one field order, and no v3 is visible.

**Question:** Does a revised version accept all four inputs from message #2, with regression tests for both field orders?

<sub>fixable, level 2, backpatchability; messages [#2](https://www.postgresql.org/message-id/CA%2BY2Bmd00%2BGkrJ6-4BR_1wrKUtQo-BoEDzObTZKdhscA5%3DXkbA%40mail.gmail.com); `standards-compat-1`</sub>

## Questions for the author

1. Will a revised version accept the four inputs from message #2 that work on master and fail with v2?
2. How does the revised accumulation make the result independent of the order of fields in the input string?
3. Will the regression tests cover cancellation near the limits with the minimum field both first and last in the string?
4. Which branches is the fix meant for, and is a separate version planned for REL_15 and REL_16?
5. Is the change from "field value out of range" to "invalid input syntax" for a repeated unit intended?

## Touch points

- **pg_dump / restore round-trip:** The fix lets text dumps of minimum-value intervals restore, but v2 rejects some interval text that master accepts.
- **backpatchability:** It is a bug fix in DecodeInterval; v2 applies to master and REL_17-19 but not REL_15/16, and changes accepted input.

## Who would commit it, and when

- Owner: silent. Someone who knows the datetime and interval input code in datetime.c and the back-patch rules for input behaviour.
- Priority: fix design first; audience niche; review cost low. The patch is small and touches only datetime.c and the interval tests, and it fixes a reported dump and restore failure at boundary values. Message #2 found a field-order regression against master, so v2 needs a revision before further review.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 3 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

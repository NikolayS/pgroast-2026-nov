# Design pre-review: Doc: clarify pg_locks descriptions of classid/objid/objsubid

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7309](https://commitfest.postgresql.org/patch/7309/); [thread](https://www.postgresql.org/message-id/flat/CA%2Br%3DywWUHOPgHp-tegoETpfjiEkE5Ytzi4g9orcKav8bWym%2B_Q%40mail.gmail.com), 2 of 2 messages read; no code read.

## Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium). No objection survived at levels 1a, 1b or 2. The patch corrects column descriptions in system-views.sgml that contradict the per-lock-type prose under the same table, and it changes no behavior. A committer agreed with the direction in message #2 and registered as committer. Confidence is medium because the v2 wording is not shown.

Not checked: Only the v1 hunk was shown, so the exact v2 wording could not be checked. The claim that pg_lock_status() fills these columns for the three lock types rests on message #1 and was not verified against the code.

## Prior attempts

None found before 2026-03-30.

## Objections a reviewer is likely to raise

No objection held up in the counter-check.

## Questions for the author

1. Should the fix be back-patched, and to which branches, given that apply transaction locks do not exist in all supported branches?
2. Which lock types still show null in classid, objid and objsubid, and does the v2 text still tell the reader that?
3. Does the per-lock-type prose in v2 describe the use of classid, objid and objsubid for every lock type that fills them, including speculative insertion locks?

## Touch points

- **backpatchability:** The doc error exists in released branches, and the thread does not say whether the fix is back-patched.

## Who would commit it, and when

- Owner: qualified committer engaged. Fujii Masao posted v2 in message #2 and is registered as committer on the CommitFest entry.
- Priority: proceed; audience a sizeable minority; review cost low. The patch touches one documentation file, corrects a description that contradicts the prose below it, and a committer has taken it on. Readers of pg_locks who use advisory locks or logical replication gain.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

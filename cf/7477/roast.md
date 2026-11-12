# Design pre-review: reindexdb: Fix schema name lookup in parallel mode

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7477](https://commitfest.postgresql.org/patch/7477/); [thread](https://www.postgresql.org/message-id/flat/TY5PR01MB1844509B94E3179A2A2434082FA9E2%40TY5PR01MB18445.jpnprd01.prod.outlook.com), 6 of 6 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No surviving objection questions the fix on master: it is a small client-side bug fix that copies the regnamespace lookup vacuumdb already uses. The one surviving objection, maintenance-fit-1, concerns the request to backpatch through 15. Today `-j -S Sales` reindexes a schema created as "Sales"; after the patch that job fails. Only a committer can decide whether a minor release may do that.

Not checked: The code was not read. Not checked: whether the catalog query quotes the argument safely and is schema-qualified, and how the error for a missing schema reads.

## Prior attempts

None found before 2026-10-13.

## Objections a reviewer is likely to raise

### 1. Backpatching turns silent successes into failures in minor releases

A site runs a nightly `reindexdb -j 4 -S Sales` job on 16.x, where `Sales` is a schema created as `"Sales"`. After a routine minor update the job exits non-zero and its scheduler raises an alert. Nobody changed the job, and the release notes are the only place the cause is explained.

The fix itself is small and copies the vacuumdb approach, so on master the cost is low. The open part is message #5's request to backpatch through 15, which changes exit codes for scripts that work today, even if they reindex nothing. Message #4 agrees these scripts will now fail and asks for a release note. Message #6 says no port to 15-17 is posted or tested, so the backpatch is not yet ready for review.

**Question:** Would a master-only fix, or a backpatch that only warns about unmatched schemas, be enough, and if not, why must back branches start failing these scripts?

<sub>fixable, level 2, backpatchability; messages [#4](https://www.postgresql.org/message-id/CADMs5OOOyzwMa7LYKZU_YxjbCVbx-5%3DscF8%2BSq7x4kG_s34Mpg%40mail.gmail.com), [#5](https://www.postgresql.org/message-id/TY5PR01MB1844575BDDB2E1B415E75A094FA722%40TY5PR01MB18445.jpnprd01.prod.outlook.com), [#6](https://www.postgresql.org/message-id/20261110131410.60486-1-joaovictor.rito%40gmail.com); `maintenance-fit-1`</sub>

## Questions for the author

1. Why must branches 15 to 18 start failing scripts that pass today, and would a master-only fix or a back-branch warning for unmatched schemas be enough?
2. Can you post and test patch versions that apply to 15, 16 and 17, since v2 does not apply there?
3. What would the release note say to users whose `-j -S` job matched a mixed-case schema without double quotes and will now fail?
4. Does the reindexdb documentation need a change to say that -S arguments follow identifier rules and accept numeric OIDs in parallel mode?
5. Is it acceptable that serial mode keeps REINDEX SCHEMA syntax while parallel mode uses regnamespace syntax, or should both modes use regnamespace as vacuumdb does?

## Touch points

- **SQL-standard conflict or non-standard syntax:** The regnamespace input accepts numeric OIDs and '-' and does not recognize U& identifiers or SQL comments, unlike REINDEX SCHEMA.
- **backpatchability:** The author asks to backpatch through 15, which changes exit codes for existing `reindexdb -S ... -j` scripts; v2 does not apply to 15 to 17.
- **maintenance burden ("who maintains this in 10 years"):** A backpatch needs separate, tested patch versions for 15 to 17, which are not posted.

## Who would commit it, and when

- Owner: silent. A committer who knows the src/bin/scripts client tools and the back-branch policy for behaviour changes would have to own it; none has replied in the visible thread.
- Priority: proceed; audience niche; review cost low. The patch touches reindexdb.c and one TAP test file, is about 3.5 KB, and reuses the vacuumdb approach. It affects only users of -S with -j, and the remaining decision is the backpatch scope.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 1 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

# Design pre-review: Delete GIN posting-tree pages without excessive locking

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7199](https://commitfest.postgresql.org/patch/7199/); [thread](https://www.postgresql.org/message-id/flat/DDD3E964-5047-499C-8208-594F182D140D%40yandex-team.ru), 1 of 1 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: low). Level 1b fails on demand-1. Message #1 names no current workload that stalls behind the root cleanup lock, and it says the 2016 case was solved by what remained after fd83c83d094. The author asks whether anyone is interested, and nobody has replied. This is a "not now, show demand" result; the design objections at level 2 were not evaluated.

Not checked: The code and the truncated part of the README were not read, so lock ordering, inserter recovery and WAL replay are unchecked. Three level 2 objections survived (prior-attempts-1, replication-ondisk-1, maintenance-fit-2) and remain open.

## Prior attempts

None found before 2026-07-21.

## Objections a reviewer is likely to raise

### 1. No current user is shown to suffer from the root cleanup lock

A committer takes on a new posting-tree deletion protocol in GIN vacuum. No workload is known to be blocked by the second pass today, so the change has no measured benefit, and users only meet its risk.

Message #1 says the only known case, Yandex.Mail in 2016, was solved by what remained after fd83c83d094. It offers to continue only "if anyone actually suffers from this". The thread has no replies, no field reports and no numbers on how long the root cleanup lock blocks searches. A reference to another thread that "reminded" the author is not a report of a user problem.

**Question:** Which current workloads see searches or inserts stalled behind the posting-tree deletion pass, and for how long, on which index sizes?

<sub>fixable, level 1b, demand; messages [#1](https://www.postgresql.org/message-id/DDD3E964-5047-499C-8208-594F182D140D%40yandex-team.ru); `demand-1`</sub>

## Questions for the author

1. Which current workloads see searches or insertions stall behind the posting tree deletion pass, for how long, and on which index sizes?
2. What was incorrect in the locking reverted by fd83c83d094, and which part of the new protocol rules out that failure?
3. How does replay of a posting tree page deletion stay safe for a hot standby scan that holds a pin on the deleted leaf?
4. After a mass delete, how much posting tree space stays unreclaimed compared with the current two-pass deletion, and what does that cost later scans?
5. What do before and after measurements show for search and insertion latency during vacuum of a large posting tree?

## Touch points

- **WAL / replication / standby:** Page deletion now runs without the root cleanup lock, and replay on a hot standby cannot see the primary's pin check.
- **maintenance burden ("who maintains this in 10 years"):** A new posting tree deletion protocol in ginvacuum.c and ginbtree.c that later GIN locking changes must keep intact.
- **performance on the hot path:** The patch changes ginbtree.c, which insertions and searches use, and leaves some empty pages for later scans to walk.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** Message #1 shows no current user who needs the change.

## Who would commit it, and when

- Owner: silent. It needs a committer who knows GIN posting tree locking, page deletion and its WAL replay. Nobody has replied to message #1.
- Not now, show demand first: A report from at least one current workload where searches or insertions stall behind the root cleanup lock, with stall times and index sizes. A reproducible benchmark that shows the stall before the patch and its absence after would also count.
- Priority: park; audience unclear; review cost high. The patch adds no user-visible surface, but the locking protocol needs care for ever and expert review of GIN concurrency. No current user is shown and nobody has replied, so it can wait until someone reports the stall.

---

[roast-full.md](roast-full.md) has 3 more objection(s) that held, 2 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

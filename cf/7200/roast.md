# Design pre-review: GiST intrapage indexing

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7200](https://commitfest.postgresql.org/patch/7200/); [thread](https://www.postgresql.org/message-id/flat/7780A07B-4D04-41E2-B228-166B41D07EEE%40yandex-team.ru), 2 of 2 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). No surviving objection is fatal, and message #2 itself asks for agreement on the on-disk representation before code review. That agreement needs evidence for the workloads the commit message names (demand-1), a check that skip keys cover their groups (maintenance-fit-1), and an answer on the Penalty assumption (maintenance-fit-2). Nobody but the author has posted, so the design is not yet discussed.

Not checked: Without the code, nobody can judge whether concurrent scans stay correct while an insert or VACUUM rebuilds a group, or whether every internal-page update path rebuilds the group key. How older page readers handle the marker is also unchecked.

## Prior attempts

None found before 2018-02-12.

## Objections a reviewer is likely to raise

### 1. The motivating workloads are not measured

A committer accepts a permanent change to the GiST internal page format on the strength of point_ops with all data in cache. Later, users of costly operator classes or non-default block sizes, the cases the proposal names, see little or no gain. The project still carries the format and its maintenance cost in inserts, splits, both builds and VACUUM.

The commit message justifies the change by operator classes with costly keys and by high fanout, and message #1 adds large block sizes. Message #2 measures neither and says the costly-Consistent gain "is untested". The only numbers cover one cheap operator class, in cache, with a gain of about 1.13x on point queries and 1.12x on inserts. The 2018 numbers in message #1 showed a different balance, and message #2 attributes the gap to measurement error, so the size of the gain for real users is not yet shown.

**Question:** For which widely used operator classes or block sizes does Consistent or Penalty cost dominate GiST time, and what speedup does v3 give there with data larger than shared_buffers?

<sub>fixable, level 1b, demand; messages [#1](https://www.postgresql.org/message-id/7780A07B-4D04-41E2-B228-166B41D07EEE%40yandex-team.ru), [#2](https://www.postgresql.org/message-id/2AF68DEC-8CD1-4460-B02D-ACE1850E6484%40yandex-team.ru); `demand-1`</sub>

### 2. Union key coverage must hold on every internal-page update path

A later patch adds a new path that changes an internal page, such as a new page-deletion step or a parent-key adjustment, and does not rebuild the group union. A scan then rejects a group whose skip key no longer covers a downlink, and queries silently miss rows until someone runs a full comparison against a seqscan.

The README hunk makes correctness depend on every page update rebuilding the affected group, but the patch adds no check for that. Today a stale or missing downlink key is the only way GiST loses rows. The patch adds a second key layer whose staleness has the same effect. The touched files do not include amcheck, so nothing would verify that a skip key covers its group.

**Question:** Will amcheck verify that each skip key covers its group, so a future path that forgets the rebuild fails a check before it returns wrong results?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#2](https://www.postgresql.org/message-id/2AF68DEC-8CD1-4460-B02D-ACE1850E6484%40yandex-team.ru); `maintenance-fit-1`</sub>

### 3. Subtree skipping assumes a penalty property opclasses do not promise

A group's union is a wide, flat box, and a new point lies just above the middle of it. The enlargement penalty for the union is large, while one small member box needs almost no enlargement, so insertion skips the best subtree. Tree quality then degrades over time, and nobody connects later slow scans to this skipping rule.

Message #1 states the rule assumes an enlarged key never has a higher penalty than its members. Area-enlargement penalties, which point_ops uses, do not guarantee this, and third-party opclasses were written without that contract. The insert benchmark in message #2 measures speed only, not the tree quality produced over many inserts.

**Question:** Which penalty functions satisfy the monotonicity assumption, and how does tree quality after many inserts compare with master for box and point opclasses?

<sub>fixable, level 2, ABI / extension API; messages [#1](https://www.postgresql.org/message-id/7780A07B-4D04-41E2-B228-166B41D07EEE%40yandex-team.ru), [#2](https://www.postgresql.org/message-id/2AF68DEC-8CD1-4460-B02D-ACE1850E6484%40yandex-team.ru); `maintenance-fit-2`</sub>

### 4. Page inspection tools will report skip tuples as ordinary downlinks

A DBA runs pageinspect's gist_page_items on an internal page. It lists entries with an invalid block number and a small offset next to real downlinks. External tools that walk GiST pages count these entries as children or report the page as corrupt.

The patch's touched-files list covers only access/gist and a new test module, not contrib/pageinspect or contrib/amcheck. The new marker reuses t_tid fields that existing readers treat as a downlink. Message #2 asks for opinions on exactly this on-disk representation, so the readers it affects belong in that discussion.

**Question:** How will pageinspect, any GiST checking in amcheck, and third-party page readers identify skip tuples and show or verify them?

<sub>fixable, level 2, design; messages [#2](https://www.postgresql.org/message-id/2AF68DEC-8CD1-4460-B02D-ACE1850E6484%40yandex-team.ru); `standards-compat-2`</sub>

### 5. Group rebuild adds Union calls to every internal-page insert

An application inserts into a GiST index whose opclass has an expensive Union, such as signature-based tsvector or intarray. Each downlink adjustment now also recomputes a group union over up to 16 keys, so insert throughput can drop where the benchmark showed a gain.

The commit message says page updates rebuild the affected group from its real tuples. The only insert numbers use point_ops, where Union is cheap. Message #2 says expensive opclasses should gain on Consistent but does not discuss the matching cost on Union during inserts.

**Question:** What is the insert cost with an opclass whose Union is expensive, compared with master?

<sub>fixable, level 2, performance on the hot path; messages [#2](https://www.postgresql.org/message-id/2AF68DEC-8CD1-4460-B02D-ACE1850E6484%40yandex-team.ru); `maintenance-fit-4`</sub>

## Questions for the author

1. What speedup does v3 give for an operator class with an expensive Consistent, with data larger than shared_buffers, and at a non-default block size?
2. Will amcheck or another check verify that each skip key covers its group, so a later update path that omits the rebuild fails before a scan misses rows?
3. Which Penalty functions satisfy the assumption from message #1, and how does tree quality after many inserts compare with master for box and point operator classes?
4. How will pageinspect's gist_page_items, amcheck and third-party page readers identify skip tuples and show or verify them?
5. What is the insert cost compared with master for an operator class with an expensive Union, and how should the group size of 16 be chosen?

## Touch points

- **on-disk format / pg_upgrade:** GiST internal pages gain skip tuples, marked by InvalidBlockNumber in t_tid with the group size in ip_posid.
- **WAL / replication / standby:** Existing GiST WAL records carry page states that now include skip tuples; the test covers standby replay and crash recovery.
- **ABI / extension API:** Group skipping on insert relies on a Penalty property of operator classes, and page readers outside access/gist see a new tuple kind.
- **maintenance burden ("who maintains this in 10 years"):** Inserts, splits, both build methods and VACUUM must keep skip tuples correct, and so must every later change to internal pages.
- **performance on the hot path:** The patch changes GiST internal-page scans and subtree choice, and adds a group Union rebuild on page updates.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** see objection demand-1: The motivating workloads are not measured

## Who would commit it, and when

- Owner: silent. It needs a committer who knows the GiST insert, split, build and VACUUM code and its WAL replay. Only the author has posted in the thread.
- Priority: needs champion; audience a sizeable minority; review cost high. Any GiST user could gain, and point_ops in cache shows 1.12x to 1.41x, but the patch changes the page format and every internal-page update path. The author asked for opinions on the representation and no one has answered.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 3 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

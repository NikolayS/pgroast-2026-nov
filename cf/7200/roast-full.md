# Design pre-review: GiST intrapage indexing

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7200](https://commitfest.postgresql.org/patch/7200/) (Needs review); authors: Andrey Borodin (x4m)
- Thread: [\[WiP\] GiST intrapage indexing](https://www.postgresql.org/message-id/flat/7780A07B-4D04-41E2-B228-166B41D07EEE%40yandex-team.ru), started 2018-02-12
- Messages read: 2 of 2
- Patch prose read: v3-0001-Add-intrapage-indexing-to-GiST.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

No surviving objection is fatal, and message #2 itself asks for agreement on the on-disk representation before code review. That agreement needs evidence for the workloads the commit message names (demand-1), a check that skip keys cover their groups (maintenance-fit-1), and an answer on the Penalty assumption (maintenance-fit-2). Nobody but the author has posted, so the design is not yet discussed.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, nobody can judge whether concurrent scans stay correct while an insert or VACUUM rebuilds a group, or whether every internal-page update path rebuilds the group key. How older page readers handle the marker is also unchecked.

## The proposal, restated

**Problem.** Scanning a GiST internal page applies the consistent function to every downlink, and insertion computes penalty for every downlink. This costs most for operator classes with expensive keys and for pages with high fanout. The author also expects it to protect GiST from slowdown on large block sizes.

**Proposed design.** Internal pages carry skip tuples, each holding the union key of the downlinks that follow it, so scans and subtree selection can reject a whole group at once. A skip tuple is marked by InvalidBlockNumber in t_tid, with the group size in ip_posid. Skip tuples are derived metadata kept up to date by inserts, splits, both build methods and VACUUM, and existing WAL records log full page states.

**User-visible surface.**
- On-disk GiST internal page format gains skip tuples; existing pages need no conversion, per the author.

**What the author claims.**
- On 5M uniform points with point_ops, all data in cache, a join runs 1.41x faster, point queries 1.13x, and inserts 1.12x.
- Index size grows by about 0.05%, and 0.2% on a larger index.
- Build time shows no measurable difference, because master varies by 27% run to run on the test machine.
- Operator classes with a more expensive Consistent should gain more; the author has not tested this.
- The format does not use INDEX_AM_RESERVED_BIT and needs no new WAL record type or pg_upgrade step.

**Thread state.** Only the author has posted: a 2018 first version and a 2026 revival with v3. He asks for opinions on the on-disk representation before code review, and CFbot reports the patch needs a rebase.

## 2. Prior attempts

No precedent found (corpus searched for items before 2018-02-12; 21 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. The motivating workloads are not measured

fixable · level 1b, rubric item 3 · demand, doesn't fit what Postgres is about (extension / external tool / already solved), maintenance burden ("who maintains this in 10 years") · `demand-1`

A committer accepts a permanent change to the GiST internal page format on the strength of point_ops with all data in cache. Later, users of costly operator classes or non-default block sizes, the cases the proposal names, see little or no gain. The project still carries the format and its maintenance cost in inserts, splits, both builds and VACUUM.

The commit message justifies the change by operator classes with costly keys and by high fanout, and message #1 adds large block sizes. Message #2 measures neither and says the costly-Consistent gain "is untested". The only numbers cover one cheap operator class, in cache, with a gain of about 1.13x on point queries and 1.12x on inserts. The 2018 numbers in message #1 showed a different balance, and message #2 attributes the gap to measurement error, so the size of the gain for real users is not yet shown.

Thread: [#1 (Andrey Borodin, 2018-02-12)](https://www.postgresql.org/message-id/7780A07B-4D04-41E2-B228-166B41D07EEE%40yandex-team.ru), [#2 (Andrey Borodin, 2026-08-03)](https://www.postgresql.org/message-id/2AF68DEC-8CD1-4460-B02D-ACE1850E6484%40yandex-team.ru)

**Question:** For which widely used operator classes or block sizes does Consistent or Penalty cost dominate GiST time, and what speedup does v3 give there with data larger than shared_buffers?

Counter-check: held. Message #2 says the costly-Consistent case is untested, and no numbers cover large block sizes or data larger than shared_buffers. Saying so in the thread does not supply the evidence for the workloads the commit message names.

### 2. Union key coverage must hold on every internal-page update path

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years"), on-disk format / pg_upgrade · `maintenance-fit-1`

A later patch adds a new path that changes an internal page, such as a new page-deletion step or a parent-key adjustment, and does not rebuild the group union. A scan then rejects a group whose skip key no longer covers a downlink, and queries silently miss rows until someone runs a full comparison against a seqscan.

The README hunk makes correctness depend on every page update rebuilding the affected group, but the patch adds no check for that. Today a stale or missing downlink key is the only way GiST loses rows. The patch adds a second key layer whose staleness has the same effect. The touched files do not include amcheck, so nothing would verify that a skip key covers its group.

Thread: [#2 (Andrey Borodin, 2026-08-03)](https://www.postgresql.org/message-id/2AF68DEC-8CD1-4460-B02D-ACE1850E6484%40yandex-team.ru)

**Question:** Will amcheck verify that each skip key covers its group, so a future path that forgets the rebuild fails a check before it returns wrong results?

Counter-check: held. The README makes scan correctness depend on every internal-page update rebuilding the group key. The TAP test covers today's paths only, and the material names no check that would catch a later path that omits the rebuild.

### 3. Subtree skipping assumes a penalty property opclasses do not promise

fixable · level 2, rubric item 11 · ABI / extension API, maintenance burden ("who maintains this in 10 years") · `maintenance-fit-2`

A group's union is a wide, flat box, and a new point lies just above the middle of it. The enlargement penalty for the union is large, while one small member box needs almost no enlargement, so insertion skips the best subtree. Tree quality then degrades over time, and nobody connects later slow scans to this skipping rule.

Message #1 states the rule assumes an enlarged key never has a higher penalty than its members. Area-enlargement penalties, which point_ops uses, do not guarantee this, and third-party opclasses were written without that contract. The insert benchmark in message #2 measures speed only, not the tree quality produced over many inserts.

Thread: [#1 (Andrey Borodin, 2018-02-12)](https://www.postgresql.org/message-id/7780A07B-4D04-41E2-B228-166B41D07EEE%40yandex-team.ru), [#2 (Andrey Borodin, 2026-08-03)](https://www.postgresql.org/message-id/2AF68DEC-8CD1-4460-B02D-ACE1850E6484%40yandex-team.ru)

**Question:** Which penalty functions satisfy the monotonicity assumption, and how does tree quality after many inserts compare with master for box and point opclasses?

Counter-check: held. Message #1 states the assumption, and area-enlargement penalties do not guarantee it when the new key lies outside a wide union. Only index quality is affected, and the uniform-point benchmark would not show it, so it stays fixable.

### 4. Page inspection tools will report skip tuples as ordinary downlinks

fixable · level 2, rubric item 13 · design, ABI / extension API, maintenance burden ("who maintains this in 10 years") · `standards-compat-2`

A DBA runs pageinspect's gist_page_items on an internal page. It lists entries with an invalid block number and a small offset next to real downlinks. External tools that walk GiST pages count these entries as children or report the page as corrupt.

The patch's touched-files list covers only access/gist and a new test module, not contrib/pageinspect or contrib/amcheck. The new marker reuses t_tid fields that existing readers treat as a downlink. Message #2 asks for opinions on exactly this on-disk representation, so the readers it affects belong in that discussion.

Thread: [#2 (Andrey Borodin, 2026-08-03)](https://www.postgresql.org/message-id/2AF68DEC-8CD1-4460-B02D-ACE1850E6484%40yandex-team.ru)

**Question:** How will pageinspect, any GiST checking in amcheck, and third-party page readers identify skip tuples and show or verify them?

Counter-check: held. The touched-files list has nothing under contrib, and pageinspect's gist_page_items reads internal tuples today. Message #2 asks for opinions on this representation, so how readers outside access/gist identify skip tuples is a fair question.

### 5. Group rebuild adds Union calls to every internal-page insert

fixable · level 2, rubric item 8 · performance on the hot path · `maintenance-fit-4`

An application inserts into a GiST index whose opclass has an expensive Union, such as signature-based tsvector or intarray. Each downlink adjustment now also recomputes a group union over up to 16 keys, so insert throughput can drop where the benchmark showed a gain.

The commit message says page updates rebuild the affected group from its real tuples. The only insert numbers use point_ops, where Union is cheap. Message #2 says expensive opclasses should gain on Consistent but does not discuss the matching cost on Union during inserts.

Thread: [#2 (Andrey Borodin, 2026-08-03)](https://www.postgresql.org/message-id/2AF68DEC-8CD1-4460-B02D-ACE1850E6484%40yandex-team.ru)

**Question:** What is the insert cost with an opclass whose Union is expensive, compared with master?

Counter-check: held. The commit message says page updates rebuild the affected group from its real tuples. The only insert numbers use point_ops, so the cost with an expensive Union is not shown.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | GiST internal pages gain skip tuples, marked by InvalidBlockNumber in t_tid with the group size in ip_posid. |
| WAL / replication / standby | **yes** | Existing GiST WAL records carry page states that now include skip tuples; the test covers standby replay and crash recovery. |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | Group skipping on insert relies on a Penalty property of operator classes, and page readers outside access/gist see a new tuple kind. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | Inserts, splits, both build methods and VACUUM must keep skip tuples correct, and so must every later change to internal pages. |
| performance on the hot path | **yes** | The patch changes GiST internal-page scans and subtree choice, and adds a group Union rebuild on page updates. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | see objection demand-1: The motivating workloads are not measured |

## 5. Questions the author must answer before anyone reviews code

1. What speedup does v3 give for an operator class with an expensive Consistent, with data larger than shared_buffers, and at a non-default block size?
2. Will amcheck or another check verify that each skip key covers its group, so a later update path that omits the rebuild fails before a scan misses rows?
3. Which Penalty functions satisfy the assumption from message #1, and how does tree quality after many inserts compare with master for box and point operator classes?
4. How will pageinspect's gist_page_items, amcheck and third-party page readers identify skip tuples and show or verify them?
5. What is the insert cost compared with master for an operator class with an expensive Union, and how should the group size of 16 be chosen?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. It needs a committer who knows the GiST insert, split, build and VACUUM code and its WAL replay. Only the author has posted in the thread.
- **Liability:** The owner signs for a GiST internal page format that later releases must read for as long as pg_upgrade carries such indexes. A skip key that fails to cover its group makes scans miss rows.
- **Author follow-up:** The author revived the thread in 2026 with v3 and moved the CommitFest entry forward. The rebase that CFbot asked for on 2026-10-12 is not in the thread.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs champion
- **Audience:** a sizeable minority
- **Maintenance burden:** high
- **Review cost:** high
- **Why:** Any GiST user could gain, and point_ops in cache shows 1.12x to 1.41x, but the patch changes the page format and every internal-page update path. The author asked for opinions on the representation and no one has answered.

## Appendix

### Refuted in the counter-check

- `standards-compat-1` Insert-time group skipping assumes a penalty property opclasses never promised. Refuted (not specific): Duplicate of maintenance-fit-2, which gives the more concrete geometric case and also asks for a tree-quality comparison.
- `replication-ondisk-1` Code outside access/gist still assumes every internal tuple is a downlink. Refuted (not specific): Duplicate of standards-compat-2. This version also states that skip tuples carry no page flag, which the material does not say.
- `maintenance-fit-3` Permanent on-disk format change rests on one in-cache benchmark. Refuted (not specific): Duplicate of demand-1, which states the same evidence gap. The added regret argument is weaker, because the README says removing skip tuples leaves a valid ordinary GiST page.

### Review angles

- prior-attempts: 0 objection(s). No listed precedent matches, so no earlier rejection reason applies here. Message #1 points to an earlier proposal to build a small GiST inside each page and reports two design changes since then: two levels instead of many, and a counter instead of a tid so no pg_upgrade is needed. The 2018 version drew no replies, which shows no engagement but records no rejection reason.
- standards-compat: 2 objection(s). There is no SQL, GUC, psql or pg_dump surface. Indexes created on older versions need no conversion, and downgrade across major versions is not supported anyway.
- replication-ondisk: 1 objection(s). WAL looks sound as described: existing records log full page states, and the test covers standby replay and crash recovery. Old pages need no conversion, and physical standbys must run the same major version anyway, so the only issue left on this angle is readers outside access/gist.
- security: 0 objection(s). The proposal has no privilege or exposure aspect: it adds no SQL surface, privilege, setting or view. Skip tuples call the same Consistent, Union and Penalty functions the operator class already runs, on keys derived from existing downlinks. Downgrade and tool handling of the new marker are format concerns for other angles.
- maintenance-fit: 4 objection(s). Objections found; see the list.
- already-exists: 0 objection(s). Postgres has no feature, setting or documented idiom that groups downlinks inside a GiST internal page so Consistent or Penalty can reject a group at once. An extension cannot add this, because the page layout belongs to the GiST access method in core. The sorted build and the buffered build improve index quality but do not cut the number of Consistent calls per page, and an alternative such as SP-GiST is a different index type.
- demand: 1 objection(s). Faster GiST scans and inserts would help any GiST user, so broad demand is plausible. The objection is that the material does not yet show gains for the workloads that motivate the change.

### Precedents retrieved

- P1: [Pageinspect - add functions on GIN and GiST indexes from gevel](https://commitfest.postgresql.org/patch/1207/) (cf, Returned with feedback, 2017-11-27) - judged not relevant; reason summarised from its thread
- P2: [Delete pages during GiST VACUUM](https://commitfest.postgresql.org/patch/1416/) (cf, Returned with feedback, 2018-02-07) - judged not relevant
- P3: [New gist vacuum](https://commitfest.postgresql.org/patch/371/) (cf, Returned with feedback, 2016-01-31) - judged not relevant
- P4: [Cube extension GiST penalty function improvement](https://commitfest.postgresql.org/patch/782/) (cf, Returned with feedback, 2016-10-03) - judged not relevant; reason summarised from its thread
- P5: [enum support in btree_gin and btree_gist](https://commitfest.postgresql.org/patch/784/) (cf, Returned with feedback, 2016-12-02) - judged not relevant; reason summarised from its thread
- P6: [support for box type in SP-GiST index](https://commitfest.postgresql.org/patch/419/) (cf, Returned with feedback, 2016-01-28) - judged not relevant; reason summarised from its thread
- P7: [Better tracking of free space during SP-GiST index build](https://commitfest.postgresql.org/patch/739/) (cf, Returned with feedback, 2016-09-28) - judged not relevant; reason summarised from its thread
- P8: [Compress method for SP-GiST](https://commitfest.postgresql.org/patch/1298/) (cf, Returned with feedback, 2017-11-30) - judged not relevant; reason summarised from its thread
- P9: [pageinspect: Hash index support](https://commitfest.postgresql.org/patch/758/) (cf, Returned with feedback, 2016-12-20) - judged not relevant
- P10: [Revert "Auto-tune effective_cache size to be 4x shared buffers"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a16d421ca4fc639929bc964b2585e8382cf16e33) (revert, Reverted, 2014-05-09) - judged not relevant
- P11: [PageRepairFragmentation optimization](https://commitfest.postgresql.org/patch/90/) (cf, Returned with feedback, 2015-07-26) - judged not relevant; reason summarised from its thread
- P12: [Revert the GinMaxItemSize calculation so that we fit 3 tuples per page.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=930fd68455576dada48521a6cef642caddea3c63) (revert, Reverted, 2014-12-30) - judged not relevant
- P13: [Predicate locking in SP-Gist index](https://commitfest.postgresql.org/patch/1215/) (cf, Returned with feedback, 2017-10-02) - judged not relevant; reason summarised from its thread
- P14: [group locking](https://commitfest.postgresql.org/patch/23/) (cf, Rejected, 2015-01-19) - judged not relevant
- P15: [Revert exporting of internal GUC variable "data_directory".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=da33a3894e0fc440ac53cdc0f2e360e703b13b8c) (revert, Reverted, 2015-05-29) - judged not relevant
- P16: [On-disk bitmap indexes](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Features_We_Do_Not_Want) (do_not_want, Not wanted, date unknown) - judged not relevant
- P17: [Revert "Use group updates when setting transaction status in clog."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=390811750d1647ed81a462da3af7f1fd66166e95) (revert, Reverted, 2017-03-10) - judged not relevant
- P18: [Continue to allow VACUUM to mark last block of index dirty even when there is no work to do.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2d2841a56c8fa37a5dd5c6d33488ba6ca37116ff) (revert, Partially reverted, 2011-11-22) - judged not relevant
- P19: [Rather than mmap()-ing in 8k pages, mmap()'ing entire files into a backend.](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Miscellaneous_Performance) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P20: [Indeterminate behavior for the GROUP BY clause](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Features_We_Do_Not_Want) (do_not_want, Not wanted, date unknown) - judged not relevant
- P21: [MONEY dumps in a locale-specific format making it difficult to restore to a system with a different locale](https://wiki.postgresql.org/wiki/Not_Worth_Doing#MONEY_Data_Type) (do_not_want, Not worth doing, date unknown) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 13; wall time: 138 s; cost reported by the CLI: $1.37
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 7 from cache, 1 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 0 of 21 kept
- Tone check: 84 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:28:04.388Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

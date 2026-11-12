# Design pre-review: synchronized_standby_slots behavior inconsistent with quorum-based synchronous replication

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7373](https://commitfest.postgresql.org/patch/7373/) (Needs review); authors: Ashutosh Sharma (ashu_coek)
- Thread: [synchronized_standby_slots behavior inconsistent with quorum-based synchronous replication](https://www.postgresql.org/message-id/flat/CAHg%2BQDfU7rOebrLDESPpHSgdiadKbpCOmBokcbmM6Gr%2BA5VobQ%40mail.gmail.com), started 2026-02-24
- Messages read: 99 of 99
- Patch prose read: 0001-Add-quorum-support-to-synchronized_standby_slots.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

No surviving objection is fatal, and demand for ANY N is shown in messages #1, #2 and #29. Three fixable design points remain at level 2: replication-ondisk-1 (which standby is a safe failover target under ANY), standards-compat-1 (legal slot names that the syncrep scanner may reject) and maintenance-fit-1 (one grammar, two bare-list meanings). The change to the shared syncrep grammar needs sign-off from a committer who knows that code. demand-1 affects only FIRST N.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Only the first patch's prose was read, and no code. Not checked: whether quoted slot names still pass slot-name validation, which names the scanner rejects, and where SYNC_REP_DEFAULT is mapped for synchronous_standby_names.

## The proposal, restated

**Problem.** synchronized_standby_slots waits for every listed physical slot, so logical decoding of failover slots stalls when one standby is down. This happens even when synchronous_standby_names uses ANY M quorum commit and the commit is already durable. The author says this also makes WAL pile up on the primary.

**Proposed design.** The GUC reuses the synchronous_standby_names parser and accepts ANY N (...), FIRST N (...) and N (...); a bare list keeps the old wait-for-all meaning. A first patch adds a SYNC_REP_DEFAULT method to the syncrep parser so a bare list differs from explicit FIRST. ANY skips missing, invalidated and logical slots and counts caught-up ones; FIRST waits on a valid lagging slot rather than skipping to a lower-priority one.

**User-visible surface.**
- synchronized_standby_slots accepts ANY num_sync (slot, ...), FIRST num_sync (...) and num_sync (...); slot names FIRST or ANY need double quotes.
- Duplicate slot names are removed silently; num_sync larger than the unique slot count is a GUC check error.
- An invalid slot name like 'invalid*' now gives a syntax error from the syncrep parser, not the slot-name error.
- New WARNING messages list unusable slots, and a DEBUG1 message reports lagging slots with their restart_lsn.
- Docs change for pg_logical_slot_get_changes and pg_replication_slot_advance, which block until the required slots confirm.

**What the author claims.**
- The default behaviour does not change; a bare list still waits for all slots.
- Choosing the right standby at failover is the job of the failover tool, as with quorum synchronous_standby_names.
- A reviewer from Fujitsu reports that many customers have asked for this.
- The new TAP test runs in about 6 seconds, using a replication connection that holds the slot without advancing it.

**Thread state.** Several reviewers (shveta malik, Hou zj, Amit Kapila, Dilip Kumar, Shlok Kyal) went through many versions, and on 2026-06-19 shveta called the three-patch set in good shape. No committer has reviewed it, and the last message is from 2026-06-29.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-02-24; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Under ANY, which standby the subscriber can safely fail over to changes at runtime

fixable · level 2, rubric item 7 · WAL / replication / standby · `replication-ondisk-1`

A DBA sets synchronized_standby_slots = 'ANY 1 (sync_slot, geo_slot)', where geo_slot serves an async standby. The async standby confirms first, so the subscriber applies WAL that the sync standby has not received. The primary then fails, the HA tool promotes the sync standby as quorum commit expects, and the subscriber is now ahead of the new primary and needs a full resync.

Today every slot listed in synchronized_standby_slots is a safe failover target for logical subscribers; under ANY, only the slots that happen to be caught up are safe. Message #5 raised the mixed sync/async case. The answers in messages #4 and #7 leave it to the failover tool, or call such a configuration invalid, but the proposal neither rejects it nor gives tools a way to see which standby is safe. The docs only point to the manual checks in the logical replication failover steps.

Thread: [#3 (Ashutosh Sharma, 2026-02-26)](https://www.postgresql.org/message-id/CAE9k0Pm_6%2B4zW-X9zgBHhyLa9dqNKLM%3DzzUnVeH%2Bikoh45iALw%40mail.gmail.com), [#4 (Amit Kapila, 2026-02-26)](https://www.postgresql.org/message-id/CAA4eK1%2BCNnJ%3Dp4yTXJKDYKLiJQttjCBdJsoioFVyt%3DJOxH6chA%40mail.gmail.com), [#5 (Ashutosh Sharma, 2026-02-26)](https://www.postgresql.org/message-id/CAE9k0PkW9pAWofkygh8DpA_Yp_9hwbVhWt%3DEgVH4Guwgvqo0KQ%40mail.gmail.com), [#6 (SATYANARAYANA NARLAPURAM, 2026-02-26)](https://www.postgresql.org/message-id/CAHg%2BQDfW8uwhO92sLaWzcyG2st8n%2Bm99sRHwHPnrQo22fA5tyQ%40mail.gmail.com), [#7 (SATYANARAYANA NARLAPURAM, 2026-02-26)](https://www.postgresql.org/message-id/CAHg%2BQDeQ6MLTR7E3RSwwoEXon4z4uo_9P_oum1aGANbi1Go8Qg%40mail.gmail.com)

**Question:** How does a failover tool find, from server state alone, which standby holds all WAL already sent to failover-enabled logical subscribers when ANY is in use?

Counter-check: held. Messages #4 and #7 give the task to the failover tool or call the mixed configuration invalid, but the patch cannot reject it and the docs shown do not warn about it. The setting is opt-in, so a documented rule may be enough.

### 2. Syncrep grammar may reject slot names that are valid today

fixable · level 2, rubric item 12 · design, backpatchability, maintenance burden ("who maintains this in 10 years") · `standards-compat-1`

A cluster has synchronized_standby_slots = 'first, any' or uses a slot such as '2nd_standby' in postgresql.auto.conf. After an upgrade to the new release, the check hook rejects the value at startup. The DBA finds out when the server will not start, or when the setting is refused during a failover.

Today the value is split with SplitIdentifierString and checked with the slot-name rules, so any legal slot name is accepted. The patch routes the value through the synchronous_standby_names scanner, where FIRST and ANY are keywords and, by the usual identifier rules, a name that starts with a digit likely lexes as NUM followed by NAME. The brief lists only the quoting rule for FIRST and ANY and the changed 'invalid*' error (message #94). The proposal does not list which legal slot names stop parsing, and it does not warn users before upgrade.

Thread: [#1 (SATYANARAYANA NARLAPURAM, 2026-02-24)](https://www.postgresql.org/message-id/CAHg%2BQDfU7rOebrLDESPpHSgdiadKbpCOmBokcbmM6Gr%2BA5VobQ%40mail.gmail.com), [#26 (Ashutosh Sharma, 2026-03-18)](https://www.postgresql.org/message-id/CAE9k0P%3Dfo%3DEoJGYDr6eXnEKXpxs_eae0RSX%3D0kyHe6rMumsy1Q%40mail.gmail.com), [#94 (Shlok Kyal, 2026-06-16)](https://www.postgresql.org/message-id/CANhcyEXPYAsLRJOwY6dqtLn82yhyPGv8kCtkc_HcwkbADr_j-Q%40mail.gmail.com)

**Question:** Which slot names accepted by the current check hook, such as keywords or names starting with digits, fail under the syncrep scanner, and how are upgraded configurations warned or kept working?

Counter-check: held. Slot names may start with a digit, and the syncrep scanner reads a leading digit run as NUM, so a name like '2nd_standby' would stop parsing unquoted. The thread covers only FIRST, ANY and 'invalid*', and says nothing about existing configurations at upgrade.

### 3. Shared syncrep grammar now carries two GUCs with different bare-list meanings

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years"), new GUC · `maintenance-fit-1`

A later patch adds a keyword or a new form to synchronous_standby_names in syncrep_gram.y. synchronized_standby_slots silently accepts the new form, or an existing slot named after the keyword stops parsing, and the server fails to load postgresql.conf after an upgrade. A reviewer of the sync rep patch has no reason to look at slot.c.

Patch 0001 adds SYNC_REP_DEFAULT so one grammar can mean FIRST 1 for one GUC and wait-for-all for the other (messages #26, #74). Every consumer of the parse result must now map DEFAULT correctly, and nothing fails loudly if a new consumer does not. The visible effect has started already: 'invalid*' now gets a parser syntax error, not the slot-name error, and the regression output had to change (message #94). Dilip Kumar's request to share check-hook code between the two GUCs is still open (messages #35, #38), so the two hooks can drift.

Thread: [#26 (Ashutosh Sharma, 2026-03-18)](https://www.postgresql.org/message-id/CAE9k0P%3Dfo%3DEoJGYDr6eXnEKXpxs_eae0RSX%3D0kyHe6rMumsy1Q%40mail.gmail.com), [#31 (Ashutosh Sharma, 2026-03-20)](https://www.postgresql.org/message-id/CAE9k0Pm-qakqxs7xGZFisNz3i_3S7C5N%3DwtFzZGxCMvcnB%2BcYg%40mail.gmail.com), [#34 (shveta malik, 2026-03-20)](https://www.postgresql.org/message-id/CAJpy0uCzRdQyde4xMLTp77YZfvO9V8dxJ3jM%3DAOm5awsvp_-QQ%40mail.gmail.com), [#35 (Dilip Kumar, 2026-03-20)](https://www.postgresql.org/message-id/CAFiTN-uf0DE2KphPJPhq3%2B_GbmzNfhTFjsSgpSqHerFdFhx6pw%40mail.gmail.com), [#38 (Ashutosh Sharma, 2026-03-23)](https://www.postgresql.org/message-id/CAE9k0Pn64k8Mhz4j%3DbBaaakM4G9pC9ctcSunFHgqc6m6q7g13Q%40mail.gmail.com), [#74 (Ashutosh Sharma, 2026-05-13)](https://www.postgresql.org/message-id/CAE9k0Pn1Xr37KDCViQLybd-K8rcY073XgQjKL_H4Ai4AE%3DPOjQ%40mail.gmail.com), [#94 (Shlok Kyal, 2026-06-16)](https://www.postgresql.org/message-id/CANhcyEXPYAsLRJOwY6dqtLn82yhyPGv8kCtkc_HcwkbADr_j-Q%40mail.gmail.com)

**Question:** What makes a future change to syncrep_gram.y, or a new consumer of SyncRepConfigData, fail a test when it does not handle synchronized_standby_slots and SYNC_REP_DEFAULT?

Counter-check: held. Reviewers weighed the shared-parser cost in messages #31 to #34 and #75, but no committer has accepted it. Nothing in the thread makes a later grammar change fail a test for synchronized_standby_slots, and the check-hook sharing request remains open.

### 4. No user in the thread asks for FIRST N priority semantics

fixable · level 1b, rubric item 3 · demand, maintenance burden ("who maintains this in 10 years"), new GUC · `demand-1`

A committer takes on patch 0003, which adds a second mode (FIRST N and N (...)) to synchronized_standby_slots, with its own skip and wait rules and its own docs and tests. If no deployment needs priority order for slots, the project maintains and documents that mode for nobody, and its subtle rule of waiting on a valid lagging slot stays untested in the field.

Every demand signal in the thread is about the quorum case: the stall in message #1, WAL growth on the primary in message #2, and customer requests in message #29. Message #64 split the series so that FIRST could be judged on its own merits, but no later message gives a use case for it. Message #71 shows the parser change in 0001 is needed in any case to reject or tell apart explicit FIRST, so a smaller option exists: 0001 plus 0002, with explicit FIRST rejected. Matching the synchronous_standby_names grammar is a reason for consistency, but it does not show that anyone needs the mode.

Thread: [#1 (SATYANARAYANA NARLAPURAM, 2026-02-24)](https://www.postgresql.org/message-id/CAHg%2BQDfU7rOebrLDESPpHSgdiadKbpCOmBokcbmM6Gr%2BA5VobQ%40mail.gmail.com), [#2 (Ashutosh Sharma, 2026-02-25)](https://www.postgresql.org/message-id/CAE9k0PnP0cPuisVeXM%2BBma7n6J%2BHYqhVO5LffosXuHSw7drEDQ%40mail.gmail.com), [#29 (Hou, Zhijie/侯 志杰, 2026-03-19)](https://www.postgresql.org/message-id/TY4PR01MB16907E4E69204C3B318866AF8944FA%40TY4PR01MB16907.jpnprd01.prod.outlook.com), [#64 (Amit Kapila, 2026-04-11)](https://www.postgresql.org/message-id/CAA4eK1JvsBwezSPnnVGC6NL2umkZjTANxvM8rh8OWxFVN_DfPg%40mail.gmail.com), [#71 (shveta malik, 2026-04-15)](https://www.postgresql.org/message-id/CAJpy0uDBxMUznWmqfoDUkg9%3DFDhiGJaQi3mHKHTd3-HZBKmx2A%40mail.gmail.com)

**Question:** Which deployment needs logical decoding to wait on the first N slots in priority order, and what would it do if only ANY N were committed?

Counter-check: held. Every demand statement in the thread concerns the quorum case, and no message after the split in #64 gives a use case for FIRST N. Matching the synchronous_standby_names syntax supports consistency but does not show that anyone needs the mode.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | **yes** | ANY lets failover-enabled logical decoding proceed before every listed physical standby slot confirms WAL. |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | Patch 0001 adds a SYNC_REP_DEFAULT method to the parser result that synchronous_standby_names also uses. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | **yes** | No new GUC, but synchronized_standby_slots gets new syntax and FIRST and ANY become keywords in its value. |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | see objection standards-compat-1: Syncrep grammar may reject slot names that are valid today |
| maintenance burden ("who maintains this in 10 years") | **yes** | The syncrep grammar now serves two GUCs whose bare lists mean different things, with separate check hooks. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. How does a failover tool find, from server state alone, which standby holds all WAL already sent to failover-enabled logical subscribers when ANY is in use?
2. Should the docs warn against, or the check hook reject, an ANY list that mixes slots of synchronous and asynchronous standbys?
3. Which slot names that the current check hook accepts, such as FIRST, ANY or names starting with a digit, fail under the syncrep scanner, and what happens to such configurations at upgrade?
4. What makes a future change to syncrep_gram.y, or a new reader of the parse result, fail a test when it does not handle synchronized_standby_slots and SYNC_REP_DEFAULT?
5. Which deployment needs FIRST N for synchronized_standby_slots, and what would it lose if only ANY N were committed and explicit FIRST were rejected?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. The review notes place Amit Kapila, a committer in this area, in messages #4, #64 and #88, but the brief says no committer has reviewed the patch set. The owner must know slot synchronization and the syncrep parser.
- **Liability:** The owner signs for the rule that a logical subscriber never gets ahead of a promotable standby under ANY, and for the shared syncrep grammar serving two GUCs. Both last as long as the syntax exists.
- **Author follow-up:** The author on the CommitFest entry reviewed from message #2 on, and the set went through many versions until June 2026. Nothing is posted after 2026-06-29.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** The stall affects sites that combine failover slots with quorum commit, and message #29 reports many customer requests. The three-patch set touches slot.c and the shared syncrep grammar, and a reviewer called it in good shape on 2026-06-19.

## Appendix

### Refuted in the counter-check

- `standards-compat-2` Third syncrep_method value changes a struct shared with synchronous_standby_names. Refuted (no failure scenario): The scenario needs a path that skips the conversion, and the objection names none. SyncRepConfig is built only in the synchronous_standby_names check hook, where message #30 places the mapping to PRIORITY with num_sync 1.
- `already-exists-1` Failover tools can already drop a down standby's slot from the list. Refuted (already handled): The thread shows the need is common: message #29 reports requests from many customers, and several independent participants, including a committer in this area, support the idea. The workaround also needs an external tool, and decoding stalls until that tool reacts.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. [P6] added ANY/FIRST to synchronous_standby_names and was returned only for polishing, [P1] died on catalog_xmin safety in slot sync, and [P12] records no reason, so none of their reasons applies here. Messages #4 and #6 point to an earlier thread where the same idea came up; they report only that the approach was similar and that it included a default-to-synchronous_standby_names option, which messages #10 to #12 then dropped.
- standards-compat: 2 objection(s). Other compatibility points look settled in the thread. These include the bare-list default, removing duplicates, the error when N is larger than the number of unique slots, and the updated function docs.
- replication-ondisk: 1 objection(s). The proposal adds no on-disk, WAL, catalog or wire protocol change. Its only persistent effect is the GUC value, and a bare list keeps today's wait-for-all meaning, so the replication concern above is the only one this angle finds.
- security: 0 objection(s). The proposal has no privilege or exposure aspect. synchronized_standby_slots stays a server-level setting that only superusers or ALTER SYSTEM can change, and logical decoding still needs REPLICATION or a slot owner. The new WARNING and DEBUG1 messages show slot names and restart_lsn values, which pg_replication_slots already shows. A single setting bounds the per-check state, so it adds no new denial-of-service path. One point cannot be checked without the code: whether quoted members still pass slot-name validation, and that is a correctness question.
- maintenance-fit: 1 objection(s). Other maintenance points do not hold. Amit Kapila is engaged as a committer in this area (messages #4, #64, #88). Costs fall mainly on users who set the new syntax. No external dependency is added. The patch set is split into separately reviewable steps.
- already-exists: 1 objection(s). No extension or contrib module offers this. The patch reuses the existing syncrep parser, so it does not add a second mechanism.
- demand: 1 objection(s). Demand for the ANY N part is real. It rests on a concrete stall scenario under quorum commit (message #1), reports of WAL growth on the primary (message #2), a Fujitsu reviewer citing many customer requests (message #29), and reviewers from several companies, so the only objection here concerns FIRST N.

### Precedents retrieved

- P1: [Synchronize logical replication slots from primary to standby](https://commitfest.postgresql.org/patch/3393/) (cf, Returned with feedback, 2022-02-18) - used in the review; reason summarised from its thread
- P2: [Synchronizing slots from primary to standby](https://commitfest.postgresql.org/patch/1961/) (cf, Returned with feedback, 2019-09-30) - judged not relevant; reason summarised from its thread
- P3: [Allow synchronous_standby_names to be disabled after communication failure with all synchronous standby servers exceeds some timeout](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Configuration_files) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P4: [An attempt to avoid locally-committed-but-not-replicated-to-standby-transactions in synchronous replication](https://commitfest.postgresql.org/patch/3665/) (cf, Returned with feedback, 2023-08-02) - judged not relevant; reason summarised from its thread
- P5: [Logical decoding on standby](https://commitfest.postgresql.org/patch/788/) (cf, Returned with feedback, 2017-11-24) - judged not relevant
- P6: [Quorum commit for multiple synchronous replication](https://commitfest.postgresql.org/patch/696/) (cf, Returned with feedback, 2016-10-02) - used in the review; reason summarised from its thread
- P7: [Reuse Workers and Replication Slots during Logical Replication](https://commitfest.postgresql.org/patch/3784/) (cf, Returned with feedback, 2024-01-10) - judged not relevant; reason summarised from its thread
- P8: [Minimal logical decoding on standbys](https://commitfest.postgresql.org/patch/2283/) (cf, Withdrawn, 2020-06-11) - judged not relevant; reason summarised from its thread
- P9: [Minimal logical decoding on standbys](https://commitfest.postgresql.org/patch/2683/) (cf, Returned with feedback, 2020-09-17) - judged not relevant; reason summarised from its thread
- P10: [Synchronous replay mode for avoiding stale reads on hot standbys](https://commitfest.postgresql.org/patch/1589/) (cf, Withdrawn, 2019-07-01) - judged not relevant; reason summarised from its thread
- P11: [GUC to disable cancellation of awaiting for synchronous replication](https://commitfest.postgresql.org/patch/2402/) (cf, Returned with feedback, 2021-10-06) - judged not relevant
- P12: [Support multiple synchronous standby servers](https://commitfest.postgresql.org/patch/293/) (cf, Returned with feedback, 2016-02-03) - used in the review
- P13: [Making WAL receiver startup rely on GUC context for primary_conninfo and primary_slot_name](https://commitfest.postgresql.org/patch/1916/) (cf, Rejected, 2019-02-16) - judged not relevant
- P14: [Syncrep and improving latency due to WAL throttling](https://commitfest.postgresql.org/patch/5309/) (cf, Withdrawn, 2025-11-14) - judged not relevant
- P15: [Optimize LISTEN/NOTIFY](https://commitfest.postgresql.org/patch/5912/) (cf, Withdrawn, 2025-08-07) - judged not relevant
- P16: [Allow LISTEN on patterns](https://commitfest.postgresql.org/patch/5630/) (cf, Withdrawn, 2025-03-07) - judged not relevant
- P17: [Pattern based listeners for asynchronous messaging (LISTEN/NOTIFY)](https://commitfest.postgresql.org/patch/1218/) (cf, Returned with feedback, 2017-11-30) - judged not relevant
- P18: [Minimal logical decoding on standbys (take 6)](https://commitfest.postgresql.org/patch/2968/) (cf, Returned with feedback, 2022-02-07) - judged not relevant
- P19: [Fix off-by-one in logical slot resource retention](https://commitfest.postgresql.org/patch/1084/) (cf, Returned with feedback, 2017-10-01) - judged not relevant
- P20: [A little cosmetic to convert_VALUES_to_ANY()](https://commitfest.postgresql.org/patch/5937/) (cf, Withdrawn, 2025-08-04) - judged not relevant
- P21: [Revert "Fix bogus %name-prefix option syntax in all our Bison files."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=71ed8b3ca76cd720f4013c3c20f0d2706583ab9a) (revert, Reverted, 2014-05-28) - judged not relevant
- P22: [Revert "Logical decoding of sequences"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2c7ea57e56ca5f668c32d4266e0a3e45b455bef5) (revert, Reverted, 2022-04-07) - judged not relevant
- P23: [SERIALIZABLE on standby servers](https://commitfest.postgresql.org/patch/1799/) (cf, Withdrawn, 2019-07-01) - judged not relevant
- P24: [logical decoding and replication of sequences, take 2](https://commitfest.postgresql.org/patch/3823/) (cf, Withdrawn, 2024-06-21) - judged not relevant

### Warnings

- synthesis said NEEDS-COMMITTER-EYES but listed a level 1b objection (demand-1); changed to NEEDS-DESIGN-DISCUSSION

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 13; wall time: 175 s; cost reported by the CLI: $6.11
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 8 from cache, 0 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 3 of 24 kept
- Tone check: 77 text field(s) checked, 1 finding(s), 1 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:47:03.337Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

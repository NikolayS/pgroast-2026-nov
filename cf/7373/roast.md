# Design pre-review: synchronized_standby_slots behavior inconsistent with quorum-based synchronous replication

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7373](https://commitfest.postgresql.org/patch/7373/); [thread](https://www.postgresql.org/message-id/flat/CAHg%2BQDfU7rOebrLDESPpHSgdiadKbpCOmBokcbmM6Gr%2BA5VobQ%40mail.gmail.com), 99 of 99 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). No surviving objection is fatal, and demand for ANY N is shown in messages #1, #2 and #29. Three fixable design points remain at level 2: replication-ondisk-1 (which standby is a safe failover target under ANY), standards-compat-1 (legal slot names that the syncrep scanner may reject) and maintenance-fit-1 (one grammar, two bare-list meanings). The change to the shared syncrep grammar needs sign-off from a committer who knows that code. demand-1 affects only FIRST N.

Not checked: Only the first patch's prose was read, and no code. Not checked: whether quoted slot names still pass slot-name validation, which names the scanner rejects, and where SYNC_REP_DEFAULT is mapped for synchronous_standby_names.

## Prior attempts

None found before 2026-02-24.

## Objections a reviewer is likely to raise

### 1. Under ANY, which standby the subscriber can safely fail over to changes at runtime

A DBA sets synchronized_standby_slots = 'ANY 1 (sync_slot, geo_slot)', where geo_slot serves an async standby. The async standby confirms first, so the subscriber applies WAL that the sync standby has not received. The primary then fails, the HA tool promotes the sync standby as quorum commit expects, and the subscriber is now ahead of the new primary and needs a full resync.

Today every slot listed in synchronized_standby_slots is a safe failover target for logical subscribers; under ANY, only the slots that happen to be caught up are safe. Message #5 raised the mixed sync/async case. The answers in messages #4 and #7 leave it to the failover tool, or call such a configuration invalid, but the proposal neither rejects it nor gives tools a way to see which standby is safe. The docs only point to the manual checks in the logical replication failover steps.

**Question:** How does a failover tool find, from server state alone, which standby holds all WAL already sent to failover-enabled logical subscribers when ANY is in use?

<sub>fixable, level 2, WAL / replication / standby; messages [#3](https://www.postgresql.org/message-id/CAE9k0Pm_6%2B4zW-X9zgBHhyLa9dqNKLM%3DzzUnVeH%2Bikoh45iALw%40mail.gmail.com), [#4](https://www.postgresql.org/message-id/CAA4eK1%2BCNnJ%3Dp4yTXJKDYKLiJQttjCBdJsoioFVyt%3DJOxH6chA%40mail.gmail.com), [#5](https://www.postgresql.org/message-id/CAE9k0PkW9pAWofkygh8DpA_Yp_9hwbVhWt%3DEgVH4Guwgvqo0KQ%40mail.gmail.com), [#6](https://www.postgresql.org/message-id/CAHg%2BQDfW8uwhO92sLaWzcyG2st8n%2Bm99sRHwHPnrQo22fA5tyQ%40mail.gmail.com), [#7](https://www.postgresql.org/message-id/CAHg%2BQDeQ6MLTR7E3RSwwoEXon4z4uo_9P_oum1aGANbi1Go8Qg%40mail.gmail.com); `replication-ondisk-1`</sub>

### 2. Syncrep grammar may reject slot names that are valid today

A cluster has synchronized_standby_slots = 'first, any' or uses a slot such as '2nd_standby' in postgresql.auto.conf. After an upgrade to the new release, the check hook rejects the value at startup. The DBA finds out when the server will not start, or when the setting is refused during a failover.

Today the value is split with SplitIdentifierString and checked with the slot-name rules, so any legal slot name is accepted. The patch routes the value through the synchronous_standby_names scanner, where FIRST and ANY are keywords and, by the usual identifier rules, a name that starts with a digit likely lexes as NUM followed by NAME. The brief lists only the quoting rule for FIRST and ANY and the changed 'invalid*' error (message #94). The proposal does not list which legal slot names stop parsing, and it does not warn users before upgrade.

**Question:** Which slot names accepted by the current check hook, such as keywords or names starting with digits, fail under the syncrep scanner, and how are upgraded configurations warned or kept working?

<sub>fixable, level 2, design; messages [#1](https://www.postgresql.org/message-id/CAHg%2BQDfU7rOebrLDESPpHSgdiadKbpCOmBokcbmM6Gr%2BA5VobQ%40mail.gmail.com), [#26](https://www.postgresql.org/message-id/CAE9k0P%3Dfo%3DEoJGYDr6eXnEKXpxs_eae0RSX%3D0kyHe6rMumsy1Q%40mail.gmail.com), [#94](https://www.postgresql.org/message-id/CANhcyEXPYAsLRJOwY6dqtLn82yhyPGv8kCtkc_HcwkbADr_j-Q%40mail.gmail.com); `standards-compat-1`</sub>

### 3. Shared syncrep grammar now carries two GUCs with different bare-list meanings

A later patch adds a keyword or a new form to synchronous_standby_names in syncrep_gram.y. synchronized_standby_slots silently accepts the new form, or an existing slot named after the keyword stops parsing, and the server fails to load postgresql.conf after an upgrade. A reviewer of the sync rep patch has no reason to look at slot.c.

Patch 0001 adds SYNC_REP_DEFAULT so one grammar can mean FIRST 1 for one GUC and wait-for-all for the other (messages #26, #74). Every consumer of the parse result must now map DEFAULT correctly, and nothing fails loudly if a new consumer does not. The visible effect has started already: 'invalid*' now gets a parser syntax error, not the slot-name error, and the regression output had to change (message #94). Dilip Kumar's request to share check-hook code between the two GUCs is still open (messages #35, #38), so the two hooks can drift.

**Question:** What makes a future change to syncrep_gram.y, or a new consumer of SyncRepConfigData, fail a test when it does not handle synchronized_standby_slots and SYNC_REP_DEFAULT?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#26](https://www.postgresql.org/message-id/CAE9k0P%3Dfo%3DEoJGYDr6eXnEKXpxs_eae0RSX%3D0kyHe6rMumsy1Q%40mail.gmail.com), [#31](https://www.postgresql.org/message-id/CAE9k0Pm-qakqxs7xGZFisNz3i_3S7C5N%3DwtFzZGxCMvcnB%2BcYg%40mail.gmail.com), [#34](https://www.postgresql.org/message-id/CAJpy0uCzRdQyde4xMLTp77YZfvO9V8dxJ3jM%3DAOm5awsvp_-QQ%40mail.gmail.com), [#35](https://www.postgresql.org/message-id/CAFiTN-uf0DE2KphPJPhq3%2B_GbmzNfhTFjsSgpSqHerFdFhx6pw%40mail.gmail.com), [#38](https://www.postgresql.org/message-id/CAE9k0Pn64k8Mhz4j%3DbBaaakM4G9pC9ctcSunFHgqc6m6q7g13Q%40mail.gmail.com), [#74](https://www.postgresql.org/message-id/CAE9k0Pn1Xr37KDCViQLybd-K8rcY073XgQjKL_H4Ai4AE%3DPOjQ%40mail.gmail.com), [#94](https://www.postgresql.org/message-id/CANhcyEXPYAsLRJOwY6dqtLn82yhyPGv8kCtkc_HcwkbADr_j-Q%40mail.gmail.com); `maintenance-fit-1`</sub>

### 4. No user in the thread asks for FIRST N priority semantics

A committer takes on patch 0003, which adds a second mode (FIRST N and N (...)) to synchronized_standby_slots, with its own skip and wait rules and its own docs and tests. If no deployment needs priority order for slots, the project maintains and documents that mode for nobody, and its subtle rule of waiting on a valid lagging slot stays untested in the field.

Every demand signal in the thread is about the quorum case: the stall in message #1, WAL growth on the primary in message #2, and customer requests in message #29. Message #64 split the series so that FIRST could be judged on its own merits, but no later message gives a use case for it. Message #71 shows the parser change in 0001 is needed in any case to reject or tell apart explicit FIRST, so a smaller option exists: 0001 plus 0002, with explicit FIRST rejected. Matching the synchronous_standby_names grammar is a reason for consistency, but it does not show that anyone needs the mode.

**Question:** Which deployment needs logical decoding to wait on the first N slots in priority order, and what would it do if only ANY N were committed?

<sub>fixable, level 1b, demand; messages [#1](https://www.postgresql.org/message-id/CAHg%2BQDfU7rOebrLDESPpHSgdiadKbpCOmBokcbmM6Gr%2BA5VobQ%40mail.gmail.com), [#2](https://www.postgresql.org/message-id/CAE9k0PnP0cPuisVeXM%2BBma7n6J%2BHYqhVO5LffosXuHSw7drEDQ%40mail.gmail.com), [#29](https://www.postgresql.org/message-id/TY4PR01MB16907E4E69204C3B318866AF8944FA%40TY4PR01MB16907.jpnprd01.prod.outlook.com), [#64](https://www.postgresql.org/message-id/CAA4eK1JvsBwezSPnnVGC6NL2umkZjTANxvM8rh8OWxFVN_DfPg%40mail.gmail.com), [#71](https://www.postgresql.org/message-id/CAJpy0uDBxMUznWmqfoDUkg9%3DFDhiGJaQi3mHKHTd3-HZBKmx2A%40mail.gmail.com); `demand-1`</sub>

## Questions for the author

1. How does a failover tool find, from server state alone, which standby holds all WAL already sent to failover-enabled logical subscribers when ANY is in use?
2. Should the docs warn against, or the check hook reject, an ANY list that mixes slots of synchronous and asynchronous standbys?
3. Which slot names that the current check hook accepts, such as FIRST, ANY or names starting with a digit, fail under the syncrep scanner, and what happens to such configurations at upgrade?
4. What makes a future change to syncrep_gram.y, or a new reader of the parse result, fail a test when it does not handle synchronized_standby_slots and SYNC_REP_DEFAULT?
5. Which deployment needs FIRST N for synchronized_standby_slots, and what would it lose if only ANY N were committed and explicit FIRST were rejected?

## Touch points

- **WAL / replication / standby:** ANY lets failover-enabled logical decoding proceed before every listed physical standby slot confirms WAL.
- **ABI / extension API:** Patch 0001 adds a SYNC_REP_DEFAULT method to the parser result that synchronous_standby_names also uses.
- **new GUC:** No new GUC, but synchronized_standby_slots gets new syntax and FIRST and ANY become keywords in its value.
- **backpatchability:** see objection standards-compat-1: Syncrep grammar may reject slot names that are valid today
- **maintenance burden ("who maintains this in 10 years"):** The syncrep grammar now serves two GUCs whose bare lists mean different things, with separate check hooks.

## Who would commit it, and when

- Owner: qualified committer engaged. The review notes place Amit Kapila, a committer in this area, in messages #4, #64 and #88, but the brief says no committer has reviewed the patch set. The owner must know slot synchronization and the syncrep parser.
- Priority: proceed; audience a sizeable minority; review cost medium. The stall affects sites that combine failover slots with quorum commit, and message #29 reports many customer requests. The three-patch set touches slot.c and the shared syncrep grammar, and a reviewer called it in good shape on 2026-06-19.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 2 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

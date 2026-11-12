# Design pre-review: pg_dump: Fetch sequence data only for sequences being dumped

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7350](https://commitfest.postgresql.org/patch/7350/); [thread](https://www.postgresql.org/message-id/flat/19688-e90025dc375a22a3%40postgresql.org), 2 of 2 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). The problem is stated with numbers and the demand is a field report of a v18 regression, so levels 1a and 1b pass. One fixable design objection survived, prior-attempts-1: the thread does not show that the OID list covers every sequence whose value pg_dump emits under binary upgrade, extension configuration or --data-only with filters. That is a coverage question for someone who knows pg_dump, and it does not put the idea in doubt.

Not checked: The code was not read. How the patch builds the OID list, and whether it covers binary-upgrade, extension member and identity sequences, could not be checked.

## Prior attempts

- [Ignore attributes of tables part of extensions in pg_dump](https://www.postgresql.org/message-id/flat/CAB7nPqSO28tQMyvCYHVuDO_jYhzBbAE17whhpPJ0-TyCbYky6w@mail.gmail.com) (related; Rejected, 2015-02-23). "Hm. I have been pondering about this code more and I am dropping the patch as either adding a check based on the flag to track dumpable objects or ext_member visibly breaks the logic that binary upgrade and tables in extensions rely on." ([message](https://www.postgresql.org/message-id/CAB7nPqTyYdQeg%2BaR3sC1y45AfXZg8FiXsffz0P3c4KqkBxj_5Q%40mail.gmail.com)) Different now: P7 skipped column fetching for extension member tables. This patch limits pg_get_sequence_data() calls to sequences whose data is dumped and keeps fetching definitions for all sequences.

## Objections a reviewer is likely to raise

### 1. Data-OID filter may miss sequences dumped through binary upgrade or extensions

A user runs pg_upgrade, or dumps an extension whose configuration sequence is marked with pg_extension_config_dump. If the OID list misses a sequence whose value pg_dump still emits, the restore writes a wrong or default last_value. The next inserts then fail with duplicate keys, possibly weeks after the upgrade.

[P7] was dropped because filtering a catalog fetch on pg_dump's dump flags "visibly breaks the logic that binary upgrade and tables in extensions rely on". This patch also filters a bulk fetch, here by which sequences have data dumped. Message #2 and the commit message do not cover binary upgrade, extension member sequences, or --data-only with filters. The reporter's tests in message #1 cover one --schema dump.

Precedent: [Ignore attributes of tables part of extensions in pg_dump](https://www.postgresql.org/message-id/flat/CAB7nPqSO28tQMyvCYHVuDO_jYhzBbAE17whhpPJ0-TyCbYky6w@mail.gmail.com) (Rejected, 2015-02-23)

**Question:** For binary-upgrade dumps, extension configuration sequences and --data-only with filters, is every sequence whose value pg_dump emits in the OID list, and which test shows it?

<sub>fixable, level 2, pg_dump / restore round-trip; messages [#1](https://www.postgresql.org/message-id/19688-e90025dc375a22a3%40postgresql.org), [#2](https://www.postgresql.org/message-id/CA%2Bnn4-qhKbHqXKtWc7asZAuHZE%2B6urDBxebxF1uCLz767khSAw%40mail.gmail.com); `prior-attempts-1`</sub>

## Questions for the author

1. For binary-upgrade dumps, extension configuration sequences and --data-only with filters, is every sequence whose value pg_dump emits in the OID list, and which test shows it?
2. Does a full dump keep the existing bulk query, or does it also pass an OID list?
3. How does the query behave when a partial dump still includes more than 100000 sequences, for example a dump that excludes one small schema?
4. Does the patch add a test for a selective dump and for the lock wait on a sequence outside the dump?
5. Do the REL_18_STABLE and REL_19_STABLE versions produce the same output as the unpatched branches for a full dump and for pg_upgrade?

## Touch points

- **on-disk format / pg_upgrade:** pg_upgrade runs pg_dump --binary-upgrade, which uses the changed collectSequences() query.
- **pg_dump / restore round-trip:** The patch changes which sequences get last_value and is_called fetched for the dump.
- **backpatchability:** The author asks for a backpatch through 18 and supplies versions for REL_18_STABLE and REL_19_STABLE.
- **performance on the hot path:** The patch changes the sequence query that every pg_dump run against a v18 or later server executes.

## Who would commit it, and when

- Owner: silent. A committer who knows pg_dump's object selection, binary-upgrade mode and the sequence data collection added by commit bd15b7db48.
- Priority: proceed; audience a sizeable minority; review cost low. Message #1 reports a schema dump going from about 50 seconds to about 5.5 minutes on a schema-per-tenant database. The patch touches one function in pg_dump.c and adds no user-visible surface.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

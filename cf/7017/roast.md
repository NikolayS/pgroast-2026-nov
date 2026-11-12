# Design pre-review: Soft DROP TABLE, recoverable drops for PostgreSQL

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7017](https://commitfest.postgresql.org/patch/7017/); [thread](https://www.postgresql.org/message-id/flat/CAOVWO5o3TQHdEb_%2Bagn0UFHmZDN19q5cxExX3Dc_5W%2Bb0Bx_mQ%40mail.gmail.com), 11 of 11 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The review stops at level 1b on already-exists-1. The patch touches only contrib and doc wiring and builds with PGXS, so the same code works as an outside extension. Message #6 asked why it must be in contrib and has no direct answer. The goal of recoverable drops is not in dispute, and the level 2 objections that survived were not evaluated.

Not checked: The code was not read, and the function reference in pgtrashcan.sgml is truncated. Lock handling during the move, search_path in the SECURITY DEFINER functions, and any per-entry privilege checks could not be judged.

## Prior attempts

- [Log LSN before table is dropped](https://www.postgresql.org/message-id/flat/CACLU5mSr+t+6K_Ww6z1fk_=jP=5-U0CEqMsHjii8uG_6UWQo-g@mail.gmail.com) (related; Returned with feedback, 2025-07-25). "What stops us from logging all the same inside object access hook defined by extension?" ([message](https://www.postgresql.org/message-id/CALdSSPiRDvbuPgZKAB1%2BBDb3Nfe-i3PYkiy%3DScMZ7tU5f0toKQ%40mail.gmail.com)) Different now: P1 logged the LSN at DROP TABLE so the table could come back through point-in-time recovery. This proposal keeps the table in a reserved schema, so no point-in-time recovery is needed.

## Objections a reviewer is likely to raise

### 1. Contrib placement adds nothing the external pgtrashcan cannot do

A committer commits contrib/pgtrashcan, a 343 KiB module that uses only the existing ProcessUtility_hook. From then on the project owns its behaviour across every DDL change, release after release. The same code could have shipped as an update to the external pgtrashcan, with no extra cost to core.

The patch touches only contrib and doc wiring, so every part of it already works as an external extension built with PGXS, as the README shows. Andreas asked in message #6 why it cannot update or fork the existing extension outside core, and that question has no answer. Amit in message #7 gives one reason for in-tree placement, coordinated review with core, but the patch does not handle pg_dump, publications or the other coupled areas he lists. A similar question about using an extension hook was raised for [P1].

Precedent: [Log LSN before table is dropped](https://www.postgresql.org/message-id/flat/CACLU5mSr+t+6K_Ww6z1fk_=jP=5-U0CEqMsHjii8uG_6UWQo-g@mail.gmail.com) (Returned with feedback, 2025-07-25)

**Question:** What does contrib placement give this module that a maintained external fork of pgtrashcan, built with PGXS on the same hook, cannot give?

<sub>fixable, level 1b, doesn't fit what Postgres is about (extension / external tool / already solved); messages [#4](https://www.postgresql.org/message-id/CAN4CZFPW60AvbCTUhY6XPNJbcB2q1-1Zh6dhEpLqXh1fJuvYTg%40mail.gmail.com), [#6](https://www.postgresql.org/message-id/ffdee390-f977-4e75-ba86-36348ea6341d%40proxel.se), [#7](https://www.postgresql.org/message-id/CAA4eK1JrsMd-1H6tZ6%3Dj_NO%3DVqoj1s%3DoFb46Zd2VAqCXLKo33g%40mail.gmail.com), [#10](https://www.postgresql.org/message-id/CAOVWO5qmGUP_nw6naGL_rjnCmeVLaFkueNNwfQ39eZKF%2B4QsnQ%40mail.gmail.com); `already-exists-1`</sub>

## Questions for the author

1. What does contrib placement give this module that a maintained outside fork of pgtrashcan, built with PGXS on the same hook, cannot give?
2. Which core hook, existing or new, would let one interception point cover drop table, drop schema cascade, drop owned and drop extension, and could it be committed first?
3. What bounds the disk space and time held by trashed relations when a rotation job drops tables without setting pgtrashcan.enabled to off?
4. How are pgtrashcan._trash_catalog and the trashed tables kept out of FOR ALL TABLES publications, and what does pg_dump emit for them?
5. Which role may restore or purge a given trash entry, and what keeps a restored table closed until its RLS policies are rebuilt?

## Touch points

- **on-disk format / pg_upgrade:** Trashed relation files stay on disk until a manual purge, and pg_upgrade of the reserved schema is not handled.
- **WAL / replication / standby:** Soft-dropped tables stay in publications, and pgtrashcan._trash_catalog is an ordinary table that FOR ALL TABLES publishes.
- **catalog changes:** The extension adds the reserved pgtrashcan schema and the protected table pgtrashcan._trash_catalog, keyed by OID-derived names.
- **ABI / extension API:** The module depends on ProcessUtility_hook, and message #11 asks for a hook in the deletion code instead.
- **SQL-standard conflict or non-standard syntax:** DROP TABLE becomes a move to another schema, and CASCADE also moves FK-referencing tables.
- **new GUC:** The patch adds pgtrashcan.enabled (USERSET, default on) and pgtrashcan.internal_operation (SUSET).
- **pg_dump / restore round-trip:** pg_dump emits trashed tables as normal tables, and message #10 confirms the patch does not handle this.
- **security / privileges:** Restore and purge are SECURITY DEFINER functions, and trashed tables keep their grants while their RLS policies are dropped.
- **maintenance burden ("who maintains this in 10 years"):** A 343.6 KB contrib module must follow every change to drop paths, dependent object types, publications and pg_dump.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** The module is a fork of an outside extension and uses only an existing hook, so it can stay outside core.

## Who would commit it, and when

- Owner: silent. The owner would need to know the dependency and deletion code (performDeletion), and also logical replication and pg_dump. Reviewers in the thread asked design questions, but none has offered to take the patch.
- Not now, show demand first: A concrete need that only in-tree placement meets would change the answer, for example a new core deletion hook that the module depends on and tests. Reports from users of the outside pgtrashcan who cannot get this version outside core would also count. The level 2 design points still need answers after that.
- Priority: needs design rethink; audience a sizeable minority; review cost high. Accidental drops are a real operator problem, and message #1 cites similar features in Oracle, Db2 and Snowflake. The 343.6 KB patch misses DROP SCHEMA ... CASCADE and DROP OWNED, and the thread asks for a hook at the right level and a reason for contrib before review continues.

---

[roast-full.md](roast-full.md) has 8 more objection(s) that held, 8 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

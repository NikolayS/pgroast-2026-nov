# Design pre-review: UPDATE/DELETE FOR PORTION OF

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7571](https://commitfest.postgresql.org/patch/7571/); [thread](https://www.postgresql.org/message-id/flat/ec498c3d-5f2b-48ec-b989-5561c8aa2024%40illuminatedcomputing.com), 68 of 68 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No surviving objection questions the goal: the remaining patches complete SQL:2011 features on top of the FOR PORTION OF core committed for v19. Three fixable design points remain open. standards-compat-1 asks how a PERIOD's hidden column behaves in pg_dump and psql. maintenance-fit-2 asks for one updated-columns set that includes the range column, and maintenance-fit-1 asks which `ForPortionOfState` fields triggers may rely on. Nobody reviewed these follow-on patches in the visible thread.

Not checked: The second mail thread on the CommitFest entry and the code were not read, so the exact v20 patch set is unknown. The foreign key actions' concurrency behaviour and the PERIOD implementation could not be judged.

## Prior attempts

- [SQL:2011 application time](https://www.postgresql.org/message-id/flat/CA+renyUApHgSZF9-nd-a0+OPGharLQLO=mDHcY4_qQ0+noCUVg@mail.gmail.com) (similar idea; Returned with feedback, 2022-02-07). "IMO, if this temporal feature is to happen, btree_gist needs to be moved into core first. Having to install an extension in order to use an in-core feature like this isn't going to be an acceptable experience." ([message](https://www.postgresql.org/message-id/f42e5be5-cd3e-bc4f-83e4-04a1e0270b17%40enterprisedb.com)) Different now: Temporal keys shipped in v18, so the btree_gist objection no longer blocks this work. FOR PORTION OF now inserts leftovers through ExecInsert without SPI and was committed for v19. Only the trigger, foreign key action, PL/pgSQL and PERIOD patches remain open.

## Objections a reviewer is likely to raise

### 1. PERIOD DDL adds hidden columns with no dump, psql or catalog story

A user runs alter table add period and later dumps the database. If pg_dump emits the hidden generated range column as a normal column, the restore creates both the period and a duplicate column. If pg_dump omits the column, a restore into v19 fails on the PERIOD syntax with no fallback.

Message #4 says PERIOD support rests on hidden GENERATED range columns. Postgres has no hidden user columns today, so select *, \d, information_schema.columns, column renames and pg_dump all need a defined rule. The visible thread does not discuss any of these, and nobody reviewed the PERIOD patch. Message #26 drops the patch from the set because it often conflicts, so its design was never checked against these paths.

**Question:** How do pg_dump, psql \d, information_schema and ALTER TABLE RENAME, DROP and SET DATA TYPE treat the hidden range column behind a PERIOD?

<sub>fixable, level 2, pg_dump / restore round-trip; messages [#3](https://www.postgresql.org/message-id/CA%2BrenyXXJJCmgG0Wdf89JgVNAeRKkVn%2BEuZGTf4Ph-BWoJafQA%40mail.gmail.com), [#4](https://www.postgresql.org/message-id/CA%2BrenyWA-b00qvz4gDdPbPBjVatacB1T5v7SJc6J_xg3R6-qRw%40mail.gmail.com), [#26](https://www.postgresql.org/message-id/CA%2BrenyVfLaa1%3Dr1OB9UdCxCRYUmQxAyLH%2BkVi1qX3ookT%3DeYEw%40mail.gmail.com); `standards-compat-1`</sub>

### 2. The FOR PORTION OF column is not tracked as an updated column

A later patch skips CHECK constraints, generated columns or UPDATE OF triggers based on ExecGetAllUpdatedCols. It never sees the range column that FOR PORTION OF narrows, so a check on valid_at is skipped. Nothing fails until a user reports bad data.

Message #45 showed that ExecGetAllUpdatedCols omits the range column. In message #53 the author proposed a separate bitmapset as the robust fix and a one-line union as a short-term patch, and the thread shows no final choice. The CASCADE and SET NULL patches and the trigger patches add more code that depends on knowing which columns change. Each new user of updated-column sets must remember this special case, and no check fails if a patch forgets it.

**Question:** Will the range column be added to one updated-columns set that all consumers use, before the trigger and foreign key patches build on it?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#45](https://www.postgresql.org/message-id/CACJufxHALFKca5SMn5DNnbrX2trPamVL6napn_nm35p15yw%2Brg%40mail.gmail.com), [#53](https://www.postgresql.org/message-id/CA%2BrenyV6_xRwrDGBiNj79oCPfMqOoAqDQg7b%3DEerEpq0HHmb4Q%40mail.gmail.com); `maintenance-fit-2`</sub>

### 3. ForPortionOfState exposed in TriggerData becomes a fixed extension API

An extension author writes a C audit trigger that reads tg_temporal->fp_targetRange and fp_leftoverstypcache. A later release reshapes ForPortionOfState for PERIODs, user-defined types or multi-dimensional ranges. The extension breaks at the next major upgrade, or misreads the struct if it is not rebuilt.

The patch documents an executor-internal struct in trigger.sgml, including a TypeCacheEntry pointer and a column name string, as the C trigger interface. The thread already plans changes that touch this state: PERIOD support, user-defined leftover functions (message #31), and links from leftovers to the old row (message #21). No message discusses which fields are a stable contract. Once trigger.sgml documents a struct, changing it costs every C trigger that reads it.

**Question:** Which minimal fields do triggers need, and why not expose those in a small dedicated struct rather than the executor's ForPortionOfState?

<sub>fixable, level 2, ABI / extension API; messages [#1](https://www.postgresql.org/message-id/ec498c3d-5f2b-48ec-b989-5561c8aa2024%40illuminatedcomputing.com), [#21](https://www.postgresql.org/message-id/CA%2BrenyXchHgpMbYX-cR8fuNnnpf_%2BFJ6PkHoXoa2AgzRnz4vxQ%40mail.gmail.com), [#31](https://www.postgresql.org/message-id/CA%2BrenyWKOj5%3DrMmQmJcbybu-Vdomxdp%3DeJ93kp76AgmQKYdfiQ%40mail.gmail.com); `maintenance-fit-1`</sub>

## Questions for the author

1. How do pg_dump, pg_upgrade, psql `\d`, information_schema and ALTER TABLE RENAME or DROP COLUMN treat the hidden range column behind a PERIOD?
2. Will the range column be added to one updated-columns set that all consumers use, before the trigger and foreign key patches build on it?
3. Which fields of `ForPortionOfState` may C triggers rely on, and why not expose only those in a small dedicated struct?
4. How do temporal CASCADE, SET NULL and SET DEFAULT behave under READ COMMITTED when a concurrent session changes the referenced row?
5. Which patches make up the v20 set in this CommitFest entry, and can each be committed on its own?

## Touch points

- **on-disk format / pg_upgrade:** PERIOD DDL rests on a hidden generated range column that pg_upgrade must carry over.
- **WAL / replication / standby:** Publications publish the UPDATE or DELETE and then one INSERT per temporal leftover.
- **catalog changes:** The patches add pg_proc entries, touch pg_constraint helper lookups, and PERIOD DDL needs catalog support.
- **ABI / extension API:** `TriggerData` gains `tg_temporal`, which points to the executor's `ForPortionOfState`.
- **SQL-standard conflict or non-standard syntax:** The patches add FOR PORTION OF, temporal foreign key actions and PERIOD DDL from SQL:2011, with range columns as an extension.
- **planner behaviour change:** The patch touches createplan.c, planner.c and pathnode.c to carry FOR PORTION OF to ModifyTable.
- **pg_dump / restore round-trip:** pg_dump must dump and restore a PERIOD and its hidden range column.
- **psql UX conventions:** psql `\d` needs a rule for showing a PERIOD and its hidden column.
- **security / privileges:** Leftover inserts need no INSERT privilege but still fire INSERT triggers.
- **maintenance burden ("who maintains this in 10 years"):** The PERIOD patch conflicts each time cataloged NOT NULL constraints change, and the range column is a special case for updated-column sets.
- **performance on the hot path:** nodeModifyTable.c gains FOR PORTION OF handling in the UPDATE and DELETE path.

## Who would commit it, and when

- Owner: qualified committer engaged. Peter Eisentraut committed the core FOR PORTION OF patches for v19. The follow-on patches need someone who knows the RI trigger code and ALTER TABLE, and nobody reviewed them in the visible thread.
- Priority: proceed; audience niche; review cost high. The patches complete a standard feature whose core is already committed, for users of temporal tables. They touch RI triggers, PL/pgSQL, ALTER TABLE and pg_dump, so review is costly, and the entry needs a rebase first.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 6 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 3.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

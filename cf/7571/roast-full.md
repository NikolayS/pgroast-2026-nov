# Design pre-review: UPDATE/DELETE FOR PORTION OF

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7571](https://commitfest.postgresql.org/patch/7571/) (Needs review); authors: Paul Jungwirth (pjungwir)
- Thread: [SQL:2011 Application Time Update & Delete](https://www.postgresql.org/message-id/flat/ec498c3d-5f2b-48ec-b989-5561c8aa2024%40illuminatedcomputing.com), started 2025-06-02
- Messages read: 68 of 68
- Patch prose read: v52-0004-Document-temporal-update-delete.patch, v52-0002-Document-temporal-foreign-keys.patch, v52-0005-Add-range_minus_multi-and-multirange_minus_multi.patch, v52-0001-Add-docs-chapter-for-temporal-tables.patch, v52-0003-Document-temporal-PERIODs.patch, v52-0007-Add-tg_temporal-to-TriggerData.patch, v52-0008-Look-up-more-temporal-foreign-key-helper-procs.patch, v52-0006-Add-UPDATE-DELETE-FOR-PORTION-OF.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No surviving objection questions the goal: the remaining patches complete SQL:2011 features on top of the FOR PORTION OF core committed for v19. Three fixable design points remain open. standards-compat-1 asks how a PERIOD's hidden column behaves in pg_dump and psql. maintenance-fit-2 asks for one updated-columns set that includes the range column, and maintenance-fit-1 asks which `ForPortionOfState` fields triggers may rely on. Nobody reviewed these follow-on patches in the visible thread.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The second mail thread on the CommitFest entry and the code were not read, so the exact v20 patch set is unknown. The foreign key actions' concurrency behaviour and the PERIOD implementation could not be judged.

## The proposal, restated

**Problem.** Users of temporal primary and foreign keys, released in v18, have no way to change only part of a row's application time. The author says temporal keys are hard to use without temporal DML. Temporal foreign keys also support only NO ACTION.

**Proposed design.** `update` and `delete` accept `FOR PORTION OF col FROM x TO y` or `FOR PORTION OF col (expr)`. The executor narrows the matched row to the target and inserts up to two "temporal leftovers" for the untouched time, computed by new set-returning functions. Later patches expose the bounds to triggers and use them for temporal foreign key CASCADE, SET NULL, SET DEFAULT and PERIOD DDL.

**User-visible surface.**
- SQL syntax: `UPDATE/DELETE ... FOR PORTION OF range_column_name { FROM start TO end | (expression) }`; NULL bounds mean unbounded.
- Functions `range_minus_multi(anyrange, anyrange)` and `multirange_minus_multi(anymultirange, anymultirange)`, both returning setof.
- Leftover inserts fire INSERT row and statement triggers nested inside the UPDATE/DELETE triggers, and need no INSERT privilege.
- C API: new `tg_temporal` field in `TriggerData`, pointing to `ForPortionOfState`, plus FOR PORTION OF bounds in PL/pgSQL trigger variables.
- Temporal foreign keys gain ON UPDATE/DELETE CASCADE, SET NULL and SET DEFAULT, and a later patch adds PERIOD DDL.

**What the author claims.**
- Trigger firing order follows SQL:2011 15.14 and paper SQL-026R2, and matches DB2 and MariaDB.
- A set-returning function for leftovers allows future user-defined types, such as spatial types or multi-dimensional ranges.
- READ COMMITTED gives lost updates under concurrency, as MariaDB does, and SELECT FOR UPDATE beforehand avoids them.
- RETURNING should omit leftovers, because they represent unchanged data.
- Leftover inserts need no INSERT privilege, because they add no new information.

**Thread state.** Peter Eisentraut committed the core FOR PORTION OF patches in April 2026 for v19, and follow-up bugs (INSTEAD OF views, SRF protocol, BEFORE triggers changing the range) were fixed or closed by July 2026. Nobody reviewed the trigger, foreign key, PL/pgSQL and PERIOD patches in the visible thread.

## 2. Prior attempts

### SQL:2011 application time (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/2048/](https://commitfest.postgresql.org/patch/2048/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CA+renyUApHgSZF9-nd-a0+OPGharLQLO=mDHcY4_qQ0+noCUVg@mail.gmail.com](https://www.postgresql.org/message-id/flat/CA+renyUApHgSZF9-nd-a0+OPGharLQLO=mDHcY4_qQ0+noCUVg@mail.gmail.com)
- **Date:** decided 2022-02-07; started 2021-06-30
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Peter Eisentraut (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Peter Eisentraut held that btree_gist must move into core first, because the temporal keys need it for most uses. He also objected that the index owns the constraint instead of the reverse, and asked for FOR PORTION OF without triggers. In January 2022 the patch set no longer applied and reviews had unanswered comments, so the entry was set to Waiting on Author.
- **Reason (quote):** "IMO, if this temporal feature is to happen, btree_gist needs to be moved into core first. Having to install an extension in order to use an in-core feature like this isn't going to be an acceptable experience." ([message](https://www.postgresql.org/message-id/f42e5be5-cd3e-bc4f-83e4-04a1e0270b17%40enterprisedb.com))
- **What is different this time:** Temporal keys shipped in v18, so the btree_gist objection no longer blocks this work. FOR PORTION OF now inserts leftovers through ExecInsert without SPI and was committed for v19. Only the trigger, foreign key action, PL/pgSQL and PERIOD patches remain open.


## 3. Objections a reviewer is likely to raise

### 1. PERIOD DDL adds hidden columns with no dump, psql or catalog story

fixable · level 2, rubric item 13 · pg_dump / restore round-trip, psql UX conventions, catalog changes, SQL-standard conflict or non-standard syntax · `standards-compat-1`

A user runs alter table add period and later dumps the database. If pg_dump emits the hidden generated range column as a normal column, the restore creates both the period and a duplicate column. If pg_dump omits the column, a restore into v19 fails on the PERIOD syntax with no fallback.

Message #4 says PERIOD support rests on hidden GENERATED range columns. Postgres has no hidden user columns today, so select *, \d, information_schema.columns, column renames and pg_dump all need a defined rule. The visible thread does not discuss any of these, and nobody reviewed the PERIOD patch. Message #26 drops the patch from the set because it often conflicts, so its design was never checked against these paths.

Thread: [#3 (Paul A Jungwirth, 2025-08-29)](https://www.postgresql.org/message-id/CA%2BrenyXXJJCmgG0Wdf89JgVNAeRKkVn%2BEuZGTf4Ph-BWoJafQA%40mail.gmail.com), [#4 (Paul A Jungwirth, 2025-09-25)](https://www.postgresql.org/message-id/CA%2BrenyWA-b00qvz4gDdPbPBjVatacB1T5v7SJc6J_xg3R6-qRw%40mail.gmail.com), [#26 (Paul A Jungwirth, 2025-12-18)](https://www.postgresql.org/message-id/CA%2BrenyVfLaa1%3Dr1OB9UdCxCRYUmQxAyLH%2BkVi1qX3ookT%3DeYEw%40mail.gmail.com)

**Question:** How do pg_dump, psql \d, information_schema and ALTER TABLE RENAME, DROP and SET DATA TYPE treat the hidden range column behind a PERIOD?

Counter-check: held. Message #4 bases PERIODs on hidden generated range columns, and the visible thread has no review of that patch. Nothing shown says how pg_dump, psql or information_schema treat the hidden column.

### 2. The FOR PORTION OF column is not tracked as an updated column

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years"), catalog changes · `maintenance-fit-2`

A later patch skips CHECK constraints, generated columns or UPDATE OF triggers based on ExecGetAllUpdatedCols. It never sees the range column that FOR PORTION OF narrows, so a check on valid_at is skipped. Nothing fails until a user reports bad data.

Message #45 showed that ExecGetAllUpdatedCols omits the range column. In message #53 the author proposed a separate bitmapset as the robust fix and a one-line union as a short-term patch, and the thread shows no final choice. The CASCADE and SET NULL patches and the trigger patches add more code that depends on knowing which columns change. Each new user of updated-column sets must remember this special case, and no check fails if a patch forgets it.

Thread: [#45 (jian he, 2026-04-07)](https://www.postgresql.org/message-id/CACJufxHALFKca5SMn5DNnbrX2trPamVL6napn_nm35p15yw%2Brg%40mail.gmail.com), [#53 (Paul A Jungwirth, 2026-04-15)](https://www.postgresql.org/message-id/CA%2BrenyV6_xRwrDGBiNj79oCPfMqOoAqDQg7b%3DEerEpq0HHmb4Q%40mail.gmail.com)

**Question:** Will the range column be added to one updated-columns set that all consumers use, before the trigger and foreign key patches build on it?

Counter-check: held. Message #45 shows ExecGetAllUpdatedCols omits the range column, and message #53 offers two fixes without the thread recording a choice. The foreign key and trigger patches build on this code, so the question applies to them.

### 3. ForPortionOfState exposed in TriggerData becomes a fixed extension API

fixable · level 2, rubric item 11 · ABI / extension API, maintenance burden ("who maintains this in 10 years") · `maintenance-fit-1`

An extension author writes a C audit trigger that reads tg_temporal->fp_targetRange and fp_leftoverstypcache. A later release reshapes ForPortionOfState for PERIODs, user-defined types or multi-dimensional ranges. The extension breaks at the next major upgrade, or misreads the struct if it is not rebuilt.

The patch documents an executor-internal struct in trigger.sgml, including a TypeCacheEntry pointer and a column name string, as the C trigger interface. The thread already plans changes that touch this state: PERIOD support, user-defined leftover functions (message #31), and links from leftovers to the old row (message #21). No message discusses which fields are a stable contract. Once trigger.sgml documents a struct, changing it costs every C trigger that reads it.

Thread: [#1 (Paul Jungwirth, 2025-06-02)](https://www.postgresql.org/message-id/ec498c3d-5f2b-48ec-b989-5561c8aa2024%40illuminatedcomputing.com), [#21 (Paul A Jungwirth, 2025-11-19)](https://www.postgresql.org/message-id/CA%2BrenyXchHgpMbYX-cR8fuNnnpf_%2BFJ6PkHoXoa2AgzRnz4vxQ%40mail.gmail.com), [#31 (Paul A Jungwirth, 2026-01-19)](https://www.postgresql.org/message-id/CA%2BrenyWKOj5%3DrMmQmJcbybu-Vdomxdp%3DeJ93kp76AgmQKYdfiQ%40mail.gmail.com)

**Question:** Which minimal fields do triggers need, and why not expose those in a small dedicated struct rather than the executor's ForPortionOfState?

Counter-check: held. TriggerData already exposes internal structs, and C extensions are rebuilt for each major version, so breakage on upgrade is normal. But trigger.sgml would document internal fields such as fp_leftoverstypcache, and the thread does not say which fields triggers may rely on.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | PERIOD DDL rests on a hidden generated range column that pg_upgrade must carry over. |
| WAL / replication / standby | **yes** | Publications publish the UPDATE or DELETE and then one INSERT per temporal leftover. |
| wire protocol | no |  |
| catalog changes | **yes** | The patches add pg_proc entries, touch pg_constraint helper lookups, and PERIOD DDL needs catalog support. |
| ABI / extension API | **yes** | `TriggerData` gains `tg_temporal`, which points to the executor's `ForPortionOfState`. |
| SQL-standard conflict or non-standard syntax | **yes** | The patches add FOR PORTION OF, temporal foreign key actions and PERIOD DDL from SQL:2011, with range columns as an extension. |
| new GUC | no |  |
| planner behaviour change | **yes** | The patch touches createplan.c, planner.c and pathnode.c to carry FOR PORTION OF to ModifyTable. |
| pg_dump / restore round-trip | **yes** | pg_dump must dump and restore a PERIOD and its hidden range column. |
| psql UX conventions | **yes** | psql `\d` needs a rule for showing a PERIOD and its hidden column. |
| security / privileges | **yes** | Leftover inserts need no INSERT privilege but still fire INSERT triggers. |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | The PERIOD patch conflicts each time cataloged NOT NULL constraints change, and the range column is a special case for updated-column sets. |
| performance on the hot path | **yes** | nodeModifyTable.c gains FOR PORTION OF handling in the UPDATE and DELETE path. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. How do pg_dump, pg_upgrade, psql `\d`, information_schema and ALTER TABLE RENAME or DROP COLUMN treat the hidden range column behind a PERIOD?
2. Will the range column be added to one updated-columns set that all consumers use, before the trigger and foreign key patches build on it?
3. Which fields of `ForPortionOfState` may C triggers rely on, and why not expose only those in a small dedicated struct?
4. How do temporal CASCADE, SET NULL and SET DEFAULT behave under READ COMMITTED when a concurrent session changes the referenced row?
5. Which patches make up the v20 set in this CommitFest entry, and can each be committed on its own?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Peter Eisentraut committed the core FOR PORTION OF patches for v19. The follow-on patches need someone who knows the RI trigger code and ALTER TABLE, and nobody reviewed them in the visible thread.
- **Liability:** The owner signs for a documented C trigger struct, foreign key actions that depend on FOR PORTION OF bounds, and a hidden-column mechanism for PERIODs across ALTER TABLE, pg_dump and psql, with no end date.
- **Author follow-up:** Strong: the author kept the series going past 50 versions, keeps a wiki page of design decisions, and took part in follow-up bug fixes through July 2026.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** high
- **Review cost:** high
- **Why:** The patches complete a standard feature whose core is already committed, for users of temporal tables. They touch RI triggers, PL/pgSQL, ALTER TABLE and pg_dump, so review is costly, and the entry needs a rebase first.

## Appendix

### Refuted in the counter-check

- `standards-compat-2` PL/pgSQL bound variables take a spelling the standard left open. Refuted (no failure scenario): PL/pgSQL trigger variables are a Postgres-specific interface with no standard counterpart, so a future standard syntax in SQL-language trigger definitions can coexist with them. Message #1 already holds back the SQL-language case for that reason.
- `standards-compat-3` NULL bounds may mean different things for range columns and PERIODs. Refuted (no failure scenario): Message #44 only says NULL could be forbidden for PERIODs if the project later wanted that; no patch proposes it. The scenario needs that undecided change plus a user converting a range column to a PERIOD.
- `replication-ondisk-1` tg_temporal exposes an executor struct as trigger API. Refuted (not specific): Duplicate of maintenance-fit-1. Its PERIOD scenario is also weaker, because message #4 builds PERIODs on a hidden range column, so a single range attno still exists.
- `replication-ondisk-2` PERIOD hidden generated columns need a pg_dump and pg_upgrade story. Refuted (not specific): Duplicate of standards-compat-1, which covers the pg_dump treatment of the hidden column. The binary-upgrade attnum question belongs in the answer to that objection.
- `maintenance-fit-3` PERIOD DDL with hidden generated columns adds ALTER TABLE cost. Refuted (not specific): Duplicate of standards-compat-1 for the ALTER, RENAME and pg_dump questions. The rest is a guess about future ALTER TABLE patches, and message #4 reports the message #3 failure as fixed.
- `demand-1` Trigger access to FOR PORTION OF bounds has no shown users. Refuted (already handled): RI actions are triggers and receive their input through TriggerData, so the field is their normal channel. Message #54 makes INSTEAD OF triggers responsible for leftovers, which they cannot compute without the bounds.

### Review angles

- prior-attempts: 0 objection(s). The only close precedent is [P7], and its recorded reason (btree_gist must be in core first) was resolved before temporal keys shipped in v18. Its other points, index ownership and FOR PORTION OF without triggers, were addressed by the committed v19 design. [P1] (revert over empty ranges) and [P3] (system versioning) record reasons that do not apply to the remaining v20 patches.
- standards-compat: 3 objection(s). The core FOR PORTION OF syntax, RETURNING, row count and trigger order were settled and committed in v19, so this review does not reopen them.
- replication-ondisk: 2 objection(s). The committed FOR PORTION OF core adds no WAL records or on-disk format change, and leftovers are published as ordinary INSERTs. The pg_range columns were settled in message #29. Both objections cover only the uncommitted follow-on patches.
- security: 0 objection(s). The one privilege decision is that leftover inserts need no INSERT privilege and still fire INSERT triggers. That rule is in the core patches committed for v19, is documented with a SECURITY DEFINER warning, and is covered by privileges tests. The remaining v20 patches are the TriggerData field, PL/pgSQL variables, temporal FK CASCADE/SET NULL/SET DEFAULT and PERIOD DDL. The FK actions run through the existing RI trigger path, so they follow current cascade practice for the executing role and for RLS, and the other patches add no new privilege or exposure. Not checked: whether INSERT row-level security policies apply to leftover rows. The visible thread does not discuss it, and it belongs to the committed core rather than this entry.
- maintenance-fit: 3 objection(s). Peter Eisentraut is engaged and committed the core FOR PORTION OF patches. The objections cover only the remaining v20 patches.
- already-exists: 0 objection(s). The core FOR PORTION OF patches were committed for v19, and Postgres has no other way to get what the remaining patches add. For a DELETE, a trigger cannot recover the FOR PORTION OF bounds from OLD, NEW or transition tables, so neither user triggers nor temporal CASCADE, SET NULL and SET DEFAULT can be built outside core today. PERIOD DDL may overlap with a plain range column, but it is the standard form, and I cannot confirm that a stored generated range column covers all its uses.
- demand: 1 objection(s). The CASCADE, SET NULL and SET DEFAULT actions and the PERIOD DDL complete standard SQL:2011 features that DB2 and MariaDB already implement, so I do not raise a demand objection against them.

### Precedents retrieved

- P1: [Revert temporal primary keys and foreign keys](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=8aee330af55d8a759b2b73f5a771d9d34a7b887f) (revert, Reverted, 2024-05-16) - used in the review
- P2: [Push down time-related SQLValue functions to foreign server](https://commitfest.postgresql.org/patch/3289/) (cf, Returned with feedback, 2022-08-02) - judged not relevant; reason summarised from its thread
- P3: [WITH SYSTEM VERSIONING Temporal Tables](https://commitfest.postgresql.org/patch/2316/) (cf, Returned with feedback, 2021-11-16) - used in the review; reason summarised from its thread
- P4: [\[WIP\] Temporal query processing with range types - Temporal Normalization](https://commitfest.postgresql.org/patch/2045/) (cf, Withdrawn, 2020-01-30) - judged not relevant
- P5: [Allow TIMESTAMP WITH TIME ZONE to store the original timezone information, either zone name or offset from UTC](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Dates_and_Times) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P6: [Lock updated tuples in tuple_update() and tuple_delete()](https://commitfest.postgresql.org/patch/4099/) (cf, Returned with feedback, 2024-10-08) - judged not relevant; reason summarised from its thread
- P7: [SQL:2011 application time](https://commitfest.postgresql.org/patch/2048/) (cf, Returned with feedback, 2022-02-07) - used in the review; reason summarised from its thread
- P8: [Re: BUG #18741: Detaching a partition referencing a partitioned table fails with a trigger-related error](https://commitfest.postgresql.org/patch/5484/) (cf, Withdrawn, 2025-02-18) - judged not relevant; reason summarised from its thread
- P9: [TimestampTz->Text->TimestampTz casting fails with DateStyle 'Postgres'](https://commitfest.postgresql.org/patch/5371/) (cf, Returned with feedback, 2025-01-07) - judged not relevant; reason summarised from its thread
- P10: [BRIN bloom and multi-minmax indexes](https://commitfest.postgresql.org/patch/1348/) (cf, Returned with feedback, 2019-12-01) - judged not relevant; reason summarised from its thread
- P11: [POC: Lock updated tuples in tuple_update() and tuple_delete()](https://commitfest.postgresql.org/patch/3752/) (cf, Returned with feedback, 2022-10-12) - judged not relevant; reason summarised from its thread
- P12: [Global temporary table](https://commitfest.postgresql.org/patch/2349/) (cf, Returned with feedback, 2022-06-30) - judged not relevant
- P13: [SQL function which allows to distinguish a server being in point in time recovery mode and an ordinary replica](https://commitfest.postgresql.org/patch/5513/) (cf, Withdrawn, 2025-03-17) - judged not relevant
- P14: [Extending the range operators so they can cope with range <op> element and element <op> range in addition to the existing range <op> range](https://commitfest.postgresql.org/patch/2747/) (cf, Returned with feedback, 2021-04-08) - judged not relevant
- P15: [Revert: Allow locking updated tuples in tuple_update() and tuple_delete()](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=193e6d18e553a104d315ff81892d509d89a30fd8) (revert, Reverted, 2024-04-11) - judged not relevant
- P16: [Revert "Use libc version as a collation version on glibc systems."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=50518ec296aea3af3e00c43c2ccef74c96cb5762) (revert, Reverted, 2019-10-09) - judged not relevant
- P17: [Allow WHEN in INSTEAD OF triggers](https://commitfest.postgresql.org/patch/2398/) (cf, Withdrawn, 2020-02-26) - judged not relevant
- P18: [Global temporary tables](https://commitfest.postgresql.org/patch/2233/) (cf, Returned with feedback, 2020-02-01) - judged not relevant
- P19: [Revert "Use Foreign Key relationships to infer multi-column join selectivity".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=77ba610805e7ef9ba9c9a593ea8b1ca8f98f8bcb) (revert, Reverted, 2016-06-07) - judged not relevant
- P20: [Revert temporary patch to debug Windows breakage.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ed0af3324702685cce63aed0641b4cbb45816b50) (revert, Reverted, 2012-07-21) - judged not relevant
- P21: [Range Merge Join](https://commitfest.postgresql.org/patch/3160/) (cf, Returned with feedback, 2022-02-07) - judged not relevant
- P22: [Foreign Key Arrays](https://commitfest.postgresql.org/patch/1252/) (cf, Returned with feedback, 2018-10-02) - judged not relevant
- P23: [test pgbench time-related features](https://commitfest.postgresql.org/patch/3242/) (cf, Returned with feedback, 2021-12-02) - judged not relevant
- P24: [Foreign Key Arrays](https://commitfest.postgresql.org/patch/2966/) (cf, Returned with feedback, 2021-09-14) - judged not relevant

### Warnings

- CommitFest entry has 2 threads; only the earliest is read (https://www.postgresql.org/message-id/flat/ec498c3d-5f2b-48ec-b989-5561c8aa2024%40illuminatedcomputing.com). Others: https://www.postgresql.org/message-id/flat/CA%2BrenyULDQq3x-qYuADAenfqbFts9HaM66zWROZO8X%2B2bJf2GA%40mail.gmail.com
- attachment limit (8) reached; v52-0009-Add-CASCADE-SET-NULL-SET-DEFAULT-for-temporal-fo.patch not read
- attachment limit (8) reached; v52-0010-Expose-FOR-PORTION-OF-to-plpgsql-triggers.patch not read

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 13; wall time: 203 s; cost reported by the CLI: $9.00
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 3 of 24 kept
- Tone check: 95 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T04:11:59.492Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

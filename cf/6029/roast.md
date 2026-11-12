# Design pre-review: NOT NULL NOT ENFORCED

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6029](https://commitfest.postgresql.org/patch/6029/); [thread](https://www.postgresql.org/message-id/flat/CACJufxFbH1_9BDow%3D4nMSdBfLSOAkiGD5hxO6bouWjZAyHbV%2BA%40mail.gmail.com), 18 of 18 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). No surviving objection is fatal, and no level fails. A committer who knows the not-null code reviewed the design, and the author followed him. The open points are fixable: the F492 remark (standards-compat-1), the SET NOT NULL error (standards-compat-2) and unproven dump round-trip for mixed hierarchies (prior-attempts-1). Demand (demand-1, demand-2) rests only on standard conformance, which lowers priority but does not block.

Not checked: Without the code, this review could not check whether information_schema views and other readers of not-null pg_constraint rows test conenforced, or whether v6 fixed the 002_pg_dump.pl failures reported for v4.

## Prior attempts

- [Revert "Improve compression and storage support with inheritance"](https://www.postgresql.org/message-id/flat/24656cec-d6ef-4d15-8b5b-e8dfc9c833a7@eisentraut.org) (related; Reverted, 2024-02-20). "pg_dump cannot currently dump all the structures that are allowed by this patch." Different now: P20 was about compression and storage inheritance, a different feature. Its reason applies because this patch also allows new inheritance states that pg_dump has to reproduce.

## Objections a reviewer is likely to raise

### 1. F492 marked complete while ALTER CONSTRAINT ENFORCED is rejected for not-null

A user reads sql_features.txt, sees F492 listed as YES with no remark, and writes `alter table t alter constraint t_x_not_null enforced` for a not-null constraint, as they would for a foreign key. The server raises an error. The user learns from the error that support covers only part of the syntax the conformance table now claims.

The patch removes the "except not-null constraints" remark, but the v6 commit message (message #17) says ALTER CONSTRAINT ... [NOT] ENFORCED is not supported for not-null constraints. The commit message implies other constraint types accept it, and message #14 cites a separate commit that added enforceability changes elsewhere. So not-null constraints would remain the one type with a partial F492, and only the remark's wording changes. A committer would likely ask for the remark to be rewritten to name the gap rather than dropped.

**Question:** Does F492 in the standard include changing enforcement through ALTER CONSTRAINT, and if so, why should the remark go away before that works?

<sub>fixable, level 1b, SQL-standard conflict or non-standard syntax; messages [#14](https://www.postgresql.org/message-id/CACJufxHvGs7RR97-vN61Y2mZaF2YEgpHScw5SFG%3DTghFQAN9gA%40mail.gmail.com), [#17](https://www.postgresql.org/message-id/CACJufxECCgByCscWPCKCmHtW1f9Riz7S72sQ0HzDht3VQEoWfw%40mail.gmail.com); `standards-compat-1`</sub>

### 2. No user need shown beyond removing the F492 remark

A user who wants an informational "never null" marker today can write CHECK (col IS NOT NULL) NOT ENFORCED. With the patch, that user gets a second spelling with the same runtime effect: nulls are accepted, attnotnull stays false and the proposal does not say the planner uses it. Postgres would then carry a new constraint state that adds nothing for this user.

Message #1 and the v6 commit message give one reason: completing F492 in sql_features.txt. No message in the thread names a user, a field report, an ETL or data-warehouse workflow, or a migration from another system that needs this syntax. Reviewers in messages #10, #15 and #16 test the patch and comment on its wording, but none of them asks for the feature. An unenforced CHECK constraint already gives the same documentation-only meaning.

**Question:** Which users or migration tools need NOT NULL NOT ENFORCED specifically, and why does CHECK (col IS NOT NULL) NOT ENFORCED not serve them?

<sub>fixable, level 1b, demand; messages [#1](https://www.postgresql.org/message-id/CACJufxFbH1_9BDow%3D4nMSdBfLSOAkiGD5hxO6bouWjZAyHbV%2BA%40mail.gmail.com), [#10](https://www.postgresql.org/message-id/176772428435.2452227.15663652319111720473.pgcf%40coridan.postgresql.org), [#15](https://www.postgresql.org/message-id/E4FK57YYN4wTz9_ExrhZVlf9oCdmiIgMZk2av73brzfGnUlJb-cJt9c9sPhqpwnJeUvF9-5xTdWyFh5zWnzJvPITyfAfVAIW8dwPeS386BY%3D%40mannem.nl), [#16](https://www.postgresql.org/message-id/CAF0whuc--DUH%3DJ7aCpbC77id1Vq1CyFbK_xe4qPr8GAVRKuh8g%40mail.gmail.com), [#17](https://www.postgresql.org/message-id/CACJufxECCgByCscWPCKCmHtW1f9Riz7S72sQ0HzDht3VQEoWfw%40mail.gmail.com); `demand-1`</sub>

### 3. SET NOT NULL errors and gives no route to enforce the constraint

A schema migration tool runs `alter table t alter column x set not null` on a column that has a NOT ENFORCED not-null constraint. Today SET NOT NULL either adds the constraint or does nothing. With the patch it raises an error, and the only way forward is to drop the constraint and run SET NOT NULL again.

Message #1 and message #17 make SET NOT NULL, VALIDATE CONSTRAINT and ALTER CONSTRAINT ... ENFORCED all fail on a not-enforced not-null constraint, so the state is one-way. Message #4 suggested `SET NOT NULL NOT ENFORCED` and letting SET NOT NULL do the expected thing, and the thread has no answer to it. SET NOT NULL is the usual Postgres verb for not-null, so a NOT ENFORCED form that only ADD CONSTRAINT can create, and that SET NOT NULL cannot turn into an enforced one, departs from existing practice without a stated reason.

**Question:** Why should SET NOT NULL on a not-enforced not-null constraint raise an error instead of enforcing and validating it, as it does for a column without one?

<sub>fixable, level 2, design; messages [#1](https://www.postgresql.org/message-id/CACJufxFbH1_9BDow%3D4nMSdBfLSOAkiGD5hxO6bouWjZAyHbV%2BA%40mail.gmail.com), [#4](https://www.postgresql.org/message-id/202509051923.n4xc5vthqyhu%40alvherre.pgsql), [#17](https://www.postgresql.org/message-id/CACJufxECCgByCscWPCKCmHtW1f9Riz7S72sQ0HzDht3VQEoWfw%40mail.gmail.com); `standards-compat-2`</sub>

### 4. Dump of mixed-enforcement inheritance trees is not shown to round-trip

A user has a child table that inherits a NOT ENFORCED not-null from one parent and an enforced one from a second parent, so the child gets an enforced constraint. pg_dump decides whether to print the child's not-null from what the parents have, and could leave it out. After restore the child accepts nulls, and nobody sees it until bad rows appear.

[P20] was reverted because pg_dump could not dump every structure the patch allowed, and this patch adds new mixed states to the same inheritance logic. Message #11 shows that flagInhAttrs() decides what to print for inherited not-nulls and was written before enforcement could differ between parent and child. Message #13 reports 002_pg_dump.pl failing for v4, and message #16 tested v5 dumps only in general terms. No message lists the parent and child combinations that were dumped and restored.

Precedent: [Revert "Improve compression and storage support with inheritance"](https://www.postgresql.org/message-id/flat/24656cec-d6ef-4d15-8b5b-e8dfc9c833a7@eisentraut.org) (Reverted, 2024-02-20)

**Question:** For each allowed parent and child combination, including conflicting multiple parents and partitions, does a dump and restore give the same conenforced and attnotnull on every table?

<sub>fixable, level 2, pg_dump / restore round-trip; messages [#11](https://www.postgresql.org/message-id/CACJufxH0130OV4JymGviw8WkB5gehPztsYy3pu-0cmUtL%2BecgA%40mail.gmail.com), [#13](https://www.postgresql.org/message-id/CACJufxEBvceJ4Ju%3D7_LtfcL5ekuXK%3DAWgOB1vAN_svKRYetgyg%40mail.gmail.com), [#14](https://www.postgresql.org/message-id/CACJufxHvGs7RR97-vN61Y2mZaF2YEgpHScw5SFG%3DTghFQAN9gA%40mail.gmail.com), [#16](https://www.postgresql.org/message-id/CAF0whuc--DUH%3DJ7aCpbC77id1Vq1CyFbK_xe4qPr8GAVRKuh8g%40mail.gmail.com); `prior-attempts-1`</sub>

### 5. Every future not-null change pays for a state few will use

A later patch changes not-null handling in inheritance, partition attach, MERGE/SPLIT PARTITIONS or pg_dump. It must also handle the not-enforced case and its parent/child rules. If it misses that case, mixed hierarchies dump or attach wrongly, and someone finds out at restore or attach time.

The rules span multiple inheritance, partitions, MERGE/SPLIT PARTITIONS (which v4 broke, message #14), pg_dump flagInhAttrs (message #11) and a changed meaning of attnotnull. The author notes that changing enforceability later would bring the complexity of an earlier commit, which future work must face (message #14). All later not-null work carries this cost, but the gain is limited to users of an unenforced not-null constraint, and the thread shows none. An engaged committer (messages #2, #12) lowers the risk but does not supply the demand.

**Question:** What use of not-enforced not-null constraints justifies the extra inheritance, partitioning and pg_dump rules that every later not-null patch must keep correct?

<sub>fixable, level 1b, maintenance burden ("who maintains this in 10 years"); messages [#11](https://www.postgresql.org/message-id/CACJufxH0130OV4JymGviw8WkB5gehPztsYy3pu-0cmUtL%2BecgA%40mail.gmail.com), [#12](https://www.postgresql.org/message-id/202602231223.yggz76dffiro%40alvherre.pgsql), [#14](https://www.postgresql.org/message-id/CACJufxHvGs7RR97-vN61Y2mZaF2YEgpHScw5SFG%3DTghFQAN9gA%40mail.gmail.com); `demand-2`</sub>

## Questions for the author

1. Does F492 in the standard include changing enforcement through ALTER CONSTRAINT, and if so, should the sql_features.txt remark name that gap instead of being removed?
2. Why should SET NOT NULL on a column with a not-enforced not-null constraint raise an error instead of enforcing and validating the constraint?
3. For each allowed parent and child combination, including conflicting multiple parents and partitions, does dump and restore give the same conenforced and attnotnull on every table?
4. Does the planner or executor ever use a not-enforced not-null constraint, for example to prove IS NOT NULL?
5. Which users or migration tools need NOT NULL NOT ENFORCED, and why does CHECK (col IS NOT NULL) NOT ENFORCED not serve them?

## Touch points

- **catalog changes:** pg_attribute.attnotnull now means an enforced not-null constraint exists; pg_constraint rows for not-null can have conenforced false.
- **ABI / extension API:** The patch touches tupdesc.h, heap.h and pg_constraint.h, which extensions include.
- **SQL-standard conflict or non-standard syntax:** The patch adds NOT NULL NOT ENFORCED syntax in gram.y and removes the F492 remark in sql_features.txt.
- **planner behaviour change:** relcache.c and tupdesc.h change how column nullability is computed; the thread does not say whether the planner uses a not-enforced constraint.
- **pg_dump / restore round-trip:** pg_dump.c emits NOT NULL NOT ENFORCED inline, and 002_pg_dump.pl failed for v4.
- **psql UX conventions:** describe.c changes \d output, and the tab completion question has no visible answer.
- **maintenance burden ("who maintains this in 10 years"):** Inheritance, partitioning, MERGE/SPLIT PARTITIONS, LIKE and pg_dump code must each handle a not-enforced not-null state.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** CHECK (col IS NOT NULL) NOT ENFORCED already gives the same runtime behavior.

## Who would commit it, and when

- Owner: qualified committer engaged. Álvaro Herrera is the listed committer and reviewed the design in message #2; the work needs someone who knows not-null constraint inheritance and pg_dump.
- Priority: proceed; audience niche; review cost high. The thread gives only standard conformance as the reason, so the audience is small. The patch is about 89 KB across tablecmds.c, the catalog code and pg_dump, but a qualified committer is engaged and the open points are fixable.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 4 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

# Design pre-review: Reject ADD CONSTRAINT NOT NULL if name mismatches existing domain not-null constraint

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6564](https://commitfest.postgresql.org/patch/6564/); [thread](https://www.postgresql.org/message-id/flat/CACJufxEcTx2U8tQTcJ2g490wzd%3DjQgJNDpNb2L%2Bf3gr9J%3DcpNA%40mail.gmail.com), 4 of 4 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). No surviving objection questions the goal on master: v1 extends the table rule from commit 96e2af6 to domains, and message #3 confirms the one-constraint rule. standards-compat-1 is about scope, because the entry targets stable branches and no wrong result from the silent success is shown. standards-compat-2 applies only to the v2 NOTICE variant, which departs from the table behaviour. Both points are fixable and need a committer's decision on ERROR versus NOTICE and on back branches.

Not checked: The code was not read, so this review cannot confirm that v1 leaves unnamed `add not null` and `set not null` unchanged. Message #4 was judged only from the brief's summary of it.

## Prior attempts

None found before 2026-03-01.

## Objections a reviewer is likely to raise

### 1. New ERROR is proposed for stable branches as a bugfix

A migration tool re-runs `alter domain d add constraint d_nn not null` on a domain whose not-null constraint has a different name, often the default one. After a minor upgrade the statement raises an ERROR and the transaction aborts. The operator finds out when a deployment that worked last week fails.

The CommitFest entry is tagged Bugfix and targets stable branches. v1 turns a statement that succeeds today into an ERROR. Message #4 raises this compatibility cost, and no one has answered it. The thread does not show that the current silent success causes wrong results or corruption, which is what would justify a behaviour change in a minor release.

**Question:** What wrong result does the current silent success cause that justifies a new ERROR in minor releases, rather than a master-only change?

<sub>fixable, level 1b, backpatchability; messages [#1](https://www.postgresql.org/message-id/CACJufxEcTx2U8tQTcJ2g490wzd%3DjQgJNDpNb2L%2Bf3gr9J%3DcpNA%40mail.gmail.com), [#4](https://www.postgresql.org/message-id/tencent_84E11BD212851AB351001EA90D8D96782205%40qq.com); `standards-compat-1`</sub>

### 2. v2 NOTICE design makes domains diverge from tables

A user runs `alter domain d add constraint other_name not null` on a domain that already has a not-null constraint. Under v2 it succeeds with a NOTICE, but the matching table command errors after 96e2af6. Scripts that repeat `set not null` and `drop not null` also start printing NOTICEs on every run.

The stated goal in message #1 is to align domains with tables. v2 in message #4 keeps the name mismatch that 96e2af6 rejects for tables. It also adds NOTICEs to `set not null` and `drop not null`, and the thread does not show that the table forms emit them. That is a second, larger departure from existing practice, and v2 gives no reason for it beyond preserving silent success.

**Question:** If domains should match tables, why should a mismatched name be accepted with a NOTICE, and do repeated table `set not null` commands emit a NOTICE today?

<sub>fixable, level 2, design; messages [#1](https://www.postgresql.org/message-id/CACJufxEcTx2U8tQTcJ2g490wzd%3DjQgJNDpNb2L%2Bf3gr9J%3DcpNA%40mail.gmail.com), [#4](https://www.postgresql.org/message-id/tencent_84E11BD212851AB351001EA90D8D96782205%40qq.com); `standards-compat-2`</sub>

## Questions for the author

1. What wrong result does the current silent success cause that justifies a new ERROR in minor releases, compared with a master-only change?
2. Which behaviour is proposed for commit: the v1 ERROR that matches tables after 96e2af6, or the v2 NOTICE that keeps the mismatched name accepted?
3. If v2 is preferred, why should repeated `set not null` and `drop not null` on a domain emit a NOTICE when the table forms do not?
4. Does the `alter domain` documentation page need to state that a named not-null constraint must match the existing constraint name?

## Touch points

- **SQL-standard conflict or non-standard syntax:** see objection standards-compat-1: New ERROR is proposed for stable branches as a bugfix
- **psql UX conventions:** see objection standards-compat-2: v2 NOTICE design makes domains diverge from tables
- **backpatchability:** The CommitFest entry is tagged Bugfix with target version stable, and v1 turns a statement that succeeds today into an ERROR.

## Who would commit it, and when

- Owner: qualified committer engaged. Álvaro Herrera answered the order-dependence question in message #3; the owner needs to know the not-null constraint work for tables and domains in typecmds.c.
- Priority: proceed; audience niche; review cost low. The v1 patch is 4.5 KB, touches only typecmds.c and the domain regression test, and follows an accepted table rule. Few users add a named not-null constraint to a domain that already has one, and the open points are the back-branch scope and ERROR versus NOTICE.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 2 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

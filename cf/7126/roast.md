# Design pre-review: Per-table resync for logical replication subscriptions

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7126](https://commitfest.postgresql.org/patch/7126/); [thread](https://www.postgresql.org/message-id/flat/CAA36mso9zgvMVr4UNHMMAQyU-dQy33EqxjZ%3DyEDG8SAB6yYDJw%40mail.gmail.com), 36 of 36 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No objection to the goal survived: message #1 shows by test that the existing workarounds are non-local or race the apply worker. Three fixable level 2 points remain. replication-ondisk-1 is the strongest: a refresh on a forwarding node can stop a downstream apply worker. maintenance-fit-1 and standards-compat-1 concern lock-order invariants and trigger rules that only a committer who knows logical replication can accept.

Not checked: The lock order, the slot-then-origin drop sequence and the placement of the origin on the truncate WAL record could not be checked without the code. Only 3 of 36 messages were read directly, so later discussion is known only through the objections.

## Prior attempts

- [Add `truncate` option to subscription commands](https://www.postgresql.org/message-id/flat/CF3B6672-2A43-4204-A60A-68F359218A9B@endpoint.com) (similar idea; Withdrawn, 2020-11-25). reason not recorded Different now: P10 added a truncate option to existing subscription commands. This proposal adds a separate command that truncates and re-copies tables the subscription already tracks, with origin tagging, cross-subscription checks and slot cleanup.

## Objections a reviewer is likely to raise

### 1. Refresh on a forwarding node can stop a downstream subscriber's apply worker

In A -> B -> C, B's publication omits truncate and C uses origin = any. An operator runs REFRESH TABLE on B, and the re-copied rows flow to C, which still holds the old rows. C logs insert_exists, its apply worker stops, and every table C subscribes to falls behind.

Message #28 shows the case itself: the 'withholds TRUNCATE, origin = any' row ends with C stale and its apply worker stopped. The patch documents this but neither detects nor refuses it. The command runs on B, and B can see its own publications on the table and their publish options. A local repair on one node therefore halts replication on another node, and the operator finds out only from C's server log.

**Question:** Why does the command not warn or refuse when the node republishes the table through a publication that does not publish truncate?

<sub>fixable, level 2, WAL / replication / standby; messages [#27](https://www.postgresql.org/message-id/OS7PR01MB1831757BF5C212B9E04F7E615F5642%40OS7PR01MB18317.jpnprd01.prod.outlook.com), [#28](https://www.postgresql.org/message-id/CAA36mspCc4wqZJMuPTxG_sy15_yBkY3sw%3DTB9Y1orrp-uS6r9Q%40mail.gmail.com); `replication-ondisk-1`</sub>

### 2. Command duplicates tablesync and catalog lock-order invariants with no loud failure

A later patch changes the lock level REFRESH PUBLICATION, CREATE SUBSCRIPTION or the tablesync worker takes on pg_subscription_rel, or adds a sync state. REFRESH TABLE keeps its own ordering and slot-then-origin drop sequence, so a deadlock or a lost-row case comes back. Users find out in production, through a worker killed by the deadlock detector or rows that are never copied back.

Each review round found a new interaction outside the command: a deadlock with a tablesync worker (#15, #16), with user transactions (#16), with partition lock order (#23), with REFRESH PUBLICATION (#29), and an origin Assert reached from triggers (#34). The fixes rest on ordering rules held in comments and TAP tests, and other subscription commands do not depend on those rules. Sawada, a committer, reviewed v3 (#9), but no committer has commented on the design since.

**Question:** Which lock-order and state-machine invariants does REFRESH TABLE depend on, and what fails at build or test time if another subscription command changes them?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#9](https://www.postgresql.org/message-id/CAD21AoAe3XmT03NLqLNEJbiYb3dyW3iTBY-RV51qYDy%2Bs0zpLA%40mail.gmail.com), [#15](https://www.postgresql.org/message-id/OS9PR01MB12149489D6B3C370CE8FEC97CF5A72%40OS9PR01MB12149.jpnprd01.prod.outlook.com), [#16](https://www.postgresql.org/message-id/CAA36mspx4Szp02%3DG5Lop9gyeqv8r2yKs-R5%2B8xB-YTtv5d9e0Q%40mail.gmail.com), [#23](https://www.postgresql.org/message-id/TY5PR01MB183149575EEA744B1CFA8BA6DF5812%40TY5PR01MB18314.jpnprd01.prod.outlook.com), [#29](https://www.postgresql.org/message-id/CANhcyEVjG62LDOuUTDAAa_iS%3DLTxUzqSm1RhRHBh_B8vewMY8g%40mail.gmail.com), [#34](https://www.postgresql.org/message-id/CANhcyEW66sNdiie1GdSusUCh1njEbVcC7PkyfFSoh4Yx%3DGpTtA%40mail.gmail.com); `maintenance-fit-1`</sub>

### 3. Truncate and re-copy fire different trigger sets

A subscriber keeps an ordinary AFTER TRUNCATE trigger and a row trigger that maintain a summary table. REFRESH TABLE fires the truncate trigger in the user's session, but the tablesync re-copy runs as replica and skips the ordinary row trigger. The summary table stays empty, and the user learns this only when reports go wrong.

No other ALTER SUBSCRIPTION command fires user triggers (message #33). Under v15 one operation has two halves with different trigger rules. Message #35 keeps TRUNCATE semantics and points users to ENABLE ALWAYS, but that changes how the trigger behaves for all replicated changes. Nobody has replied to that choice.

**Question:** Why should one command fire the session's TRUNCATE triggers for the discard but only replica or always triggers for the refill, and not one rule for both halves?

<sub>fixable, level 2, design; messages [#33](https://www.postgresql.org/message-id/CANhcyEWG6DsUudBSnj7Swsf%2BsJXioDgAphtHXFvgimqXJ%2Bq2SQ%40mail.gmail.com), [#35](https://www.postgresql.org/message-id/CAA36msotccBGaiej3n3qsUgxAqjtwpB-F3vYt8UoW0krHUFK9w%40mail.gmail.com); `standards-compat-1`</sub>

## Questions for the author

1. Should REFRESH TABLE warn or refuse when the node republishes the table through a publication that publishes inserts but not truncate?
2. Which lock-order and sync-state invariants does REFRESH TABLE depend on, and which test fails if another subscription command changes them?
3. Why does the discard fire the session's TRUNCATE triggers while the re-copy fires only replica or always triggers, instead of one rule for both?
4. Would a later enabled-subscription version keep truncate-and-recopy with the table empty during sync, or use a different mechanism?

## Touch points

- **WAL / replication / standby:** The truncate WAL record carries the subscription's origin, and the command drops tablesync slots and origins.
- **catalog changes:** The command resets pg_subscription_rel rows to init from a user backend; it adds no catalog column.
- **ABI / extension API:** ExecuteTruncateGuts() gains an origin parameter, and three truncate_check functions become extern.
- **SQL-standard conflict or non-standard syntax:** New ALTER SUBSCRIPTION name REFRESH TABLE table_name [, ...] syntax in gram.y.
- **psql UX conventions:** The patch adds tab completion in src/bin/psql/tab-complete.in.c.
- **security / privileges:** The command needs TRUNCATE privilege on each table and fires TRUNCATE triggers in the caller's session.
- **maintenance burden ("who maintains this in 10 years"):** The command keeps its own lock order and slot-then-origin drop sequence next to tablesync and REFRESH PUBLICATION.

## Who would commit it, and when

- Owner: qualified committer engaged. Masahiko Sawada is the CommitFest reviewer and reviewed early versions, but nobody has replied to v14 or v15. The owner must know tablesync, replication origins and truncate.
- Priority: proceed; audience a sizeable minority; review cost high. The need is shown by test in message #1 and the author responds to review. The two patches total about 135 KiB across subscriptioncmds.c, tablecmds.c and worker.c, and each review round found a new lock-order or origin problem.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 6 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

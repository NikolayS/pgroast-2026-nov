# Design pre-review: Introducing find_all_inheritors_ordered()

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7074](https://commitfest.postgresql.org/patch/7074/); [thread](https://www.postgresql.org/message-id/flat/57100518-3FF1-48AD-B550-47B9A3E88688%40gmail.com), 10 of 10 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). No surviving objection is fatal, and no level fails. demand-1 asks for confirmation that the v3 refactor leaves find_all_inheritors() results and lock order unchanged, since many commands use that function and no one has replied to v3. maintenance-fit-1 asks for a second caller and for a guard against choosing the cycle-raising function on a path that must tolerate cycles. Both are fixable, and a committer who knows the inheritance locking code can settle them.

Not checked: Without the code, this review could not check whether v3 keeps find_all_inheritors() lock order and results, or whether the cycle check covers every cycle shape. Only 3 of 10 messages were shown in full.

## Prior attempts

None found before 2026-07-30.

## Objections a reviewer is likely to raise

### 1. One caller does not yet justify a shared traversal refactor

v3 changes the internals of find_all_inheritors(), which TRUNCATE, ANALYZE, LOCK and other DDL use on every inheritance tree. If the shared data-gathering step changes lock or result order in some case, users of those commands see new deadlocks or different behaviour. They find out through a bug report long after commit, and the only gain is a cleaner ALTER CONSTRAINT ... NOT ENFORCED path.

The only user problem shown is code shape. The fix in 0cd17fdd3c0 already works, and v3 keeps master's success and failure outcomes (message #8), apart from the error text in a cycle case that is reachable only through concurrent ALTER TABLE ... INHERIT (messages #4, #10). The claim that other features need parent-before-child order (message #1) names no feature. Message #7 shows the two functions are tied by lock order, so the risk of the shared refactor reaches every caller of find_all_inheritors(), while the benefit stays with one code path.

**Question:** Which existing or planned callers besides the NOT ENFORCED path need parent-before-child order, and has anyone confirmed find_all_inheritors() results and lock order are unchanged?

<sub>fixable, level 1b, desirability; messages [#1](https://www.postgresql.org/message-id/57100518-3FF1-48AD-B550-47B9A3E88688%40gmail.com), [#7](https://www.postgresql.org/message-id/460958.1788824290%40sss.pgh.pa.us), [#8](https://www.postgresql.org/message-id/478CC29D-1E62-4804-9FF0-C8E25E604FEE%40gmail.com), [#10](https://www.postgresql.org/message-id/CAN4CZFNBn%2BrQst%3D6_3HDLksbT_EHe_5SUc0X5ECu%3D3HV1gE0vA%40mail.gmail.com); `demand-1`</sub>

### 2. Second traversal API with different cycle semantics and one caller

A later patch adds a DDL path and calls find_all_inheritors_ordered() because it wants parent-before-child order. Tests pass, since cycles appear only through concurrent ALTER TABLE ... INHERIT. Months later a user with such a cycle finds that the command now errors where find_all_inheritors() would have coped.

After v3, pg_inherits.h exports two functions that return the same set of OIDs but differ on cycles: one tolerates them, the other raises an error (message #5, message #8). The only rule for choosing between them is a header comment. The patch has one caller, and even ALTER CONSTRAINT splits by direction because the ordered function is unsafe for ENFORCED (message #6, message #8). The claim that other features could use it names no second caller.

**Question:** Which other current or planned callers need ordered traversal, and how would a reviewer catch a caller that picks the cycle-raising function on a path where cycles must be tolerated?

<sub>fixable, level 2, ABI / extension API; messages [#5](https://www.postgresql.org/message-id/20EAC3F9-81A2-41DD-89FA-1539EA8ED11B%40gmail.com), [#6](https://www.postgresql.org/message-id/CAN4CZFNZD-1_XcEA%2BBGAi9_KgqzuONzecyVsMu9sB1Xp8%2BgHkg%40mail.gmail.com), [#8](https://www.postgresql.org/message-id/478CC29D-1E62-4804-9FF0-C8E25E604FEE%40gmail.com); `maintenance-fit-1`</sub>

## Questions for the author

1. Has anyone confirmed that find_all_inheritors() returns the same results and takes locks in the same order after the v3 refactor, and how was that checked?
2. Which existing or planned callers besides the NOT ENFORCED path need parent-before-child order?
3. How would a reviewer catch a new caller that uses find_all_inheritors_ordered() on a path where an inheritance cycle must be tolerated?
4. What are the message text and SQLSTATE of the new cycle error?
5. Does v3 include the isolation test from message #6 and the comment cleanup and typedefs.list entry from message #3?

## Touch points

- **ABI / extension API:** The patch exports a new function, find_all_inheritors_ordered(), in src/include/catalog/pg_inherits.h.
- **maintenance burden ("who maintains this in 10 years"):** Two traversal functions with different cycle behaviour share data gathering and locking, and the ordered one has one caller.

## Who would commit it, and when

- Owner: qualified committer engaged. Tom Lane asked for shared data gathering in message #7 but has not replied to v3. The owner needs to know inheritance traversal and DDL lock order.
- Priority: proceed; audience niche; review cost medium. The patch is a backend cleanup of one call site, about 19 KiB across pg_inherits.c and tablecmds.c, with no end-user surface beyond one error message. Review cost is medium because the v3 refactor changes the internals of find_all_inheritors() and lock order needs a careful read.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 2 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

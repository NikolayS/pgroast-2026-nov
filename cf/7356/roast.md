# Design pre-review: [PATCH v1] Fix for Bug#19724 - ALTER TYPE ... ALTER ATTRIBUTE triggers internal error for base type of domain with check

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7356](https://commitfest.postgresql.org/patch/7356/); [thread](https://www.postgresql.org/message-id/flat/CAH5HC94%2B4teDZvuWkBiAikcK1pM2DH9W6iCNDoGpHxUdPGTNPw%40mail.gmail.com), 11 of 11 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). The goal is a bug fix for an internal error and a possible crash on every supported branch, and reviewers agree on 0001. Two fixable design points remain for stable branches: replication-ondisk-2 (0003 widens the name-based domain lookup race to domains the caller does not own) and standards-compat-1 (0002 touches an exported prototype in typecmds.h). Neither blocks the idea, but a committer must decide what goes to back branches.

Not checked: Without the code, I could not check whether 0002 changes the existing AlterDomainAddConstraint() signature or only adds a declaration. I also could not check how the deferred validation orders its catalog updates or which identity it runs under.

## Prior attempts

None found before 2026-09-28.

## Objections a reviewer is likely to raise

### 1. Name-based re-add can attach a constraint to the wrong domain

Session A runs ALTER TYPE while session B renames domain d and creates a new domain d over the same type. The rebuilt d_check is stored in pg_constraint on the new domain, and the original domain loses it. Later inserts into columns of the original domain skip the check, and pg_dump keeps the wrong state.

Message #10 shows this race succeeding silently, and message #11 defers the fix to a separate patch that would not be back-patched. The bug is older than this patch, but 0003 removes the ownership check, so after v4 the stale lookup also succeeds when the domain belongs to another user. 0002 already tracks constraint OIDs in the work queue, so resolving the domain by OID may not need much more code.

**Question:** Can the re-add use the domain OID taken before the drop, so that 0003 does not ship to stable branches while the name lookup can still pick the wrong domain?

<sub>fixable, level 2, catalog changes; messages [#8](https://www.postgresql.org/message-id/CAN4CZFMhvTw7J8kNcU%2BZxxWq2oUg%2B-hiaeVH%3DsdEt1%2BrtOK3Pg%40mail.gmail.com), [#10](https://www.postgresql.org/message-id/DLRUNGYX33TS.2M17VTR2OBXPC%40gmail.com), [#11](https://www.postgresql.org/message-id/CAH5HC945XpFgKhZK9vQug5-A%3D_-dWRt2_LzOp9unA4JEAA941Q%40mail.gmail.com); `replication-ondisk-2`</sub>

### 2. 0002 changes an exported function signature in a back-patched fix

An extension built against an earlier minor release of 14 through 19 calls AlterDomainAddConstraint(). After a minor upgrade, the server passes or expects arguments the extension's compiled call does not supply. The extension then fails at load time, or misbehaves at run time.

0002 touches src/include/commands/typecmds.h and changes AlterDomainAddConstraint(), so the fix depends on a changed prototype. Message #3 supports back-patching to REL_14_STABLE through REL_19_STABLE, and the CommitFest entry targets stable. The thread checks only that v1 applies cleanly. Nobody has said whether 0002 goes to the back branches, or how its signature change would be kept ABI-compatible there.

**Question:** For back branches, will 0002 keep the existing AlterDomainAddConstraint() signature, for example with a new wrapper function, so extensions built against earlier minor releases keep working?

<sub>fixable, level 2, backpatchability; messages [#3](https://www.postgresql.org/message-id/CAJJjRRf%2Bb21kqB6-fKCBAqrOhBT12K5%2BV-Pa6wCpZi%2BD9xKiCg%40mail.gmail.com), [#5](https://www.postgresql.org/message-id/DLR67P6TO2C8.3SL4J7XR66QBJ%40gmail.com), [#11](https://www.postgresql.org/message-id/CAH5HC945XpFgKhZK9vQug5-A%3D_-dWRt2_LzOp9unA4JEAA941Q%40mail.gmail.com); `standards-compat-1`</sub>

## Questions for the author

1. Which of 0001, 0002 and 0003 are meant for back-patching to REL_14_STABLE through REL_19_STABLE?
2. Can the re-add use the domain OID taken before the drop, so 0003 does not reach stable branches while the name lookup can pick another user's domain?
3. For back branches, does 0002 keep the existing AlterDomainAddConstraint() signature, for example through a new wrapper function?
4. Does the new validation loop after phase 3 need CommandCounterIncrement() between constraints, and what case shows it?
5. Will the stack depth error seen with a ROW(VALUE, NULL)::t check be handled in this patch set or left as is?

## Touch points

- **catalog changes:** The rebuild drops and re-creates pg_constraint rows for domain CHECK constraints, and finds the domain by name.
- **ABI / extension API:** 0002 changes AlterDomainAddConstraint() in src/include/commands/typecmds.h, which extensions can call.
- **security / privileges:** 0003 skips the domain ownership check on re-add and restores comments with CreateComments().
- **backpatchability:** The CommitFest entry targets stable, and 0002 and 0003 change a C prototype and permission behaviour.
- **maintenance burden ("who maintains this in 10 years"):** 0002 adds a deferred validation step for domain constraints after phase 3 in ATRewriteTables().

## Who would commit it, and when

- Owner: silent. Someone who knows the ALTER TABLE phase 2 and phase 3 work queue in tablecmds.c and the domain constraint code in typecmds.c; no such committer has spoken in the visible thread.
- Priority: needs champion; audience niche; review cost medium. The bug gives an internal error or a crash on all supported branches, but only for domains whose CHECK references an altered composite or row type. The three patches are small, yet back-patching 0002 and 0003 needs a committer's decision and none is engaged.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 5 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

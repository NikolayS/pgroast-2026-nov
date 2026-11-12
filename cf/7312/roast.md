# Design pre-review: BUG #19686: Rolling back SET TABLESPACE

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7312](https://commitfest.postgresql.org/patch/7312/); [thread](https://www.postgresql.org/message-id/flat/CAE8JnxP%2BUbdj-kaxfgNt%3D3_FzD3GgOc3gEvPAvrXKfgTgH2q5w%40mail.gmail.com), 33 of 33 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). The bug is real and committers in the thread back the eager index copy, so levels 1a and 1b pass. Two fixable design points remain at level 2, both about the shortcut and the back branches: standards-compat-1 (space needs depend on the protocol message type) and replication-ondisk-1 (an uninitialized isTopLevel from an extension caller can skip the copy). No committer has replied to v6 or v7, so the choice between them is still open.

Not checked: Without the code, it could not be checked whether any non-plain ALTER path or extension caller still reaches the shortcut, or whether WAL replay of the index copies is correct. The pinned-buffer tests for the deferred copy cover only btree and hash in one backend.

## Prior attempts

None found before 2026-09-16.

## Objections a reviewer is likely to raise

### 1. Back-branch ABI change lets an uninitialized field skip the copy

An extension built against unpatched minor-release headers fills AlterTableUtilityContext on the stack and calls AlterTable() for SET TABLESPACE. The new trailing isTopLevel field holds uninitialized stack memory, so the shortcut may be taken where it should not be. A later write and abort in that statement leave stale index entries, found later as wrong results or nbtree assertion failures.

Message #28 says an uninitialized value "can at worst skip the copy, as today", but today's behavior is the corruption the patch fixes. So an old-header build silently keeps the bug on a patched server, and nothing reports it. The material does not say which other conditions, such as the simple-Query and transaction-block checks, still guard this path when the caller is an extension. The author already offers to always copy in back branches (message #28), which avoids the struct change.

Precedent: [Fix index-only scan plans, take 2.](https://www.postgresql.org/message-id/flat/3179992.1641150853@sss.pgh.pa.us) (Partially reverted, 2022-01-03)

**Question:** With an uninitialized isTopLevel from an extension caller, which independent check still forces the index copy, or will back branches always copy instead?

<sub>fixable, level 2, ABI / extension API; messages [#28](https://www.postgresql.org/message-id/179110230672.1690838.3721224931432361508%40gmail.com), [#31](https://www.postgresql.org/message-id/179195789433.990372.6387431139267045114%40gmail.com); `replication-ondisk-1`</sub>

### 2. Space needs of SET TABLESPACE depend on the client protocol

After a minor update, an application sends a plain `alter table t set tablespace ts` through pgJDBC or pgx while the volume holding the indexes is nearly full. The ALTER now fails with "could not extend file", but the same statement typed in psql succeeds. The DBA finds this at the next routine move, on a back branch.

With v7, whether the indexes are copied depends on the wire message type (simple Query or extended protocol), not only on what the SQL says. Message #32 shows a single extended-protocol ALTER failing with ENOSPC where master succeeds. Existing top-level checks such as PreventInTransactionBlock use isTopLevel and XACT_FLAGS_NEEDIMMEDIATECOMMIT, and v7 adds a new IsExtendedQueryMessage() accessor in their place. No committer has ruled on v6 versus v7 (messages #29 to #31), and the material has no documentation hunk for the new space rule.

**Question:** Why should a minor release make the same SET TABLESPACE need extra space in the index tablespace only for extended-protocol clients, and where will users read about it?

<sub>fixable, level 2, backpatchability; messages [#28](https://www.postgresql.org/message-id/179110230672.1690838.3721224931432361508%40gmail.com), [#29](https://www.postgresql.org/message-id/CAGRkXqSwuvuYrKdf5o9Jn%3DJqkdO7YT0%2BnTS6L9tf_%2BHu-_gWaQ%40mail.gmail.com), [#30](https://www.postgresql.org/message-id/179114425825.497956.17396695057926688547%40gmail.com), [#31](https://www.postgresql.org/message-id/179195789433.990372.6387431139267045114%40gmail.com), [#32](https://www.postgresql.org/message-id/CAGRkXqRKCNG1LKBV7d98QSdZKf1iT8Qb7LM9jTDKofn33hdsyw%40mail.gmail.com), [#33](https://www.postgresql.org/message-id/179234193175.237803.1636724777877426621%40gmail.com); `standards-compat-1`</sub>

## Questions for the author

1. With an uninitialized isTopLevel from an extension built against unpatched headers, which independent check still forces the index copy?
2. Should back branches always copy the indexes and leave AlterTableUtilityContext unchanged, as message #28 offers?
3. Is it acceptable in a minor release that a single extended-protocol SET TABLESPACE fails with ENOSPC where master succeeds (message #32)?
4. Where will users read that SET TABLESPACE can need free space of about the index size in the indexes' own tablespace, and when the shortcut applies?
5. Which version is proposed for commit: v6, v7, or a deferred-copy v8 for 19 and master?

## Touch points

- **on-disk format / pg_upgrade:** Each index gets a new relfilenumber and a copied file when its table moves.
- **WAL / replication / standby:** Each index copy writes WAL of about the index size, which standbys replay.
- **wire protocol:** v7 skips the index copy only for a lone simple Query, using the new IsExtendedQueryMessage().
- **ABI / extension API:** Back branches add a field at the end of AlterTableUtilityContext, which extensions build.
- **SQL-standard conflict or non-standard syntax:** see objection standards-compat-1: Space needs of SET TABLESPACE depend on the client protocol
- **backpatchability:** The fix targets REL_14 to REL_19, with new space needs and a struct change in minor releases.
- **maintenance burden ("who maintains this in 10 years"):** The shortcut conditions in tablecmds.c depend on protocol state and event trigger checks.

## Who would commit it, and when

- Owner: qualified committer engaged. Tom Lane, Andres Freund and Michael Paquier set the eager-copy direction in the thread, but none has replied to v6, v7 or the deferred-copy question. The owner needs to know tablecmds.c, relation storage and back-branch ABI rules.
- Priority: proceed; audience a sizeable minority; review cost medium. This fixes reproducible index corruption and wrong results (message #1), and committers already agreed on the eager copy. The patch touches mainly tablecmds.c, but the back-branch and shortcut choices need a committer's ruling.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 4 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

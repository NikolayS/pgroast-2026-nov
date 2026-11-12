# Design pre-review: Bug: Whole-row var in indexes corrupts indexes after DDL

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7071](https://commitfest.postgresql.org/patch/7071/); [thread](https://www.postgresql.org/message-id/flat/CAEze2WjDaDyvztdXh3Cb2J%3D11CMVRp4NWAW0E6dBRKa1T9w5ag%40mail.gmail.com), 9 of 9 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). The goal is not in question: the thread shows index corruption and message #2 lists reports of this class since 2020. The surviving objections are fixable design points about the catalog representation (replication-ondisk-2, maintenance-fit-1) and the missing back-branch plan (replication-ondisk-1). A committer who knows pg_depend and the object address code has to decide on the sentinel subid and the new ObjectType value.

Not checked: Without the code, the review cannot check how aclchk.c, seclabel.c and dropcmds.c treat OBJECT_WHOLE_ROW, or whether every reader of refobjsubid handles -32768. Messages #4 to #9 were not shown in full.

## Prior attempts

None found before 2026-07-29.

## Objections a reviewer is likely to raise

### 1. New refobjsubid value breaks the assumption that a nonzero subid is a column

A later patch or an extension walks pg_depend and treats any nonzero refobjsubid as an attnum. When it meets -32768 it fails with an error like "column -32768 of relation ... does not exist", or it silently skips the row. The failure shows up only for tables that have whole-row expressions.

Until now refobjsubid was either 0 or a positive attnum. Review found two consumers that missed the new value: process_function_rte_ref() in message #2 and the object address functions in messages #2 and #5. Between v1 and v5 the patch grew from three backend files to seven, adding aclchk.c, dropcmds.c, seclabel.c and event_trigger.c. Nothing makes a missed consumer fail at build time, and the material does not compare this design with other ways to mark whole-row rows in pg_depend.

**Question:** Which code in core reads refobjsubid, has each place been checked for WholeRowAttrNumber, and why use a magic subid instead of another marker?

<sub>fixable, level 2, catalog changes; messages [#2](https://www.postgresql.org/message-id/BEE8820C-50BD-467A-BDBC-F95827D737EB%40yandex-team.ru), [#5](https://www.postgresql.org/message-id/F4161B2A-79DC-453A-BCE1-4A22139B0664%40yandex-team.ru), [#7](https://www.postgresql.org/message-id/CAEze2WhuqrHBhUkt_QNYTq1YajV_unu3NVJKnOvsjizaLDo%3DMA%40mail.gmail.com); `replication-ondisk-2`</sub>

### 2. OBJECT_WHOLE_ROW adds a public object type that is not a real object

An extension or event trigger that switches over ObjectType gets OBJECT_WHOLE_ROW after upgrade and errors or ignores it. A later patch adds an ObjectType-wide feature and must decide again what a whole-row "object" means for GRANT, SECURITY LABEL, DROP or COMMENT. Nothing fails at build time if that decision is skipped.

v4 added OBJECT_WHOLE_ROW only to make the pg_identify_object_as_address() round trip work (message #5, message #7). As a result, v5 touches aclchk.c, seclabel.c, dropcmds.c and event_trigger.c, though a whole row cannot be granted, labelled or dropped on its own. Message #7 also notes that "<relkind> column" addresses already fail the round trip for indexes and views, so the round trip does not force a new enum value. A committer would likely ask for a representation that reuses the attribute path with subobject -32768.

**Question:** Why can the whole-row pseudo-subobject not be encoded through the existing OBJECT_COLUMN address path, without a new ObjectType value seen by extensions and event triggers?

<sub>fixable, level 2, ABI / extension API; messages [#5](https://www.postgresql.org/message-id/F4161B2A-79DC-453A-BCE1-4A22139B0664%40yandex-team.ru), [#7](https://www.postgresql.org/message-id/CAEze2WhuqrHBhUkt_QNYTq1YajV_unu3NVJKnOvsjizaLDo%3DMA%40mail.gmail.com), [#9](https://www.postgresql.org/message-id/CAEze2WhgWxZx_1XyHE75CbxeTLWWwPUEq%3D112S-5TWaH5wMAkw%40mail.gmail.com); `maintenance-fit-1`</sub>

### 3. Catalog-based bug fix leaves supported branches and their existing indexes uncovered

A user on a supported back branch has an index on f(t) and runs ALTER TABLE t ADD COLUMN c int. The index is still not rebuilt, so index scans keep returning wrong rows. The user finds out from inconsistent query results, possibly long after the DDL.

The entry is tagged Bugfix, but the fix depends on new pg_depend rows and a new ObjectType value in parsenodes.h. Back branches would need the new rows for every existing index and constraint, plus an ABI change. The material says nothing about back branches, and message #2 shows the same bug class reported since 2020. A committer would likely ask whether a separate fix for released branches is planned, such as a catalog scan during shape-changing DDL.

**Question:** What fix, if any, do supported back branches get, and how does it handle indexes created before the fix?

<sub>fixable, level 2, backpatchability; messages [#1](https://www.postgresql.org/message-id/CAEze2WjDaDyvztdXh3Cb2J%3D11CMVRp4NWAW0E6dBRKa1T9w5ag%40mail.gmail.com), [#2](https://www.postgresql.org/message-id/BEE8820C-50BD-467A-BDBC-F95827D737EB%40yandex-team.ru); `replication-ondisk-1`</sub>

## Questions for the author

1. Which code in core reads refobjsubid, and has each place been checked for WholeRowAttrNumber?
2. Why does the patch use a sentinel subid instead of another marker in pg_depend, such as a distinct deptype?
3. Why can the whole-row pseudo-subobject not go through the existing column address path, without a new ObjectType value?
4. What fix, if any, do supported back branches get, and how does it cover indexes created before the fix?
5. Will a patch that rejects whole-row partition keys ship with this one, or can mispartitioned rows remain after it?

## Touch points

- **on-disk format / pg_upgrade:** Existing indexes and constraints get the new pg_depend rows only when pg_upgrade restores the schema.
- **catalog changes:** The patch adds pg_depend rows with refobjsubid = WholeRowAttrNumber (-32768).
- **ABI / extension API:** The patch adds OBJECT_WHOLE_ROW to the ObjectType enum in parsenodes.h, visible to extensions and event triggers.
- **security / privileges:** v5 touches aclchk.c and seclabel.c to handle OBJECT_WHOLE_ROW.
- **backpatchability:** The entry is tagged Bugfix, but the fix needs new pg_depend rows and a new enum value.
- **maintenance burden ("who maintains this in 10 years"):** Every reader of refobjsubid and every ObjectType-wide feature must now handle the whole-row case.

## Who would commit it, and when

- Owner: silent. Someone who knows pg_depend, the object address functions and the ALTER TABLE rebuild passes in tablecmds.c would have to own it.
- Priority: proceed; audience niche; review cost medium. Few users have whole-row expression indexes or constraints, but for them the bug is silent index corruption with reports since 2020. The patch is about 37 KiB across dependency.c, objectaddress.c and tablecmds.c, and adds a lasting catalog convention.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 4 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

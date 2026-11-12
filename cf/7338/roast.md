# Design pre-review: Use C11 alignas instead of palloc/malloc for alignment

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7338](https://commitfest.postgresql.org/patch/7338/); [thread](https://www.postgresql.org/message-id/flat/94a128da-bf3e-46bd-9e2d-609573c484da%40eisentraut.org), 11 of 11 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). No surviving objection questions the goal, and none is fatal. Three fixable points remain. standards-compat-2 asks whether the rename in 0001 is worth a compile break for the out-of-tree users in message #8. standards-compat-1 shows that 0003 reuses the old name with 8-byte alignment. maintenance-fit-1 notes that the per-record memset in 0005 has no timing.

Not checked: Without the code, this review could not check that the memset in 0005 covers all padding, that the new assertion matches the HEADER_SCRATCH_SIZE calculation, or how much stack each converted call site uses. Only 3 of 11 messages were read directly.

## Prior attempts

None found before 2026-09-08.

## Objections a reviewer is likely to raise

### 1. Renaming PGAlignedXLogBlock breaks known extensions for naming consistency only

Maintainers of pg_tde, pg_flashback, WAL-DIFF and similar tools build against the next major release. Compilation fails on PGAlignedXLogBlock, and they must add version-conditional code although nothing in their behaviour changes.

Message #8 lists at least seven out-of-tree projects that use the type for plain WAL page buffers. Patch 0001 gives them no functional gain; its stated reason is symmetry with PGAlignedBlock and PGIOAlignedBlock. A compatibility alias would keep them building, but it would also block the later MAXALIGNed reuse of the name. The proposal does not weigh these options. A committer would likely ask whether the naming symmetry is worth a break that the project chooses on purpose.

**Question:** Why is a rename without a compatibility alias better for the projects in message #8 than keeping the old name and adding a separate name for a MAXALIGNed block?

<sub>fixable, level 1b, ABI / extension API; messages [#7](https://www.postgresql.org/message-id/9e81a344-b927-41e2-acea-da444a03aaa0%40eisentraut.org), [#8](https://www.postgresql.org/message-id/CA%2BbCEdCnFaAhpfzYRh4cWCRUYCxMjxCM%2BV3_KXau_1pYhbDTbQ%40mail.gmail.com); `standards-compat-2`</sub>

### 2. Patch 0003 reuses an existing type name with weaker alignment

An extension declares PGAlignedXLogBlock and reads WAL with O_DIRECT. After upgrade it still compiles, but the buffer is only 8-byte aligned. pread fails with EINVAL on ext4 and works on btrfs, so the failure shows up only on some customer systems.

Message #8 measured this: with 0001-0005, alignof(PGAlignedXLogBlock) drops from 4096 to 8, and an O_DIRECT read fails on ext4 but succeeds on btrfs. Message #7 already doubts reusing the name without a gap, and 0003 is still marked XXX. The same type name gives a different contract with no compile-time signal. Manuel found no direct-I/O user, but a GitHub search on default branches is only a lower bound.

**Question:** If 0003 goes in, how many releases will separate the rename from the reintroduced name, and what tells a direct-I/O user that the alignment changed?

<sub>fixable, level 2, ABI / extension API; messages [#7](https://www.postgresql.org/message-id/9e81a344-b927-41e2-acea-da444a03aaa0%40eisentraut.org), [#8](https://www.postgresql.org/message-id/CA%2BbCEdCnFaAhpfzYRh4cWCRUYCxMjxCM%2BV3_KXau_1pYhbDTbQ%40mail.gmail.com); `standards-compat-1`</sub>

### 3. XLogInsert() now zeroes a 952-byte workspace on every WAL record

Every backend that writes WAL now zeroes a 952-byte stack workspace in XLogInsert() for each record, where before a static buffer was zeroed once. A small-record, high-insert workload pays this cost on every call, and nobody has measured it. In _mdfd_getseg(), each call also fills an 8 kB const zero buffer, and the stack frame grows to about 20 kB.

Patch 0005 adds the memset because the static variable used to be zero-initialized for free. Message #8 shows the memset is needed, since without it 16033 WAL records carry uninitialized padding bytes. No message measures the cost on the XLogInsert() path, which every WAL-writing user takes. Message #11 shows the _mdfd_getseg() and dsm_impl_mmap() const locals copy or clear 8 kB per call and suggests static const; v4 does not take that up.

**Question:** What does a pgbench or small-insert benchmark show for v4 0005 against master, and will the 8 kB zero buffers become static const?

<sub>fixable, level 2, performance on the hot path; messages [#7](https://www.postgresql.org/message-id/9e81a344-b927-41e2-acea-da444a03aaa0%40eisentraut.org), [#8](https://www.postgresql.org/message-id/CA%2BbCEdCnFaAhpfzYRh4cWCRUYCxMjxCM%2BV3_KXau_1pYhbDTbQ%40mail.gmail.com), [#9](https://www.postgresql.org/message-id/8bdc9593-cbe1-4c61-abe0-732cc8724acb%40eisentraut.org), [#11](https://www.postgresql.org/message-id/CABkqMMu-DpTsAnmwwmuFNuqeYmrAPfYZS8WjJtEhbPSdNVJ5Bg%40mail.gmail.com); `maintenance-fit-1`</sub>

## Questions for the author

1. Will 0003 be part of the commit, and if so, how many major releases will separate the rename in 0001 from the reuse of the name PGAlignedXLogBlock?
2. Why is renaming PGAlignedXLogBlock without a compatibility alias better than keeping the old name and choosing a new name for a MAXALIGNed block?
3. What does a small-record insert benchmark show for v4 with 0005 against master?
4. Will the zero buffers in _mdfd_getseg() and dsm_impl_mmap() become static const, as message #11 suggests?
5. Which call sites does 0002 convert, and how much stack does each one add?

## Touch points

- **WAL / replication / standby:** Patches 0004 and 0005 change how xloginsert.c builds the WAL record header; 0002 changes buffers in xlogreader.c.
- **ABI / extension API:** 0001 renames PGAlignedXLogBlock in c.h, 0003 reuses the name with weaker alignment, and the XLogReader struct in xlogreader.h changes.
- **backpatchability:** With 0003, PGAlignedXLogBlock means a different alignment in new and old branches, so back-patched code using the name behaves differently.
- **maintenance burden ("who maintains this in 10 years"):** The pgindent workaround for alignas and the HEADER_SCRATCH_SIZE assertion are small new pieces to keep correct.
- **performance on the hot path:** 0005 zeroes a 952-byte local workspace in XLogInsert() for every WAL record.

## Who would commit it, and when

- Owner: qualified committer engaged. The author is listed as Peter Eisentraut (petere), and Heikki Linnakangas reviewed the WAL insert part in message #2. The owner needs to know the WAL insert path and the direct I/O alignment rules.
- Priority: proceed; audience niche; review cost medium. This is internal refactoring that helps core developers and the multi-threading work, and it adds no user-visible surface apart from the type rename. The patches are small, but 0005 changes the WAL insert path, which needs a careful reader and a timing.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 3 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

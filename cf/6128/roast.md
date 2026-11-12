# Design pre-review: GIN pageinspect support for entry tree and non-leaf posting tree

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6128](https://commitfest.postgresql.org/patch/6128/); [thread](https://www.postgresql.org/message-id/flat/CALdSSPiN13n7feQcY0WCmq8jzxjwqhNrt1E%3Dg%3Dg6aZANyE_OoQ%40mail.gmail.com), 46 of 46 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No objection survived at levels 1a or 1b: the problem is stated, the cost stays inside pageinspect, and the page-level shape answers the concern recorded in P1. Two fixable level 2 points remain, standards-compat-1 and standards-compat-2, both about output columns that pageinspect 1.14 would fix as a user-facing contract. A committer would likely want them settled before commit, but neither requires a different proposal.

Not checked: The code was not read, so safety on corrupted pages, the privilege setup in the 1.13--1.14 script and the actual NULL category output could not be checked. Only 3 of 46 messages were visible, so some open points may have been answered in the thread.

## Prior attempts

- [Pageinspect - add functions on GIN and GiST indexes from gevel](https://www.postgresql.org/message-id/flat/accae316-5e4d-8963-0c3d-277ef13c396c@postgrespro.ru) (similar idea; Returned with feedback, 2017-11-27). "The proposed patch uses a very different approach, reading the whole index, not individual pages. Why should it be part of pageinspect?" ([message](https://www.postgresql.org/message-id/2514c060-717e-6fb9-981c-bf1344174da5%402ndquadrant.com)) Different now: P1 moved gevel's whole-index functions into pageinspect. This proposal decodes one raw page per call through get_raw_page, which is the page-level shape the P1 reviewers asked for.
- [Allow pageinspect's bt_page_stats function to return a set of rows instead of a single row](https://www.postgresql.org/message-id/flat/CANugjht-=oGMRmNJKMqnBC69y7vr+wHDmm0ZK6-1pJsxoBKBbA@mail.gmail.com) (related; Returned with feedback, 2022-10-12). "This feedback has not been addressed and the thread is idle four weeks, so I have marked this CF entry as RwF." ([message](https://www.postgresql.org/message-id/Y0ZV%2BV%2FkDk4ygdPm%40paquier.xyz)) Different now: P5 stalled partly on a layout-dependent test that failed on 32-bit. This patch ships a 32-bit expected file (gin_1.out), and the thread accepts the 8 KiB block size dependency.

## Objections a reviewer is likely to raise

### 1. Output columns encode on-disk details that later fixes cannot change

A user reads downlink (2147483696,3) on an entry leaf page as a block number and goes looking for block 2147483696. A script that compares output across machines gets a different value on 32-bit builds. Once pageinspect 1.14 ships, changing these columns needs another extension version and an upgrade script.

Message #1 says the leaf downlink has a different meaning from the internal one but is shown the same way, and message #10 confirms it varies by platform. The column named downlink is a tid in gin_entrypage_items but an integer in gin_datapage_items. gin_entrypage_items takes reloid oid, while gist_page_items takes regclass. The doc hunk shows examples only and gives no column definitions.

**Question:** Can leaf tuples report a decoded posting list offset and item count, or a posting tree root block, instead of the raw tid, and why does downlink differ in type between the two functions?

<sub>fixable, level 2, SQL-standard conflict or non-standard syntax; messages [#1](https://www.postgresql.org/message-id/CALdSSPiN13n7feQcY0WCmq8jzxjwqhNrt1E%3Dg%3Dg6aZANyE_OoQ%40mail.gmail.com), [#10](https://www.postgresql.org/message-id/CALdSSPgxaLX-%3DyXm-mVF47S7fYJAaEoqycXw0G3yLS5wH%3D019w%40mail.gmail.com), [#11](https://www.postgresql.org/message-id/CALdSSPjX0rpG%2BWDf8CFE_jZGFOAFvx5uoihjPrXTpjJJvMfGeA%40mail.gmail.com); `standards-compat-1`</sub>

### 2. keys column format and NULL categories are not defined

A user repairing a corrupted index sees y=NULL and cannot tell a NULL array element from a NULL array or an empty array, which GIN stores as different key categories. Tools that parse keys from gist_page_items cannot reuse that code, because the format differs.

GIN stores separate null categories, and message #35 shows only the planned output y=NULL. The material does not say whether the keys column keeps the categories apart. gist_page_items prints keys in the index value description style, and this proposal adds a new attname=value text form without saying why.

**Question:** How does the keys column show each GIN null category and multi-column keys, and why not use the key format gist_page_items already prints?

<sub>fixable, level 2, SQL-standard conflict or non-standard syntax; messages [#29](https://www.postgresql.org/message-id/0A0E3993-7617-48EA-8555-6C1907C147E5%40yandex-team.ru), [#35](https://www.postgresql.org/message-id/CALdSSPjgB2%3Dcn2KsMSYD%3D55ft1kWv0xThNT7%2BeDHmPtGOuKo5w%40mail.gmail.com); `standards-compat-2`</sub>

## Questions for the author

1. Can gin_entrypage_items report the posting list offset, item count and posting tree root block for leaf tuples as decoded values instead of a raw tid such as (2147483696,3)?
2. Why is the downlink column a tid in gin_entrypage_items and an integer in gin_datapage_items?
3. How does the keys column show each GIN null category (null key, empty item, null item), and can a user tell them apart?
4. Will the documentation define each output column, including what tids and downlink mean on leaf pages versus internal pages?
5. Has anyone run both functions on deliberately corrupted pages, given that corruption repair is a stated use?

## Touch points

- **ABI / extension API:** pageinspect moves from 1.13 to 1.14 with two new SQL functions, and patch 0003 moves the IS_INDEX macro to pageinspect.h.
- **SQL-standard conflict or non-standard syntax:** see objection standards-compat-1: Output columns encode on-disk details that later fixes cannot change
- **security / privileges:** The functions decode raw pages and run type output functions on keys; the thread does not show whether the 1.13--1.14 script revokes execute from PUBLIC.
- **maintenance burden ("who maintains this in 10 years"):** The output columns become a contract, and the tests need a 32-bit expected file and assume the 8 KiB block size.

## Who would commit it, and when

- Owner: silent. Someone who knows the GIN page layout and pageinspect conventions would have to own it. Peter Eisentraut is credited as co-author in the commit message, but the brief says no committer has replied to the latest version.
- Priority: needs champion; audience niche; review cost medium. The patch fills a gap in pageinspect's GIN coverage for people who study or repair GIN indexes, and the cost stays in one contrib module. It reached Ready for Committer after five reviewers and is now waiting for a committer, with two output-shape points still open.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 1 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

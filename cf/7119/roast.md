# Design pre-review: Support specialized B-tree page searches

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7119](https://commitfest.postgresql.org/patch/7119/); [thread](https://www.postgresql.org/message-id/flat/79FF16D2-B815-45C9-A24A-D0851981E78B%40yandex-team.ru), 3 of 3 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The goal is stated and the benchmarks are detailed, but the design of support procedure 7 is not settled. The int4 callback copies nbtree search rules (maintenance-fit-1), and the two documentation hunks describe different contracts (standards-compat-1). The thread does not measure the existing SortSupport comparator as an alternative (already-exists-1). Message #3 proposes a different split and has no answer, so the list has to agree on the callback contract before code review makes sense.

Not checked: Without the code, nobody can check that the interpolation probe stays inside the page's line pointer range on a corrupted page with unsorted keys. The cost of the new path for opclasses without support procedure 7 is also unchecked.

## Prior attempts

- [Delaying/avoiding BTreeTupleGetNAtts() call within _bt_compare()](https://www.postgresql.org/message-id/flat/CAH2-Wzngz5MrkiTaZNny0GzfTxNQE+QWPPaO-C390BgruhgjEw@mail.gmail.com) (similar idea; Withdrawn, 2020-11-09). The visible messages do not show why. Andres Freund suggested loading ntupatts where the code loads it today and changing only the loop termination, because the delayed load and extra branches could add stalls. Peter Geoghegan agreed to rework the patch that way, and the thread has no later message before the November 2020 withdrawal. (AI summary of the thread) Different now: P4 cut per-call overhead inside the generic _bt_compare(). This proposal skips _bt_compare() for int4 through an opclass callback that reads the key directly and may add an interpolation probe.
- [_bt_binsrch* improvements - equal-prefix-skip binary search](https://www.postgresql.org/message-id/flat/CAEze2WjnrkzT9tfqFhSCPeH6LV-UUj5_oO5vX038dvfK7fDuyQ@mail.gmail.com) (similar idea; Withdrawn, 2020-09-15). reason not recorded Different now: P15 changed the generic _bt_binsrch loop to skip equal prefixes. This proposal keeps the generic loop and adds an opt-in per-opclass search for single-column, same-type keys.

## Objections a reviewer is likely to raise

### 1. Search control flow copied into per-opclass callbacks in nbtcompare.c

A later patch changes how nbtree handles posting lists, pivot tuples or heap TID tie-breaking in nbtsearch.c. The int4 callback, and the int8 one proposed in message #3, keeps the old rules, and no test fails. Users of specialized opclasses get wrong search bounds only on pages that hit the changed case.

The commit message of 0001 says the int4 callback itself preserves NULLs, descending order, pivot tuples, heap TID tie breakers and posting lists. Message #3 confirms that nbtcompare.c duplicates the binary search loop, slot probing, TID tie-breaking and posting list traversal. Each added type then copies nbtree search rules again. Message #3 proposes keeping the loop in nbtsearch.c, but the author has not replied and no new version exists.

**Question:** Will the next version keep all search control flow in nbtsearch.c, so that a type callback holds only value extraction, comparison and interpolation math?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#1](https://www.postgresql.org/message-id/79FF16D2-B815-45C9-A24A-D0851981E78B%40yandex-team.ru), [#3](https://www.postgresql.org/message-id/CAAU0Mh3YyyRjSopS26N7uX4BS8tEdRMmEJ063w4%3DNNYiGdVM4g%40mail.gmail.com); `maintenance-fit-1`</sub>

### 2. Support procedure 7 contract is not settled before it becomes permanent

An extension author follows xindex.sgml and writes a support function 7 that returns a whole-page search function. A later release adopts the primitive-callback split from message #3, and the extension opclass then fails to compile or gives wrong search results. The author finds out when users upgrade.

Support procedure numbers and their signatures become a documented, catalog-visible contract in pg_amproc and in CREATE OPERATOR CLASS. The docs disagree on what that contract is: xindex.sgml says the function returns the address of one search function, while btree.sgml says it installs callbacks. The name binsearchsupport names one algorithm, but patch 0002 adds interpolation. Message #3 proposes a different callback split on the model of sortsupport and skipsupport, and the author has not answered it.

**Question:** Which callback struct will support procedure 7 fill, does it follow the sortsupport and skipsupport pattern, and is it meant for third-party opclasses?

<sub>fixable, level 2, ABI / extension API; messages [#1](https://www.postgresql.org/message-id/79FF16D2-B815-45C9-A24A-D0851981E78B%40yandex-team.ru), [#3](https://www.postgresql.org/message-id/CAAU0Mh3YyyRjSopS26N7uX4BS8tEdRMmEJ063w4%3DNNYiGdVM4g%40mail.gmail.com); `standards-compat-1`</sub>

### 3. Procedure 7 adds a second fast-comparator mechanism next to SortSupport

An opclass author who wants faster index searches for int8, date or timestamp must write new procedure 7 callbacks, even though the opclass already ships an fmgr-free comparator through support procedure 2. Each new type adds about 70 lines of search callbacks. A fix to ordering rules in one mechanism can be missed in the other.

btint4sortsupport and btint8sortsupport already give fmgr-free comparators that handle reverse order and NULL placement, so the specialization part of the win could come from calling the existing SortSupport comparator in _bt_compare, plus fast extraction of a fixed-width first key. Only the interpolation in the second patch needs type-specific knowledge. Message #3 shows the current split duplicates search control flow and needs new per-type code for int8, and it asks for the SortSupport model. The author has not answered message #3.

**Question:** What share of the 3-12% specialization gain remains if nbtree calls the existing SortSupport comparator, and what does procedure 7 add beyond the interpolation hook?

<sub>fixable, level 2, design; messages [#1](https://www.postgresql.org/message-id/79FF16D2-B815-45C9-A24A-D0851981E78B%40yandex-team.ru), [#2](https://www.postgresql.org/message-id/2F19CB19-5FDE-4B06-9CEF-6D8075B52B21%40yandex-team.ru), [#3](https://www.postgresql.org/message-id/CAAU0Mh3YyyRjSopS26N7uX4BS8tEdRMmEJ063w4%3DNNYiGdVM4g%40mail.gmail.com); `already-exists-1`</sub>

### 4. Support procedure 7 exposes nbtree page internals as an extension API

An extension author implements support function 7 for a custom fixed-width type, following btree.sgml. A later release changes the index tuple or posting list layout. The extension's page search compiles and runs but reads keys wrongly, and users see wrong query results after upgrading.

The documentation offers support function 7 to any operator family, and its callbacks read keys directly and search a whole page. That makes the nbtree tuple layout part of a contract with code outside core. The material does not say whether third-party opclasses are expected to use it, or how stable the callback struct is. In [P6], Tom Lane rejected a change because it gave up page layout flexibility without a compelling gain, and the same cost applies here.

Precedent: [Preventing indirection for IndexPageGetOpaque for known-size page special areas](https://www.postgresql.org/message-id/flat/CAEze2WjE3+tGO9Fs9+iZMU+z6mMZKo54W1Zt98WKqbEUHbHOBg@mail.gmail.com) (Rejected, 2022-11-30)

**Question:** Is support function 7 meant for third-party opclasses, and if so, which nbtree layout details does the callback API promise to keep stable?

<sub>fixable, level 2, ABI / extension API; messages [#1](https://www.postgresql.org/message-id/79FF16D2-B815-45C9-A24A-D0851981E78B%40yandex-team.ru), [#2](https://www.postgresql.org/message-id/2F19CB19-5FDE-4B06-9CEF-6D8075B52B21%40yandex-team.ru); `maintenance-fit-2`</sub>

### 5. End-to-end gain is small outside server-side microbenchmarks

The project adds support procedure 7, a new C callback API and a second page-search path in nbtree. Users who run ordinary OLTP queries see about 3% or nothing, and the 16-client duplicate case regresses. Future nbtree changes must keep both search paths correct for years.

The 35% to 47% figures come from loops of one million server-side lookups. Message #1 reports that standard pgbench -S showed no measurable difference, and message #2 reports about 2.5% to 3% for regular point lookups. The thread has no field report or production profile where in-page search dominates. The benefit reaches all int4 users, but its size in real workloads is not yet shown against the cost of a second search path.

**Question:** Which real workloads, such as nested loop joins, large IN lists or random inserts, gain end to end, with profiles showing page search as a real cost?

<sub>fixable, level 1b, demand; messages [#1](https://www.postgresql.org/message-id/79FF16D2-B815-45C9-A24A-D0851981E78B%40yandex-team.ru), [#2](https://www.postgresql.org/message-id/2F19CB19-5FDE-4B06-9CEF-6D8075B52B21%40yandex-team.ru); `demand-1`</sub>

## Questions for the author

1. Will the next version keep all search control flow in nbtsearch.c, as message #3 proposes, so that a type callback holds only value extraction, comparison and interpolation math?
2. Which callback struct does support procedure 7 fill, and does it return one search function (xindex.sgml) or install several callbacks (btree.sgml)?
3. What share of the 3% to 12% specialization gain remains if nbtree calls the existing SortSupport comparator and extracts the fixed-width key directly?
4. Is support procedure 7 meant for third-party opclasses, and if so, which nbtree tuple layout details does the callback API promise to keep stable?
5. What causes the 1.3% regression for duplicate lookups with 16 clients, and which real workload shows a clear end-to-end gain with a profile?

## Touch points

- **on-disk format / pg_upgrade:** see objection maintenance-fit-2: Support procedure 7 exposes nbtree page internals as an extension API
- **catalog changes:** The patch adds a pg_amproc row for support procedure 7 and a pg_proc entry for btint4binsearchsupport.
- **ABI / extension API:** nbtree.h gains a C callback API that opclasses use to compare keys and search a page interval.
- **maintenance burden ("who maintains this in 10 years"):** nbtree gets a second page-search path, and the int4 callback repeats pivot, heap TID and posting list rules.
- **performance on the hot path:** The patch changes the page search in nbtsearch.c; duplicate lookups with 16 clients regressed by about 1.3%.

## Who would commit it, and when

- Owner: silent. It needs a committer who knows nbtree page search, posting lists and heap TID tie-breaking; none has spoken in the visible thread.
- Priority: fix design first; audience most users; review cost high. int4 B-tree indexes are common, and the specialization gain of 3% to 12% holds across the tested distributions. The callback contract and the split between nbtsearch.c and nbtcompare.c are still open, and a review of nbtree search correctness is costly.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

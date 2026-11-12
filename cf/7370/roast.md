# Design pre-review: BUG #19705: One NaN box makes BRIN and GiST box indexes omit unrelated rows

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7370](https://commitfest.postgresql.org/patch/7370/); [thread](https://www.postgresql.org/message-id/flat/19705-548fda77321e062d%40postgresql.org), 16 of 16 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). The bug is a reported wrong-results case with reproductions, and no objection survived at levels 1a or 1b. The one surviving objection, maintenance-fit-2, is a fixable design choice: master would carry box_mergeable while back branches carry the self-containment check. The author already offers to use one check everywhere, so a committer who knows BRIN can settle it.

Not checked: Without the code, the claims that old NaN summaries and internal keys are handled without REINDEX, and that the self-containment check catches every poisoned range, could not be checked. Only the first 3 of 16 messages were read directly.

## Prior attempts

- [Fix behavior of geo_ops when NaN is involved](https://www.postgresql.org/message-id/flat/CAGf+fX70rWFOk5cd00uMfa__0yP+vtQg5ck7c2Onb-Yczp0URA@mail.gmail.com) (related; Returned with feedback, 2022-08-01). "Rather than code changes, what this patch needs is more community interest." ([message](https://www.postgresql.org/message-id/f84318e2-2918-8c21-436f-a79823c9e097%40timescale.com)) Different now: P1 changed what geo_ops operators return for NaN. This proposal leaves the operators alone and fixes BRIN and GiST so index scans match the heap. It also has three reviewers who tested it, which P1 lacked.

## Objections a reviewer is likely to raise

### 1. Two different BRIN fixes for master and back branches

A committer later back-patches a fix in brin_inclusion_consistent() or brin_inclusion_add_value(). Master has the box_mergeable path and the back branches have the self-containment check, so the patch conflicts. A hand-merged version on 14 to 18 then gets no testing against the master code.

Message #7 places box_mergeable plus a consistent-side guard on master, and a separate self-containment check on 14 to 18. Messages #7, #14 and #16 say the self-containment check also works on master and needs no catversion bump. Master would then keep a pg_amproc row it does not need for the old-summary case. If master later bans NaN as message #15 suggests, the extra support function stays forever.

**Question:** What does box_mergeable give on master that the catalog-free self-containment check does not, enough to justify carrying two BRIN fixes?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#7](https://www.postgresql.org/message-id/CAGRkXqQ7q0U3jmitwOjPY2BNBkxWsnyahgoYyMs4H%3DhwVtPFgw%40mail.gmail.com), [#14](https://www.postgresql.org/message-id/CAGRkXqSQHEe0R5EuuLYthMf1C%3DL_tvPD74aPbgtuOT0Rg0Y99w%40mail.gmail.com), [#15](https://www.postgresql.org/message-id/CANWCAZZmuP9xsDv%2Bk6WrKAfQK-_6pzRDF4XrPQHuQxEj-qSSyA%40mail.gmail.com), [#16](https://www.postgresql.org/message-id/CAGRkXqRbimqBSg9GYMGDSp7bpaDWed91Tyb6WHcr4Yw50bU8Rg%40mail.gmail.com); `maintenance-fit-2`</sub>

## Questions for the author

1. What does box_mergeable give on master that the catalog-free self-containment check does not, enough to justify two different BRIN fixes?
2. What is the measured cost of the added NaN checks in GiST search and KNN paths for box indexes that hold no NaN?
3. Do the rebased patches for 14 to 16 exist, and do they pass the same differential check of seq scan against index scan?
4. What is the plan for the SP-GiST box_ops and poly_ops mismatches that this set leaves unfixed?
5. What does the Tom Lane message that John Naylor linked propose for master, and how does this set fit with it?

## Touch points

- **on-disk format / pg_upgrade:** New GiST box keys store Infinity where NaN appeared, and old NaN summaries and keys must still be read correctly after pg_upgrade.
- **catalog changes:** Master adds a pg_amproc row for box_inclusion_ops (support function 12, box_mergeable) with a catversion bump.
- **backpatchability:** The GiST patches and a catalog-free BRIN fix are planned for back-patch to 14, with a different BRIN fix on master.
- **maintenance burden ("who maintains this in 10 years"):** Two BRIN fixes by branch, plus NaN checks in several GiST consistent, union and KNN code paths.
- **performance on the hot path:** Extra NaN checks run in GiST search and KNN distance paths, and the thread gives no measurement.

## Who would commit it, and when

- Owner: silent. Needs a committer who knows the BRIN inclusion framework and the GiST geometric opclasses. The author asked Tomas and Heikki to commit, and neither has answered in the visible thread.
- Priority: needs champion; audience niche; review cost medium. Few tables hold NaN boxes, but affected users get silently wrong results, and three reviewers have tested the set. It waits on a committer to choose the master BRIN fix; review cost is estimated because no code was read.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 4 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

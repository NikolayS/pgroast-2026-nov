# Design pre-review: Remove invalid SS2/SS3 handling from EUC-KR routines

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6761](https://commitfest.postgresql.org/patch/6761/); [thread](https://www.postgresql.org/message-id/flat/CAE%2BcgNgTWvCT2%2BHZYRzQA8-wSQrj-FjPQQNffn%3D_3DOpz0pKgA%40mail.gmail.com), 10 of 10 messages read; no code read.

## Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium). No objection survived the critic. The only one raised, demand-1, was refuted: the new width estimates match the true 2-byte maximum of EUC_KR, and the effects reach only EUC_KR databases on master. The patch aligns mblen, dsplen and maxmblen with KS X 2901 and with pg_euckr_verifychar(). Confidence is medium because no one has replied to v4 and two points could not be checked without the code.

Not checked: Without the code, two points could not be checked: whether pg_upgrade keeps existing TOAST tables for EUC_KR tables under the new size rule, and whether each reader of mblen and maxmblen is safe with unverified 0x8F bytes.

## Prior attempts

None found before 2026-05-12.

## Objections a reviewer is likely to raise

No objection held up in the counter-check.

## Questions for the author

1. Does pg_upgrade in binary-upgrade mode still create the TOAST table for an EUC_KR table that had one in the old cluster but would not get one under maxmblen = 2?
2. How do the new routines split unverified text containing 0x8F that is already on disk, such as shared catalog names written from a database with another encoding?
3. Does any reader of mblen or maxmblen for EUC_KR still assume a 3-byte maximum after the change?
4. Were the checks P1 to P5 in Henson Choi's review list completed against v4?
5. Is src/test/locale/ affected by the change, as Michael Paquier asked in message #3?

## Touch points

- **on-disk format / pg_upgrade:** New tables with bounded-width columns may no longer get a TOAST table, and unverified 0x8F text on disk splits differently.
- **ABI / extension API:** pg_wchar_table[PG_EUC_KR].maxmblen in src/common/wchar.c drops from 3 to 2, which extensions and frontend code can read.
- **planner behaviour change:** Width estimates for char(n) and varchar(n) shrink in EUC_KR databases, for example from 34 to 24 for char(10).
- **pg_dump / restore round-trip:** A restored table in an EUC_KR database may get no TOAST table where the old cluster had one.
- **security / privileges:** The patch changes character length and buffer sizing rules in wchar.c, an area with a recent CVE fix for EUC_CN.
- **maintenance burden ("who maintains this in 10 years"):** Three EUC-KR-specific routines replace calls to the shared pg_euc_* helpers, in the same shape as UHC.

## Who would commit it, and when

- Owner: qualified committer engaged. John Naylor reviewed the patch, showed the plan width changes and set the status; Michael Paquier asked for tests. The owner needs to know the multibyte encoding code in wchar.c.
- Priority: proceed; audience niche; review cost low. The change affects only EUC_KR databases, adds no new mechanism, and the 0002 patch is about 6.5 KB in wchar.c plus docs and tests. v4 answers the reviewers' requests and waits for a reply.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 1 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

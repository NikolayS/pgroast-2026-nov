# Design pre-review: pgcrypto crypt() can return text with invalid encoding

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7417](https://commitfest.postgresql.org/patch/7417/); [thread](https://www.postgresql.org/message-id/flat/19600-e5fb479f50f5022d%40postgresql.org), 3 of 3 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). The goal and the design copy what pgp_*_decrypt_text already do, and no objection survived at levels 1a or 1b. The one surviving objection, standards-compat-1, is a fixable back-branch question: crypt(pw, stored_hash) starts to raise an error for stored hashes that hold invalid encoding. A committer who knows pgcrypto has to decide between master only and back branches.

Not checked: The code was not read. The claim that the check reads only the finished result, and so leaves valid hashes unchanged, rests on message #3; the back branches were checked for applicability but not built.

## Prior attempts

- [Encoding protection for pgcrypto](https://www.postgresql.org/message-id/flat/CAGRkXqRZyo0gLxPJqUsDqtWYBbgM14betsHiLRPj9mo2=z9VvA@mail.gmail.com) (similar idea; Returned with feedback, 2024-10-07). The visible messages do not show why. Cary Huang reported failing tests and questioned aborting the transaction instead of issuing a warning. The author posted v2 and argued that callers who want no encoding check can use the bytea variants, and no later message follows. (AI summary of the thread) Different now: P2 added pg_verifymbstr() to pgp_sym_decrypt_text and pgp_pub_decrypt_text; this proposal adds the same check to crypt(), which has no bytea variant. Message #3 reports that this check later went in as d536aee5566.

## Objections a reviewer is likely to raise

### 1. Back-patching turns working password checks into errors after minor upgrade

A site stored DES or MD5 hashes made with a hand-written non-ASCII salt. After a minor upgrade, `where pwhash = crypt($1, pwhash)` raises an error instead of returning false, so those users cannot log in. Any query that evaluates crypt() over several rows fails as a whole if one row holds such a hash.

Message #3 says these hashes stop working for the right password and for a wrong one, and that crypt() has no bytea variant to fall back on. Message #3 also checked that the patch applies to REL_14 through REL_19, but the thread has not decided whether to back-patch. Committers usually keep minor releases free of changes that make working queries fail. The proposal also gives no way to find or rehash affected rows before upgrading, and the author has not replied to the release-note request.

**Question:** Will this go to master only, and if back-patched, what query lets users find affected stored hashes before the upgrade, and how do they rehash them?

<sub>fixable, level 2, backpatchability; messages [#2](https://www.postgresql.org/message-id/CAB8bMivmoRuUcNF1pEFx%3DTr2Q_b31VwtJse4P22pTMgoG_uC1w%40mail.gmail.com), [#3](https://www.postgresql.org/message-id/CA%2BY2BmeT0uQBP5ewhSqxTOAx6COr0uEYuLpiBfeyFRxD5e3ZDw%40mail.gmail.com); `standards-compat-1`</sub>

## Questions for the author

1. Should this fix go to master only, or also to REL_14 through REL_19, where existing stored hashes start to raise errors after a minor upgrade?
2. What query lets users find stored crypt() hashes that hold invalid encoding before they upgrade?
3. How should a user rehash an affected row, given that crypt(pw, stored_hash) now raises an error even for the right password?
4. Was rejecting non-ASCII salts up front for DES, xDES and MD5 considered, as SHA-crypt and Blowfish already do?
5. Will the next version add the xDES and MD5 test cases and the release-note text that message #3 asked for?

## Touch points

- **on-disk format / pg_upgrade:** No format change, but stored hashes with invalid encoding survive pg_upgrade and then make crypt() raise an error.
- **SQL-standard conflict or non-standard syntax:** see objection standards-compat-1: Back-patching turns working password checks into errors after minor upgrade
- **pg_dump / restore round-trip:** The fix stops crypt() from producing text that COPY cannot reload; rows already stored stay unrestorable.
- **security / privileges:** crypt() is executable by PUBLIC, so any user can store invalid text today; the patch closes that path.
- **backpatchability:** Message #3 checked REL_14 through REL_19; a back-patch makes existing password checks raise an error after a minor upgrade.
- **maintenance burden ("who maintains this in 10 years"):** Small: one pg_verifymbstr() call in pg_crypt() and one regression test with an alternative expected file for non-UTF8 databases.

## Who would commit it, and when

- Owner: silent. No committer has spoken in the thread. It needs someone who knows pgcrypto and the back-branch policy for behaviour changes.
- Priority: needs champion; audience niche; review cost low. Only callers who pass a hand-written non-ASCII salt to DES, xDES or MD5 crypt() are affected, and the 4.3 KB patch touches pgcrypto only. One reviewer tested it and called it good; what remains is a committer's decision on back branches.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 2 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

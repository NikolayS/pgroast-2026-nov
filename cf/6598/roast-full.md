# Design pre-review: Add ssl_(supported|shared)_groups to sslinfo

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6598](https://commitfest.postgresql.org/patch/6598/) (Needs review); authors: Dmitry Dolgov (erthalion)
- Thread: [Add ssl_(supported|shared)_groups to sslinfo](https://www.postgresql.org/message-id/flat/d57duqvzkxe43oons3jkdq7pj2wacidg7qorxommri74evu3l2%404x53she7mf77), started 2026-02-19
- Messages read: 22 of 22
- Patch prose read: v7-0001-contrib-sslinfo-Add-ssl_group_info.patch, v1-0001-contrib-sslinfo-Add-ssl_-supported-shared-_groups.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

Reviewers agree the negotiated group is worth exposing, and the cost stays inside contrib/sslinfo. The shape of the result is not settled. demand-1 and maintenance-fit-3 show no use case for the shared and client supported rows, and the server's list cannot be fetched. standards-compat-1 and standards-compat-2 show that v7 ships an undocumented helper function and an empty result that has several meanings. Nobody has replied to v7.

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, I could not check what v7 returns for RSA or non-EC DHE key exchange, or whether it still raises an error for an unknown group name. The build checks on LibreSSL and old OpenSSL were also not verified.

## The proposal, restated

**Problem.** The sslinfo extension shows the cipher and protocol version of the current connection but not the TLS group used for key exchange. The author wants this for TLS diagnostics and debugging, for example to see which key share was negotiated, such as X25519MLKEM768 on newer OpenSSL.

**Proposed design.** The patch adds a set-returning function ssl_group_info() to sslinfo version 1.3, with one row per group and a group_type of negotiated, shared or client supported. v7 adds ssl_group_info_with_nulls(), which returns null rows when no group was negotiated, and ssl_group_info() wraps it and filters the nulls out. Build checks in configure and meson detect the OpenSSL group functions, and the function returns no rows when they are missing.

**User-visible surface.**
- New function ssl_group_info() returns setof record, with columns group_type and name.
- New function ssl_group_info_with_nulls(), which v7 adds as the base for ssl_group_info().
- group_type values: negotiated, shared, client supported.
- sslinfo extension version bumped to 1.3 through sslinfo--1.2--1.3.sql.
- New configure and meson checks for OpenSSL group functions in pg_config.h.in.

**What the author claims.**
- The function helps diagnose which key share the client and server negotiate.
- The sslinfo scope has grown beyond certificates before, so adding handshake facts is acceptable if documented.
- Builds whose OpenSSL lacks the group functions, such as 1.1.1w, still compile, and the function returns no rows.
- The implementation follows ssl_print_groups from OpenSSL.

**Thread state.** Gustafsson, Champion, Huang, Parragi and Si reviewed it, and v7 says it addresses the comments, but nobody has replied to v7 yet. Champion and Si doubt the value of the non-negotiated rows, and the naming, the error on unknown group names and the silent skip on old OpenSSL were still open.

## 2. Prior attempts

### Revert "Add notBefore and notAfter to SSL cert info display" (related)

- **Link:** [https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=29a0ccbce97978e5d65b8f96c85a00611bb403c4](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=29a0ccbce97978e5d65b8f96c85a00611bb403c4)
- **Thread:** [https://www.postgresql.org/message-id/flat/182b8565486.10af1a86f158715.2387262617218380588@highgo.ca](https://www.postgresql.org/message-id/flat/182b8565486.10af1a86f158715.2387262617218380588@highgo.ca)
- **Date:** decided 2023-07-20; started 2023-07-20
- **Outcome:** Reverted (revert)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (as recorded in the corpus):** Due to an oversight in reviewing, this used functionality not compatible with old versions of OpenSSL. This reverts commit 75ec5e7bec700577d39d653c316e3ae6c505842c.
- **Reason (quote):** "Due to an oversight in reviewing, this used functionality not compatible with old versions of OpenSSL."
- **What is different this time:** P4 reverted a different sslinfo addition, cert validity dates, because it used OpenSSL functions that old versions lack. This proposal adds configure and meson checks for the group functions.

### Revert "Add notBefore and notAfter to SSL cert info display" (related)

- **Link:** [https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=697f8d266cfb33409f7ccf3319f4448477066329](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=697f8d266cfb33409f7ccf3319f4448477066329)
- **Thread:** [https://www.postgresql.org/message-id/flat/F0DF7102-192D-4C21-96AE-9A01AE153AD1@yesql.se](https://www.postgresql.org/message-id/flat/F0DF7102-192D-4C21-96AE-9A01AE153AD1@yesql.se), [https://www.postgresql.org/message-id/flat/182b8565486.10af1a86f158715.2387262617218380588@highgo.ca](https://www.postgresql.org/message-id/flat/182b8565486.10af1a86f158715.2387262617218380588@highgo.ca)
- **Date:** decided 2024-03-22; started 2024-03-22
- **Outcome:** Reverted (revert)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (as recorded in the corpus):** This reverts commit 6acb0a628eccab8764e0306582c2b7e2a1441b9b since LibreSSL didn't support ASN1_TIME_diff until OpenBSD 7.1, leaving the older OpenBSD animals in the buildfarm complaining. Per plover in the buildfarm.
- **Reason (quote):** "since LibreSSL didn't support ASN1_TIME_diff until OpenBSD 7.1, leaving the older OpenBSD animals in the buildfarm complaining."
- **What is different this time:** P17 reverted the same cert-date feature again because LibreSSL on older OpenBSD lacked a function. This proposal guards the group functions with build-time checks and returns no rows when they are missing.


## 3. Objections a reviewer is likely to raise

### 1. Row-type layout fixed before the wider handshake-info design is settled

fixable · level 2, rubric item 6 · design, ABI / extension API, maintenance burden ("who maintains this in 10 years") · `maintenance-fit-3`

A later patch exposes ciphersuites or signature algorithms with the same negotiated/shared/supported pattern. It must either copy this group_type vocabulary, with its OpenSSL-derived names and no server row, or add a different shape next to it. sslinfo then has two styles that users must correlate.

In message #8, Champion asked whether this table is how future negotiated parameters should look, and noted the missing server side. Message #9 answers 'I don't see why not', but the server list cannot be fetched (no public OpenSSL API), and 'shared' is not a TLS term. Champion and Si (messages #6, #17) doubt the non-negotiated rows. A single ssl_group() scalar, like ssl_cipher(), would commit the project to less.

Thread: [#6 (Jacob Champion, 2026-02-23)](https://www.postgresql.org/message-id/CAOYmi%2Bk7v6hP5nM7BQdKu37TJFi-X%3Dd7_SDswZBV5q0awxPVYg%40mail.gmail.com), [#8 (Jacob Champion, 2026-02-28)](https://www.postgresql.org/message-id/CAOYmi%2B%3Dr50Kk1c7A7O8yXwJzALyyqDmQE7FiCZvZmt_3WRBGwQ%40mail.gmail.com), [#9 (Dmitry Dolgov, 2026-03-03)](https://www.postgresql.org/message-id/izopeja2suconbqpbswmtto5czajnc4p7rsml5gwwrs56lexaj%40kmqdrqiepays), [#17 ("Si, Evan", 2026-08-31)](https://www.postgresql.org/message-id/a86c254e-834b-49e0-8ea1-f5ae58f61838%40gmail.com)

**Question:** If ssl_group() returned only the negotiated group like ssl_cipher(), what diagnostic case would still need the shared and client supported rows?

Counter-check: held. Message #9 answers Champion's question about the table shape with "I don't see why not" and gives no design. The server list is still missing, and Champion and Si both doubt the non-negotiated rows.

### 2. ssl_group_info_with_nulls() becomes a permanent, undocumented part of sslinfo 1.3

fixable · level 2, rubric item 10 · SQL-standard conflict or non-standard syntax, maintenance burden ("who maintains this in 10 years") · `standards-compat-1`

A user finds ssl_group_info_with_nulls() through \dx+ sslinfo and uses it in a monitoring query. A later release wants to drop or change it and needs a new extension version, which breaks that query. Both an RSA connection and an unknown client group produce a null row, so neither user nor maintainer can tell them apart.

Message #18 adds the function as an internal way around SRF limits, but every function in an extension script is user-callable API. The documentation hunk does not describe it, and no other sslinfo function follows a "_with_nulls" naming pattern. Message #21 maps the unknown-name case of message #20 onto the same null rows, so two different facts look the same after filtering. A committer would likely ask for one function whose result states the absent or unknown group directly.

Thread: [#18 (Dmitry Dolgov, 2026-09-04)](https://www.postgresql.org/message-id/aprTMCAqd16B__py%40ddolgov-thinkpadt14sgen1.rmtde.csb), [#19 ("Si, Evan", 2026-09-04)](https://www.postgresql.org/message-id/439f9b38-c610-48e0-ae1d-798a58dc2cc8%40gmail.com), [#20 (Zsolt Parragi, 2026-09-06)](https://www.postgresql.org/message-id/CAN4CZFNm6xwvnYfqCQLASUp6FtB%2B5mOJKnoLFjs1-NbLx1oyxQ%40mail.gmail.com), [#21 (Dmitry Dolgov, 2026-09-08)](https://www.postgresql.org/message-id/aqAlC4G8bTekShA5%40ddolgov-thinkpadt14sgen1.rmtde.csb)

**Question:** Can a single documented ssl_group_info() return explicit rows, such as a null name or an 'unknown (0x...)' name, so that no helper function ships?

Counter-check: held. Message #18 adds the helper function to the extension script, and the v7 documentation hunk does not describe it. Message #21 maps the unknown-name case onto the same null rows, so the two cases look the same.

### 3. Same sslinfo 1.3 returns an empty set on builds without group functions

fixable · level 2, rubric item 12 · design, SQL-standard conflict or non-standard syntax, backpatchability · `standards-compat-2`

A DBA runs ssl_group_info() on a package built against OpenSSL 1.1.1 and gets zero rows. They read this as no key exchange group, or a non-TLS connection. Nothing tells them the build lacks the API, and the same query returns rows on another server running the same extension version.

Message #20 asked for diagnostics for the silent skip, and message #21 answered a different point. The documentation hunk does not mention build-dependent results. Other build-dependent features usually report "not supported by this build" when called. A committer would likely want an error, or at least a documented and distinguishable result, before calling the contract stable.

Thread: [#12 (Dmitry Dolgov, 2026-03-21)](https://www.postgresql.org/message-id/tuobeew3dd3eubtelwroocwgac3jhlselwshs2km5ovei77tao%40wghkpdvvthbi), [#16 (Dmitry Dolgov, 2026-05-13)](https://www.postgresql.org/message-id/uy2ho4qzwzsm5zqi2s3po4iesgvzfybdq6vldqpsldeot6xkyn%40repwtjoc63ok), [#20 (Zsolt Parragi, 2026-09-06)](https://www.postgresql.org/message-id/CAN4CZFNm6xwvnYfqCQLASUp6FtB%2B5mOJKnoLFjs1-NbLx1oyxQ%40mail.gmail.com), [#21 (Dmitry Dolgov, 2026-09-08)](https://www.postgresql.org/message-id/aqAlC4G8bTekShA5%40ddolgov-thinkpadt14sgen1.rmtde.csb)

**Question:** Why should a build without the OpenSSL group API return an empty set instead of raising a feature-not-supported error?

Counter-check: held. Message #20 asked for documentation or diagnostics for the skip, and message #21 answers only the unknown-name point. The v7 documentation hunk does not mention the empty result on builds without the group functions.

### 4. No shown demand for the shared and client supported rows

fixable · level 1b, rubric item 3 · demand, doesn't fit what Postgres is about (extension / external tool / already solved), maintenance burden ("who maintains this in 10 years") · `demand-1`

An administrator asks why a client did not use X25519MLKEM768 and gets the client supported and shared lists, but not the server's list, so the question stays open. The project still has to keep the three group_type values, their names and their meaning stable in every later sslinfo version.

The only use case given for the non-negotiated rows is the author's own experiments (messages #1, #5), and message #9 says the work began only to get the negotiated group. Champion says he uses other tools for TLS debugging and suggests server-side debug logging (message #6), and Si doubts the rows (message #17). No field report or user request in the material asks for these lists. Without the server side, which message #9 says OpenSSL cannot expose, the lists cannot explain a negotiation result.

Thread: [#1 (Dmitry Dolgov, 2026-02-19)](https://www.postgresql.org/message-id/d57duqvzkxe43oons3jkdq7pj2wacidg7qorxommri74evu3l2%404x53she7mf77), [#5 (Dmitry Dolgov, 2026-02-23)](https://www.postgresql.org/message-id/srua2tidoiztaytmxlwjfpjhntxelmxpfta4lhulvlker444yg%40sf232zqm3qvs), [#6 (Jacob Champion, 2026-02-23)](https://www.postgresql.org/message-id/CAOYmi%2Bk7v6hP5nM7BQdKu37TJFi-X%3Dd7_SDswZBV5q0awxPVYg%40mail.gmail.com), [#9 (Dmitry Dolgov, 2026-03-03)](https://www.postgresql.org/message-id/izopeja2suconbqpbswmtto5czajnc4p7rsml5gwwrs56lexaj%40kmqdrqiepays), [#17 ("Si, Evan", 2026-08-31)](https://www.postgresql.org/message-id/a86c254e-834b-49e0-8ea1-f5ae58f61838%40gmail.com)

**Question:** Which users have needed the shared or client supported lists from SQL, and what question could they answer with them that openssl s_client or server logs cannot?

Counter-check: held. Message #9 says the work began only to get the negotiated group, and the thread shows no user request for the other rows. Champion and Si both question them, and the author's reply gives no diagnostic case.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | sslinfo 1.3 adds two SQL functions that stay installed until a later extension version removes them. |
| SQL-standard conflict or non-standard syntax | **yes** | see objection standards-compat-1: ssl_group_info_with_nulls() becomes a permanent, undocumented part of sslinfo 1.3 |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | **yes** | The function reports TLS handshake facts, limited to the caller's own connection. |
| backpatchability | **yes** | see objection standards-compat-2: Same sslinfo 1.3 returns an empty set on builds without group functions |
| maintenance burden ("who maintains this in 10 years") | **yes** | New configure and meson checks for OpenSSL group functions, plus fixed group_type values and a helper function. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | Reviewers in messages #6 and #17 doubt that the shared and client supported rows belong in SQL. |

## 5. Questions the author must answer before anyone reviews code

1. Which diagnostic task needs the shared and client supported rows from SQL, given that the server's own group list is not available?
2. Why not ship a scalar ssl_group() that returns only the negotiated group, like ssl_cipher(), and add the lists later if users ask?
3. Can ssl_group_info() skip or label missing groups in C, so that ssl_group_info_with_nulls() is not installed?
4. How does a user tell apart no negotiated group, an unknown group name, and a build without the OpenSSL group functions?
5. If ciphersuites or signature algorithms are exposed later, will they use the same group_type table shape and values?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Daniel Gustafsson reviewed in message #2 and handled the earlier sslinfo reverts (P4, P17). The owner needs to know the OpenSSL and LibreSSL build matrix.
- **Liability:** The owner signs for the build checks on every supported OpenSSL and LibreSSL version, and for the function names and group_type values, for as long as sslinfo 1.3 is installed anywhere.
- **Author follow-up:** The author posted seven versions in response to review and moved the entry between CommitFests, which suggests continued follow-up.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch is about 15 KiB, confined to contrib/sslinfo and build checks, and serves people who debug TLS key exchange. The function set and result shape need agreement before another review round is useful.

## Appendix

### Refuted in the counter-check

- `standards-compat-3` group_type text values are a fixed contract that is not yet settled. Refuted (no failure scenario): A new server-side value would not stop an existing filter on 'client supported' from matching, and no one in the thread proposes a rename after v7. The open design question about the table shape is covered by maintenance-fit-3.
- `maintenance-fit-1` Version-dependent OpenSSL API added to top-level configure for one contrib function. Refuted (no failure scenario): P4 and P17 were reverted for lacking compatibility checks, which this patch adds, and Postgres already probes OpenSSL functions in configure and meson. The silent empty result is a duplicate of standards-compat-2, and the future macro change is speculation.
- `maintenance-fit-2` ssl_group_info_with_nulls() makes an implementation detail a permanent SQL API. Refuted (not specific): Duplicate of standards-compat-1. Its point that a C set-returning function can advance past an entry before returning a row is correct and supports that objection.
- `already-exists-1` Client supported and shared group rows repeat what a handshake capture shows. Refuted (not specific): Duplicate of demand-1, which makes the same point about the shared and client supported rows with better support from the thread. The packet capture alternative can be raised under that objection.

### Review angles

- prior-attempts: 0 objection(s). No earlier attempt at this idea is in the list. P4 and P17 are two reverts of other sslinfo functions, caused by missing OpenSSL or LibreSSL functions. This patch answers that risk with build-time checks, so the only check left is a buildfarm run on LibreSSL and old OpenSSL. The thread does not mention any earlier discussion of exposing TLS groups.
- standards-compat: 3 objection(s). P4 and P17 were reverts over OpenSSL and LibreSSL build breaks. This patch adds configure and meson checks, so those recorded reasons do not apply directly.
- replication-ondisk: 0 objection(s). The proposal holds no persistent or cross-node state. It adds functions that read the TLS state of the current backend's connection, with no WAL, on-disk, wire protocol or core catalog change. The 1.2 to 1.3 upgrade script follows normal extension practice, and v7 no longer touches libpq-be.h.
- security: 0 objection(s). No security objection. ssl_group_info() reports only the current connection's own handshake facts, the same scope as ssl_cipher() and ssl_version(). It shows nothing about other sessions, server memory or files. The error on an unknown group name affects only the calling session. Precedents [P4] and [P17] are OpenSSL portability reverts with no privilege aspect, so they belong to another angle.
- maintenance-fit: 3 objection(s). The change is confined to sslinfo and costs nothing for non-users. The objections concern build-matrix upkeep and the permanent SQL surface.
- already-exists: 1 objection(s). The negotiated group is not exposed today by ssl_cipher(), pg_stat_ssl or connection logging, so that part has no existing alternative.
- demand: 1 objection(s). Demand for the negotiated group itself is reasonable: Champion and Si want it next to ssl_cipher() (messages #4, #17), and Huang supports the extension (message #13). The cost stays inside contrib/sslinfo.

### Precedents retrieved

- P1: [Add test case for sslinfo](https://commitfest.postgresql.org/patch/2203/) (cf, Returned with feedback, 2019-09-30) - judged not relevant; reason summarised from its thread
- P2: [Revert "Add GUC checks for ssl_min_protocol_version and ssl_max_protocol_version"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=414c2fd1e1c0ccd1af1cad046bef1264e3f4347f) (revert, Reverted, 2020-02-06) - judged not relevant
- P3: [Direct SSL Connections](https://commitfest.postgresql.org/patch/4135/) (cf, Returned with feedback, 2023-12-02) - judged not relevant; reason summarised from its thread
- P4: [Revert "Add notBefore and notAfter to SSL cert info display"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=29a0ccbce97978e5d65b8f96c85a00611bb403c4) (revert, Reverted, 2023-07-20) - used in the review
- P5: [Revert "Optimize order of GROUP BY keys".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f4c7c410ee4a7baa06f51ebb8d5333c169691dd3) (revert, Reverted, 2022-10-03) - judged not relevant
- P6: [Revert changes for SSL compression in libpq](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=0ba71107efeeccde9158f47118f95043afdca0bb) (revert, Reverted, 2021-03-10) - judged not relevant
- P7: [Parallel grouping sets](https://commitfest.postgresql.org/patch/2147/) (cf, Returned with feedback, 2020-07-12) - judged not relevant; reason summarised from its thread
- P8: [remove_useless_groupby_columns is too enthusiastic](https://commitfest.postgresql.org/patch/3766/) (cf, Returned with feedback, 2022-10-12) - judged not relevant; reason summarised from its thread
- P9: [group locking](https://commitfest.postgresql.org/patch/23/) (cf, Rejected, 2015-01-19) - judged not relevant; reason summarised from its thread
- P10: [Indeterminate behavior for the GROUP BY clause](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Features_We_Do_Not_Want) (do_not_want, Not wanted, date unknown) - judged not relevant
- P11: [Collecting statistics about contents of JSONB columns](https://commitfest.postgresql.org/patch/3500/) (cf, Returned with feedback, 2022-04-07) - judged not relevant; reason summarised from its thread
- P12: [When deleting the plpgsql function, release the CachedPlan of the function](https://commitfest.postgresql.org/patch/5974/) (cf, Withdrawn, 2025-08-19) - judged not relevant; reason summarised from its thread
- P13: [Add sslmode "no-clientcert" to avoid auth failure in md5/scram connections](https://commitfest.postgresql.org/patch/4064/) (cf, Withdrawn, 2023-02-21) - judged not relevant; reason summarised from its thread
- P14: [new diagnostic items for the dynamic sql](https://commitfest.postgresql.org/patch/3537/) (cf, Returned with feedback, 2022-12-14) - judged not relevant
- P15: [When deleting the plpgsql function, release the CachedPlan of the function](https://commitfest.postgresql.org/patch/5975/) (cf, Withdrawn, 2025-12-31) - judged not relevant
- P16: [GET DIAGNOSTICS FUNCTION_NAME](https://commitfest.postgresql.org/patch/1457/) (cf, Rejected, 2018-03-07) - judged not relevant
- P17: [Revert "Add notBefore and notAfter to SSL cert info display"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=697f8d266cfb33409f7ccf3319f4448477066329) (revert, Reverted, 2024-03-22) - used in the review
- P18: [Support tls-exporter as channel binding for TLSv1.3](https://commitfest.postgresql.org/patch/3850/) (cf, Returned with feedback, 2022-11-30) - judged not relevant
- P19: [Remove useless GROUP BY columns considering unique index](https://commitfest.postgresql.org/patch/4751/) (cf, Returned with feedback, 2024-10-07) - judged not relevant
- P20: [Revert "Only provide new libpq sslpasskey hook for openssl-enabled builds"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6741cfa5964768279a13e01e46931664fe0903a4) (revert, Reverted, 2020-04-17) - judged not relevant
- P21: [Revert 019_replslot_limit.pl related debugging aids.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=3f8148c256e067dc2e8929ed174671ba7dc3339c) (revert, Reverted, 2022-07-05) - judged not relevant
- P22: [Tab completion for ALTER COLUMN SET STATISTICS](https://commitfest.postgresql.org/patch/377/) (cf, Rejected, 2015-12-21) - judged not relevant
- P23: [Add parallel columns for seq scan and index scan on pg_stat_all_tables and _indexes](https://commitfest.postgresql.org/patch/5238/) (cf, Returned with feedback, 2024-12-11) - judged not relevant
- P24: [elog/ereport VS misleading backtrace_function function address](https://commitfest.postgresql.org/patch/4977/) (cf, Rejected, 2024-08-22) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 15; wall time: 145 s; cost reported by the CLI: $2.11
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 5 from cache, 3 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 89 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:55:40.732Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

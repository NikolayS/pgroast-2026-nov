# Design pre-review: Add ssl_(supported|shared)_groups to sslinfo

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6598](https://commitfest.postgresql.org/patch/6598/); [thread](https://www.postgresql.org/message-id/flat/d57duqvzkxe43oons3jkdq7pj2wacidg7qorxommri74evu3l2%404x53she7mf77), 22 of 22 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). Reviewers agree the negotiated group is worth exposing, and the cost stays inside contrib/sslinfo. The shape of the result is not settled. demand-1 and maintenance-fit-3 show no use case for the shared and client supported rows, and the server's list cannot be fetched. standards-compat-1 and standards-compat-2 show that v7 ships an undocumented helper function and an empty result that has several meanings. Nobody has replied to v7.

Not checked: Without the code, I could not check what v7 returns for RSA or non-EC DHE key exchange, or whether it still raises an error for an unknown group name. The build checks on LibreSSL and old OpenSSL were also not verified.

## Prior attempts

- [Revert "Add notBefore and notAfter to SSL cert info display"](https://www.postgresql.org/message-id/flat/182b8565486.10af1a86f158715.2387262617218380588@highgo.ca) (related; Reverted, 2023-07-20). "Due to an oversight in reviewing, this used functionality not compatible with old versions of OpenSSL." Different now: P4 reverted a different sslinfo addition, cert validity dates, because it used OpenSSL functions that old versions lack. This proposal adds configure and meson checks for the group functions.
- [Revert "Add notBefore and notAfter to SSL cert info display"](https://www.postgresql.org/message-id/flat/F0DF7102-192D-4C21-96AE-9A01AE153AD1@yesql.se) (related; Reverted, 2024-03-22). "since LibreSSL didn't support ASN1_TIME_diff until OpenBSD 7.1, leaving the older OpenBSD animals in the buildfarm complaining." Different now: P17 reverted the same cert-date feature again because LibreSSL on older OpenBSD lacked a function. This proposal guards the group functions with build-time checks and returns no rows when they are missing.

## Objections a reviewer is likely to raise

### 1. Row-type layout fixed before the wider handshake-info design is settled

A later patch exposes ciphersuites or signature algorithms with the same negotiated/shared/supported pattern. It must either copy this group_type vocabulary, with its OpenSSL-derived names and no server row, or add a different shape next to it. sslinfo then has two styles that users must correlate.

In message #8, Champion asked whether this table is how future negotiated parameters should look, and noted the missing server side. Message #9 answers 'I don't see why not', but the server list cannot be fetched (no public OpenSSL API), and 'shared' is not a TLS term. Champion and Si (messages #6, #17) doubt the non-negotiated rows. A single ssl_group() scalar, like ssl_cipher(), would commit the project to less.

**Question:** If ssl_group() returned only the negotiated group like ssl_cipher(), what diagnostic case would still need the shared and client supported rows?

<sub>fixable, level 2, design; messages [#6](https://www.postgresql.org/message-id/CAOYmi%2Bk7v6hP5nM7BQdKu37TJFi-X%3Dd7_SDswZBV5q0awxPVYg%40mail.gmail.com), [#8](https://www.postgresql.org/message-id/CAOYmi%2B%3Dr50Kk1c7A7O8yXwJzALyyqDmQE7FiCZvZmt_3WRBGwQ%40mail.gmail.com), [#9](https://www.postgresql.org/message-id/izopeja2suconbqpbswmtto5czajnc4p7rsml5gwwrs56lexaj%40kmqdrqiepays), [#17](https://www.postgresql.org/message-id/a86c254e-834b-49e0-8ea1-f5ae58f61838%40gmail.com); `maintenance-fit-3`</sub>

### 2. ssl_group_info_with_nulls() becomes a permanent, undocumented part of sslinfo 1.3

A user finds ssl_group_info_with_nulls() through \dx+ sslinfo and uses it in a monitoring query. A later release wants to drop or change it and needs a new extension version, which breaks that query. Both an RSA connection and an unknown client group produce a null row, so neither user nor maintainer can tell them apart.

Message #18 adds the function as an internal way around SRF limits, but every function in an extension script is user-callable API. The documentation hunk does not describe it, and no other sslinfo function follows a "_with_nulls" naming pattern. Message #21 maps the unknown-name case of message #20 onto the same null rows, so two different facts look the same after filtering. A committer would likely ask for one function whose result states the absent or unknown group directly.

**Question:** Can a single documented ssl_group_info() return explicit rows, such as a null name or an 'unknown (0x...)' name, so that no helper function ships?

<sub>fixable, level 2, SQL-standard conflict or non-standard syntax; messages [#18](https://www.postgresql.org/message-id/aprTMCAqd16B__py%40ddolgov-thinkpadt14sgen1.rmtde.csb), [#19](https://www.postgresql.org/message-id/439f9b38-c610-48e0-ae1d-798a58dc2cc8%40gmail.com), [#20](https://www.postgresql.org/message-id/CAN4CZFNm6xwvnYfqCQLASUp6FtB%2B5mOJKnoLFjs1-NbLx1oyxQ%40mail.gmail.com), [#21](https://www.postgresql.org/message-id/aqAlC4G8bTekShA5%40ddolgov-thinkpadt14sgen1.rmtde.csb); `standards-compat-1`</sub>

### 3. Same sslinfo 1.3 returns an empty set on builds without group functions

A DBA runs ssl_group_info() on a package built against OpenSSL 1.1.1 and gets zero rows. They read this as no key exchange group, or a non-TLS connection. Nothing tells them the build lacks the API, and the same query returns rows on another server running the same extension version.

Message #20 asked for diagnostics for the silent skip, and message #21 answered a different point. The documentation hunk does not mention build-dependent results. Other build-dependent features usually report "not supported by this build" when called. A committer would likely want an error, or at least a documented and distinguishable result, before calling the contract stable.

**Question:** Why should a build without the OpenSSL group API return an empty set instead of raising a feature-not-supported error?

<sub>fixable, level 2, design; messages [#12](https://www.postgresql.org/message-id/tuobeew3dd3eubtelwroocwgac3jhlselwshs2km5ovei77tao%40wghkpdvvthbi), [#16](https://www.postgresql.org/message-id/uy2ho4qzwzsm5zqi2s3po4iesgvzfybdq6vldqpsldeot6xkyn%40repwtjoc63ok), [#20](https://www.postgresql.org/message-id/CAN4CZFNm6xwvnYfqCQLASUp6FtB%2B5mOJKnoLFjs1-NbLx1oyxQ%40mail.gmail.com), [#21](https://www.postgresql.org/message-id/aqAlC4G8bTekShA5%40ddolgov-thinkpadt14sgen1.rmtde.csb); `standards-compat-2`</sub>

### 4. No shown demand for the shared and client supported rows

An administrator asks why a client did not use X25519MLKEM768 and gets the client supported and shared lists, but not the server's list, so the question stays open. The project still has to keep the three group_type values, their names and their meaning stable in every later sslinfo version.

The only use case given for the non-negotiated rows is the author's own experiments (messages #1, #5), and message #9 says the work began only to get the negotiated group. Champion says he uses other tools for TLS debugging and suggests server-side debug logging (message #6), and Si doubts the rows (message #17). No field report or user request in the material asks for these lists. Without the server side, which message #9 says OpenSSL cannot expose, the lists cannot explain a negotiation result.

**Question:** Which users have needed the shared or client supported lists from SQL, and what question could they answer with them that openssl s_client or server logs cannot?

<sub>fixable, level 1b, demand; messages [#1](https://www.postgresql.org/message-id/d57duqvzkxe43oons3jkdq7pj2wacidg7qorxommri74evu3l2%404x53she7mf77), [#5](https://www.postgresql.org/message-id/srua2tidoiztaytmxlwjfpjhntxelmxpfta4lhulvlker444yg%40sf232zqm3qvs), [#6](https://www.postgresql.org/message-id/CAOYmi%2Bk7v6hP5nM7BQdKu37TJFi-X%3Dd7_SDswZBV5q0awxPVYg%40mail.gmail.com), [#9](https://www.postgresql.org/message-id/izopeja2suconbqpbswmtto5czajnc4p7rsml5gwwrs56lexaj%40kmqdrqiepays), [#17](https://www.postgresql.org/message-id/a86c254e-834b-49e0-8ea1-f5ae58f61838%40gmail.com); `demand-1`</sub>

## Questions for the author

1. Which diagnostic task needs the shared and client supported rows from SQL, given that the server's own group list is not available?
2. Why not ship a scalar ssl_group() that returns only the negotiated group, like ssl_cipher(), and add the lists later if users ask?
3. Can ssl_group_info() skip or label missing groups in C, so that ssl_group_info_with_nulls() is not installed?
4. How does a user tell apart no negotiated group, an unknown group name, and a build without the OpenSSL group functions?
5. If ciphersuites or signature algorithms are exposed later, will they use the same group_type table shape and values?

## Touch points

- **ABI / extension API:** sslinfo 1.3 adds two SQL functions that stay installed until a later extension version removes them.
- **SQL-standard conflict or non-standard syntax:** see objection standards-compat-1: ssl_group_info_with_nulls() becomes a permanent, undocumented part of sslinfo 1.3
- **security / privileges:** The function reports TLS handshake facts, limited to the caller's own connection.
- **backpatchability:** see objection standards-compat-2: Same sslinfo 1.3 returns an empty set on builds without group functions
- **maintenance burden ("who maintains this in 10 years"):** New configure and meson checks for OpenSSL group functions, plus fixed group_type values and a helper function.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** Reviewers in messages #6 and #17 doubt that the shared and client supported rows belong in SQL.

## Who would commit it, and when

- Owner: qualified committer engaged. Daniel Gustafsson reviewed in message #2 and handled the earlier sslinfo reverts (P4, P17). The owner needs to know the OpenSSL and LibreSSL build matrix.
- Priority: fix design first; audience niche; review cost low. The patch is about 15 KiB, confined to contrib/sslinfo and build checks, and serves people who debug TLS key exchange. The function set and result shape need agreement before another review round is useful.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 4 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

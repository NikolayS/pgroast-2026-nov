# Design pre-review: Warn, and disallow, clear text passwords via SQL

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 5597](https://commitfest.postgresql.org/patch/5597/); [thread](https://www.postgresql.org/message-id/flat/CAKAnmmJcXyLeBUJfqCx%2B-gRmkooDPnH7OmM4o%3D2HJRXzbMkP_g%40mail.gmail.com), 33 of 33 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The proposal fails at level 1b on fixable points. The latest patch still defaults to `warn` with a deprecation DETAIL, but the thread has no agreement on removal (maintenance-fit-1). Nobody has answered message #28, which says the mechanism does not solve the reported complaints (demand-1), or said why a check_password_hook module is not enough (already-exists-1). The goal of keeping passwords out of logs is not in dispute.

Not checked: The code was not read, so the GUC context and the exact point where the check runs are unknown. Level 2 objections on logging, passwordcheck and the GUC context survived but were not evaluated, because level 1b fails first.

## Prior attempts

None found before 2025-02-21.

## Objections a reviewer is likely to raise

### 1. Default warn puts a lasting cost on every user with no removal plan

An extension or tool runs `create role ... password 'x'` in its regression suite or its setup scripts. After upgrading, every run prints the new WARNING, so expected outputs fail and logs fill. The patch itself already has to change passwordcheck, ecpg and core expected files.

The warning reaches everyone, while the gain goes to sites that care about password logging, and those sites can opt in. Nathan (messages #12, #14, #29) and Robert Treat (message #32) tie a default warning to a removal decision that does not exist; the author places removal no sooner than v24 (message #11). A deprecation warning with no end date stays in the product for good. The author has offered default `allow` (messages #26, #31), but the patch still documents `warn`.

**Question:** Will the next version ship with default `allow`, and if `warn` stays, which release removes clear text passwords?

<sub>fixable, level 1b, desirability; messages [#11](https://www.postgresql.org/message-id/CAKAnmmLPmt_%3DtC_KsPMV1W2qGmgyLTx47bAKmh1RQhofhNZcLQ%40mail.gmail.com), [#12](https://www.postgresql.org/message-id/Z73jBpWlBEuvLjEj%40nathan), [#14](https://www.postgresql.org/message-id/Z8XZ4f4XXfptmRS-%40nathan), [#26](https://www.postgresql.org/message-id/CAKAnmm%2BO%3DqBBYaKwYCRm_yfMNbpfYz9xjbdQdQ9RLdgjgRX0CA%40mail.gmail.com), [#29](https://www.postgresql.org/message-id/Z9rcRn53zhFdgIDc%40nathan), [#31](https://www.postgresql.org/message-id/CAKAnmm%2B5Yi1uSmmjGop2Gz28PRzWAuxL9LZQMuccHoC588C3YQ%40mail.gmail.com), [#32](https://www.postgresql.org/message-id/CABV9wwOrh9aEdaRQQa52_fEMzxGGfji8mL_Bi5PzqrvCyxLBXw%40mail.gmail.com); `maintenance-fit-1`</sub>

### 2. No evidence that users would turn on the disallow mode

A DBA who sees passwords in logs sets `disallow`. Clients still send `alter role ... password` with a clear text password, so the statement and its password reach the log through the ERROR path (message #8). The DBA's problem remains, and the project still carries a GUC that few people set.

The thread shows real complaints about passwords in logs, but not demand for this mechanism. Robert Haas expects it to solve "0%" of the cases he has seen (message #28). Robert Treat notes little adoption of credcheck, an extension that already rejects clear text passwords (message #32), and Aleksander Alekseev expects few users to choose `disallow` (message #4). The PCI DSS point in message #33 is about logs and saved pg_stat_statements data, and the GUC alone does not keep those clean.

**Question:** Which users or installations would enable `disallow`, and why have they not used credcheck or a passwordcheck-style hook for this?

<sub>fixable, level 1b, demand; messages [#4](https://www.postgresql.org/message-id/CAJ7c6TP%3DnGgMhzrNgT8uzqaQj_nkT6HkYNgUyYcDqaL3R8dGog%40mail.gmail.com), [#8](https://www.postgresql.org/message-id/Z7za0Q9Yj5HDU-ye%40nathan), [#28](https://www.postgresql.org/message-id/CA%2BTgmobL3w8bytd%2BeOCcpn%3DNHqqS%2Bvyddcsimv4Yq72sqwWyYw%40mail.gmail.com), [#32](https://www.postgresql.org/message-id/CABV9wwOrh9aEdaRQQa52_fEMzxGGfji8mL_Bi5PzqrvCyxLBXw%40mail.gmail.com), [#33](https://www.postgresql.org/message-id/CAKZiRmwkv8iWwxfHjCyA5Ewq-P8zGgbLfnu%3DxNoTj8ks3Y2P8Q%40mail.gmail.com); `demand-1`</sub>

### 3. check_password_hook already lets an extension reject or warn on clear text

A DBA who must keep clear text passwords out of `create role` and `alter role` can load a module on check_password_hook today and get the same ERROR or WARNING. Core instead carries a permanent GUC whose default is disputed, and every installation sees the new warning and changed test output.

The existing check_password_hook receives the password type, so a module can tell a clear text password from a pre-encrypted one; contrib/passwordcheck uses this hook. Message #32 points to credcheck as an existing tool for this need and reports little adoption of it. The `disallow` mode is therefore mostly a convenience, and the opt-in setting that message #33 asks for fits an extension. Only a core-wide default `warn` needs core, and messages #14, #28 and #32 oppose that default.

**Question:** With check_password_hook and credcheck already available, what does core placement of `disallow` give a DBA that a loaded module cannot?

<sub>fixable, level 1b, doesn't fit what Postgres is about (extension / external tool / already solved); messages [#14](https://www.postgresql.org/message-id/Z8XZ4f4XXfptmRS-%40nathan), [#28](https://www.postgresql.org/message-id/CA%2BTgmobL3w8bytd%2BeOCcpn%3DNHqqS%2Bvyddcsimv4Yq72sqwWyYw%40mail.gmail.com), [#32](https://www.postgresql.org/message-id/CABV9wwOrh9aEdaRQQa52_fEMzxGGfji8mL_Bi5PzqrvCyxLBXw%40mail.gmail.com), [#33](https://www.postgresql.org/message-id/CAKZiRmwkv8iWwxfHjCyA5Ewq-P8zGgbLfnu%3DxNoTj8ks3Y2P8Q%40mail.gmail.com); `already-exists-1`</sub>

## Questions for the author

1. Will the next version default to `allow` and drop the deprecation DETAIL, and if `warn` stays, which release removes clear text passwords?
2. Which installations would set `disallow`, and what does a core GUC give them that a check_password_hook module such as credcheck does not?
3. How does `disallow` keep the rejected statement and its password out of the server log under the default `log_min_error_statement`?
4. What is the GUC context, and which roles can lower the setting from `disallow` for their own session or role?
5. How would `cleartext_passwords_action` fit with the `PASSWORD` command from message #33 without a second overlapping setting?

## Touch points

- **new GUC:** Adds the enum GUC `cleartext_passwords_action` with default `warn`.
- **psql UX conventions:** The HINT recommends `\password` when application_name is 'psql'.
- **security / privileges:** `disallow` is a password policy control, and the documentation hunk does not state who may change it.
- **maintenance burden ("who maintains this in 10 years"):** A permanent GUC and a deprecation warning with no agreed removal release.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** A module on check_password_hook can already reject or warn on clear text passwords.

## Who would commit it, and when

- Owner: none obvious. A committer who knows authentication and password handling would have to own it. Robert Haas says in message #28 that he will not commit it, and Nathan Bossart supports only the enforcement part.
- Priority: needs design rethink; audience a sizeable minority; review cost low. The patch is small and adds one GUC, but the default, the scope and the relation to the `PASSWORD` command in message #33 are all open. No committer has offered to own it until the list agrees on the overall approach.

---

[roast-full.md](roast-full.md) has 4 more objection(s) that held, 5 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

# Design pre-review: Warn, and disallow, clear text passwords via SQL

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 5597](https://commitfest.postgresql.org/patch/5597/) (Needs review); authors: Greg Sabino Mullane (greg)
- Thread: [PATCH: warn about, and deprecate, clear text passwords](https://www.postgresql.org/message-id/flat/CAKAnmmJcXyLeBUJfqCx%2B-gRmkooDPnH7OmM4o%3D2HJRXzbMkP_g%40mail.gmail.com), started 2025-02-21
- Messages read: 33 of 33
- Patch prose read: 0003-Add-new-server-config-cleartext_passwords_action.patch, 0001-Add-new-server-config-cleartext_passwords_action.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The proposal fails at level 1b on fixable points. The latest patch still defaults to `warn` with a deprecation DETAIL, but the thread has no agreement on removal (maintenance-fit-1). Nobody has answered message #28, which says the mechanism does not solve the reported complaints (demand-1), or said why a check_password_hook module is not enough (already-exists-1). The goal of keeping passwords out of logs is not in dispute.

First failing level, from the base up: **Level 1b: desirability**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: failed
- Level 2: architecture and design: not evaluated
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so the GUC context and the exact point where the check runs are unknown. Level 2 objections on logging, passwordcheck and the GUC context survived but were not evaluated, because level 1b fails first.

## The proposal, restated

**Problem.** Clear text passwords in `create role` or `alter role` end up in pg_stat_activity, pg_stat_statements, .psql_history and the server logs. Today the only advice to users and DBAs is "don't do that". Message #33 adds that PCI DSS requires credentials to be unreadable in storage, and logs and saved pg_stat_statements data break that rule.

**Proposed design.** A new enum GUC `cleartext_passwords_action` takes `allow`, `warn` or `disallow`, with `warn` as the default. On `warn` the server emits a WARNING, and the HINT suggests `\password` when application_name is 'psql'. On `disallow` the server raises an ERROR when a password is set or changed in clear text, while pre-encrypted passwords still pass.

**User-visible surface.**
- GUC `cleartext_passwords_action` (enum: allow, warn, disallow; default warn), in postgresql.conf.sample.
- New WARNING "using a clear text password" with a deprecation DETAIL and a HINT that depends on application_name.
- New ERROR on `create role` / `alter role` with a clear text password when set to disallow.
- Changed regression and contrib/passwordcheck expected output, so existing tests now see the warning.
- Message #33 PoC only: a `PASSWORD $1, $2` top-level command, log redaction, and pg_dumpall -g using `\bind`.

**What the author claims.**
- The patch is fully backward compatible, because `allow` restores the old behavior.
- The warning tells users, drivers and frameworks that clear text passwords are a bad practice.
- `disallow` lets a DBA keep password changes out of logs and may help with PCI compliance.
- It prepares a later move of the default to disallow, and maybe removal, in about Postgres v24.
- Filtering passwords out of logs reliably is very hard because of syntax errors and pg_stat_statements.

**Thread state.** Guillaume Lelarge and Roberto Mello support the patch. Nathan Bossart and Robert Treat oppose a default warning without a removal plan, Robert Haas will not commit it, and Jakub Wartak (Aug 2026) asks for default `allow` and posts PoC patches for a `PASSWORD` command using the extended protocol.

## 2. Prior attempts

No precedent found (corpus searched for items before 2025-02-21; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Default warn puts a lasting cost on every user with no removal plan

fixable · level 1b, rubric item 4 · desirability, maintenance burden ("who maintains this in 10 years") · `maintenance-fit-1`

An extension or tool runs `create role ... password 'x'` in its regression suite or its setup scripts. After upgrading, every run prints the new WARNING, so expected outputs fail and logs fill. The patch itself already has to change passwordcheck, ecpg and core expected files.

The warning reaches everyone, while the gain goes to sites that care about password logging, and those sites can opt in. Nathan (messages #12, #14, #29) and Robert Treat (message #32) tie a default warning to a removal decision that does not exist; the author places removal no sooner than v24 (message #11). A deprecation warning with no end date stays in the product for good. The author has offered default `allow` (messages #26, #31), but the patch still documents `warn`.

Thread: [#11 (Greg Sabino Mullane, 2025-02-24)](https://www.postgresql.org/message-id/CAKAnmmLPmt_%3DtC_KsPMV1W2qGmgyLTx47bAKmh1RQhofhNZcLQ%40mail.gmail.com), [#12 (Nathan Bossart, 2025-02-25)](https://www.postgresql.org/message-id/Z73jBpWlBEuvLjEj%40nathan), [#14 (Nathan Bossart, 2025-03-03)](https://www.postgresql.org/message-id/Z8XZ4f4XXfptmRS-%40nathan), [#26 (Greg Sabino Mullane, 2025-03-19)](https://www.postgresql.org/message-id/CAKAnmm%2BO%3DqBBYaKwYCRm_yfMNbpfYz9xjbdQdQ9RLdgjgRX0CA%40mail.gmail.com), [#29 (Nathan Bossart, 2025-03-19)](https://www.postgresql.org/message-id/Z9rcRn53zhFdgIDc%40nathan), [#31 (Greg Sabino Mullane, 2026-01-09)](https://www.postgresql.org/message-id/CAKAnmm%2B5Yi1uSmmjGop2Gz28PRzWAuxL9LZQMuccHoC588C3YQ%40mail.gmail.com), [#32 (Robert Treat, 2026-01-09)](https://www.postgresql.org/message-id/CABV9wwOrh9aEdaRQQa52_fEMzxGGfji8mL_Bi5PzqrvCyxLBXw%40mail.gmail.com)

**Question:** Will the next version ship with default `allow`, and if `warn` stays, which release removes clear text passwords?

Counter-check: held. The latest patch still documents `warn` as the default, and messages #14, #29 and #32 show no agreement on removal. The author's offer of `allow` in messages #26 and #31 is not yet in a patch version.

### 2. No evidence that users would turn on the disallow mode

fixable · level 1b, rubric item 3 · demand, doesn't fit what Postgres is about (extension / external tool / already solved), new GUC, maintenance burden ("who maintains this in 10 years") · `demand-1`

A DBA who sees passwords in logs sets `disallow`. Clients still send `alter role ... password` with a clear text password, so the statement and its password reach the log through the ERROR path (message #8). The DBA's problem remains, and the project still carries a GUC that few people set.

The thread shows real complaints about passwords in logs, but not demand for this mechanism. Robert Haas expects it to solve "0%" of the cases he has seen (message #28). Robert Treat notes little adoption of credcheck, an extension that already rejects clear text passwords (message #32), and Aleksander Alekseev expects few users to choose `disallow` (message #4). The PCI DSS point in message #33 is about logs and saved pg_stat_statements data, and the GUC alone does not keep those clean.

Thread: [#4 (Aleksander Alekseev, 2025-02-24)](https://www.postgresql.org/message-id/CAJ7c6TP%3DnGgMhzrNgT8uzqaQj_nkT6HkYNgUyYcDqaL3R8dGog%40mail.gmail.com), [#8 (Nathan Bossart, 2025-02-24)](https://www.postgresql.org/message-id/Z7za0Q9Yj5HDU-ye%40nathan), [#28 (Robert Haas, 2025-03-19)](https://www.postgresql.org/message-id/CA%2BTgmobL3w8bytd%2BeOCcpn%3DNHqqS%2Bvyddcsimv4Yq72sqwWyYw%40mail.gmail.com), [#32 (Robert Treat, 2026-01-09)](https://www.postgresql.org/message-id/CABV9wwOrh9aEdaRQQa52_fEMzxGGfji8mL_Bi5PzqrvCyxLBXw%40mail.gmail.com), [#33 (Jakub Wartak, 2026-08-24)](https://www.postgresql.org/message-id/CAKZiRmwkv8iWwxfHjCyA5Ewq-P8zGgbLfnu%3DxNoTj8ks3Y2P8Q%40mail.gmail.com)

**Question:** Which users or installations would enable `disallow`, and why have they not used credcheck or a passwordcheck-style hook for this?

Counter-check: held. The thread shows support for enforcement from Guillaume Lelarge, Nathan Bossart, Roberto Mello and Jakub Wartak, so "no evidence" overstates it. But nobody has answered Robert Haas's point in message #28 that this mechanism does not solve the complaints he has seen.

### 3. check_password_hook already lets an extension reject or warn on clear text

fixable · level 1b, rubric item 3 · doesn't fit what Postgres is about (extension / external tool / already solved), new GUC, maintenance burden ("who maintains this in 10 years") · `already-exists-1`

A DBA who must keep clear text passwords out of `create role` and `alter role` can load a module on check_password_hook today and get the same ERROR or WARNING. Core instead carries a permanent GUC whose default is disputed, and every installation sees the new warning and changed test output.

The existing check_password_hook receives the password type, so a module can tell a clear text password from a pre-encrypted one; contrib/passwordcheck uses this hook. Message #32 points to credcheck as an existing tool for this need and reports little adoption of it. The `disallow` mode is therefore mostly a convenience, and the opt-in setting that message #33 asks for fits an extension. Only a core-wide default `warn` needs core, and messages #14, #28 and #32 oppose that default.

Thread: [#14 (Nathan Bossart, 2025-03-03)](https://www.postgresql.org/message-id/Z8XZ4f4XXfptmRS-%40nathan), [#28 (Robert Haas, 2025-03-19)](https://www.postgresql.org/message-id/CA%2BTgmobL3w8bytd%2BeOCcpn%3DNHqqS%2Bvyddcsimv4Yq72sqwWyYw%40mail.gmail.com), [#32 (Robert Treat, 2026-01-09)](https://www.postgresql.org/message-id/CABV9wwOrh9aEdaRQQa52_fEMzxGGfji8mL_Bi5PzqrvCyxLBXw%40mail.gmail.com), [#33 (Jakub Wartak, 2026-08-24)](https://www.postgresql.org/message-id/CAKZiRmwkv8iWwxfHjCyA5Ewq-P8zGgbLfnu%3DxNoTj8ks3Y2P8Q%40mail.gmail.com)

**Question:** With check_password_hook and credcheck already available, what does core placement of `disallow` give a DBA that a loaded module cannot?

Counter-check: held. check_password_hook does receive the password type, and the thread does not say why a module is not enough. Nathan Bossart's +1 for core enforcement in messages #12 and #14 shows some support but does not answer the question.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | **yes** | Adds the enum GUC `cleartext_passwords_action` with default `warn`. |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | **yes** | The HINT recommends `\password` when application_name is 'psql'. |
| security / privileges | **yes** | `disallow` is a password policy control, and the documentation hunk does not state who may change it. |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | A permanent GUC and a deprecation warning with no agreed removal release. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | A module on check_password_hook can already reject or warn on clear text passwords. |

## 5. Questions the author must answer before anyone reviews code

1. Will the next version default to `allow` and drop the deprecation DETAIL, and if `warn` stays, which release removes clear text passwords?
2. Which installations would set `disallow`, and what does a core GUC give them that a check_password_hook module such as credcheck does not?
3. How does `disallow` keep the rejected statement and its password out of the server log under the default `log_min_error_statement`?
4. What is the GUC context, and which roles can lower the setting from `disallow` for their own session or role?
5. How would `cleartext_passwords_action` fit with the `PASSWORD` command from message #33 without a second overlapping setting?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** none obvious. A committer who knows authentication and password handling would have to own it. Robert Haas says in message #28 that he will not commit it, and Nathan Bossart supports only the enforcement part.
- **Liability:** The owner signs for a GUC name and values that stay for years, and for a default deprecation warning with no agreed removal date.
- **Author follow-up:** The author has kept the patch rebased and moved it through six CommitFests, and offered a default of `allow` in messages #26 and #31.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs design rethink
- **Audience:** a sizeable minority
- **Maintenance burden:** medium
- **Review cost:** low
- **Why:** The patch is small and adds one GUC, but the default, the scope and the relation to the `PASSWORD` command in message #33 are all open. No committer has offered to own it until the list agrees on the overall approach.

## Appendix

### Held up in the counter-check, not listed above

- `security-1` (fixable, level 2) Disallow mode bypasses passwordcheck while verifiers still reach logs. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `security-2` (fixable, level 2) GUC context decides whether an ordinary user can turn enforcement off. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `security-3` (fixable, level 2) Rejected statements still log the clear text password by default. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `maintenance-fit-3` (fixable, level 2) The GUC may be replaced by the PASSWORD command design. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).

### Refuted in the counter-check

- `standards-compat-1` Default warn changes output for every existing clear text password user. Refuted (not specific): Duplicate of maintenance-fit-1, which makes the same point about the default `warn` and the missing removal agreement, with fuller thread support.
- `standards-compat-2` GUC context does not say who can bypass disallow. Refuted (not specific): Duplicate of security-2, which asks the same question about the GUC context and who can lower the setting.
- `standards-compat-3` Disallow makes contrib passwordcheck unable to check password strength. Refuted (not specific): Duplicate of security-1, which covers the passwordcheck conflict and adds the point about logged verifiers.
- `maintenance-fit-2` The disallow mode can already be built on check_password_hook. Refuted (not specific): Duplicate of already-exists-1, which makes the same check_password_hook argument with more thread support.
- `demand-2` Default warning costs every user for a benefit nobody has agreed on. Refuted (not specific): Duplicate of maintenance-fit-1, which states the same cost of the default `warn` and cites the same messages.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. The one listed row, [P3], is about passwordcheck GUCs and closed because it had no patch, which does not apply here. Inside the thread, message #2 points to an almost identical warning patch by Guillaume Lelarge, and message #6 says log filtering was discussed before; the material does not say how either ended, so neither supports an objection.
- standards-compat: 3 objection(s). pg_dumpall emits pre-encrypted passwords, so restores are not affected by disallow.
- replication-ondisk: 0 objection(s). The proposal adds one enum GUC checked during create/alter role. It changes no on-disk format, WAL record, catalog, wire message or extension API, and pg_dumpall output is unaffected because dumps carry pre-encrypted passwords. The PASSWORD command and pg_dumpall `\bind` in message #33 do cross the protocol boundary, but they are PoC patches posted for discussion, so they are out of scope for this review.
- security: 3 objection(s). Three objections are listed above. The `warn` mode and the hint based on application_name add no privilege or exposure issue.
- maintenance-fit: 3 objection(s). Committability: no committer has offered to own this. Robert Haas says he will not commit it (message #28), and Nathan supports only the enforcement part. The author has kept the patch rebased for six CommitFests.
- already-exists: 1 objection(s). Only one objection from this angle: the enforcement half can already be done through an existing hook and extension.
- demand: 2 objection(s). The problem of passwords in logs has real demand (messages #19, #33). The objections are about demand for this mechanism and its default.

### Precedents retrieved

- P1: [Support multiple password-based authentication tokens concurrently](https://commitfest.postgresql.org/patch/3748/) (cf, Returned with feedback, 2022-10-12) - judged not relevant; reason summarised from its thread
- P2: [Multiple passwords, interval expirations](https://commitfest.postgresql.org/patch/4432/) (cf, Returned with feedback, 2024-02-01) - judged not relevant; reason summarised from its thread
- P3: [Enhancements to passwordcheck](https://commitfest.postgresql.org/patch/1309/) (cf, Returned with feedback, 2017-11-02) - used in the review; reason summarised from its thread
- P4: [Pg14, pg_dumpall and "password_encryption=true"](https://commitfest.postgresql.org/patch/2939/) (cf, Withdrawn, 2021-03-04) - judged not relevant; reason summarised from its thread
- P5: [add DETAIL for wrong SCRAM password](https://commitfest.postgresql.org/patch/3022/) (cf, Returned with feedback, 2021-04-08) - judged not relevant; reason summarised from its thread
- P6: [Password identifiers, protocol aging and SCRAM protocol](https://commitfest.postgresql.org/patch/535/) (cf, Returned with feedback, 2016-03-22) - judged not relevant; reason summarised from its thread
- P7: [pg_stat_statements and extended query protocol](https://commitfest.postgresql.org/patch/4146/) (cf, Withdrawn, 2023-01-30) - judged not relevant; reason summarised from its thread
- P8: [Refactoring of connection with password prompt loop for frontends](https://commitfest.postgresql.org/patch/2248/) (cf, Returned with feedback, 2019-09-27) - judged not relevant; reason summarised from its thread
- P9: [Add WAL recovery info such as current WAL file and WAL source to server logs](https://commitfest.postgresql.org/patch/3443/) (cf, Withdrawn, 2022-10-26) - judged not relevant
- P10: [GUC to disable cancellation of awaiting for synchronous replication](https://commitfest.postgresql.org/patch/2402/) (cf, Returned with feedback, 2021-10-06) - judged not relevant
- P11: [Remove deprecated exclusive backup mode](https://commitfest.postgresql.org/patch/1913/) (cf, Withdrawn, 2021-03-19) - judged not relevant
- P12: [Allow placeholders in ALTER ROLE w/o superuser](https://commitfest.postgresql.org/patch/3749/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P13: [pg_stat_statements with plans (v02)](https://commitfest.postgresql.org/patch/1470/) (cf, Returned with feedback, 2018-10-02) - judged not relevant
- P14: [remove deprecated v8.2 containment operators](https://commitfest.postgresql.org/patch/2798/) (cf, Returned with feedback, 2021-02-01) - judged not relevant
- P15: [ALTER ROLE SET/RESET for multiple options](https://commitfest.postgresql.org/patch/486/) (cf, Rejected, 2016-02-18) - judged not relevant
- P16: [Avoid use deprecated Windows Memory API](https://commitfest.postgresql.org/patch/3893/) (cf, Rejected, 2023-03-20) - judged not relevant
- P17: [Extended protocol COPY FROM STDIN fix](https://commitfest.postgresql.org/patch/3180/) (cf, Returned with feedback, 2021-12-01) - judged not relevant
- P18: [pg_stat_statements with plans](https://commitfest.postgresql.org/patch/969/) (cf, Returned with feedback, 2017-03-04) - judged not relevant
- P19: [Data at rest encryption](https://commitfest.postgresql.org/patch/2104/) (cf, Returned with feedback, 2019-11-25) - judged not relevant
- P20: [Sample values for pg_stat_statements](https://commitfest.postgresql.org/patch/1460/) (cf, Returned with feedback, 2019-01-31) - judged not relevant
- P21: [Revert "Replace use of deprecated Python module distutils.sysconfig"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=dda42ff8e60541e9eb5ac21b79af32d914f408a8) (revert, Reverted, 2022-01-18) - judged not relevant
- P22: [Add control-C support for some psql meta-commands](https://commitfest.postgresql.org/patch/3412/) (cf, Withdrawn, 2021-11-15) - judged not relevant
- P23: [If pg_hba.conf has changed since last reload, emit a HINT to the client and server log](https://commitfest.postgresql.org/patch/50/) (cf, Returned with feedback, 2015-04-30) - judged not relevant
- P24: [Sampling of queries in pg_stat_statements](https://commitfest.postgresql.org/patch/5389/) (cf, Withdrawn, 2024-11-19) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 12; wall time: 152 s; cost reported by the CLI: $2.52
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 7 from cache, 1 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 112 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:33:26.728Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

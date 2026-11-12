# Design pre-review: pg_dump: skip pg_init_privs entries for non-existent roles

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6915](https://commitfest.postgresql.org/patch/6915/); [thread](https://www.postgresql.org/message-id/flat/19483-80de42dc4e62cfd6%40postgresql.org), 35 of 35 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No objection survived at levels 1a or 1b: the patch fixes a reproducible pg_upgrade failure and follows the pg_dump direction suggested in message #4. Two fixable design points remain at level 2: standards-compat-1 (dropping an aclitem whose grantor alone is dangling causes lasting dump drift) and maintenance-fit-1 (no server version gate on the filter). Neither blocks the idea, but a committer who knows pg_dump's ACL code has to accept the trade-off and the backpatch range.

Not checked: Without the code, I could not check that the filter covers every query that reads pg_init_privs, nor how often the per-table column ACL query runs. The behaviour against v15 and v16 source servers is untested in the material.

## Prior attempts

- [pg_upgrade fails with non-standard ACL](https://www.postgresql.org/message-id/flat/f85991ad-bbd4-ad57-fde4-e12f0661dbf0@postgrespro.ru) (similar idea; Returned with feedback, 2021-11-26). "I'm not generally enthusiastic about adding pg_upgrade code to predict whether the dump will fail to restore, because such code will never be as good as just trying the restore." ([message](https://www.postgresql.org/message-id/20210127112135.GB2481576%40rfd.leadboat.com)) Different now: P3 had pg_upgrade predict restore failures from ACLs and ask the user to run a cleanup script. This proposal dropped the pg_upgrade check from message #1 and makes pg_dump skip the dangling aclitems, so the restore itself succeeds.

## Objections a reviewer is likely to raise

### 1. Dropping whole aclitems leaves upgraded extensions with permanent dump drift

A DBA upgrades a v15 cluster whose extension owner was dropped. The new cluster has no pg_init_privs row for those members, so every later pg_dump emits REVOKE and GRANT lines after CREATE EXTENSION. A restore against a newer extension version where a member was renamed then fails, and schema-diff tools report privilege changes nobody made.

The filter drops an aclitem when only its grantor is dangling, so a valid PUBLIC grant such as =r/16384 also leaves pg_init_privs (messages #33, #35). The patch now documents this drift, but documenting it does not remove it. A committer would likely ask why the patch drops these items instead of mapping the dangling role to the object owner. As far as I know that mapping is closer to what REASSIGN OWNED does on v17 and later, and it would match a fresh install.

Precedent: [pg_upgrade fails with non-standard ACL](https://www.postgresql.org/message-id/flat/f85991ad-bbd4-ad57-fde4-e12f0661dbf0@postgrespro.ru) (Returned with feedback, 2021-11-26)

**Question:** Why drop items with a dangling grantor or grantee instead of substituting the object owner, so upgraded extension members dump the same as a fresh install?

<sub>fixable, level 2, pg_dump / restore round-trip; messages [#4](https://www.postgresql.org/message-id/104812.1780847519%40sss.pgh.pa.us), [#19](https://www.postgresql.org/message-id/1030965.1782313078%40sss.pgh.pa.us), [#33](https://www.postgresql.org/message-id/CAHWVJhGQ5qGHnxYqbDEkj1WdXosbR0xk_ZdaY5tK7y0HHhWvhg%40mail.gmail.com), [#34](https://www.postgresql.org/message-id/CAB5wL7ZXt0h1F4pQfKERUxcHhkKUMYEMm%3Dy9%2BJe7eWTLjrLTPg%40mail.gmail.com), [#35](https://www.postgresql.org/message-id/CAHWVJhGamC1wN5LB6b5XAKwQBxqx1wtM-GaYq2CJXZVNa4MF7w%40mail.gmail.com); `standards-compat-1`</sub>

### 2. Filter runs on every dump, including from v17+ servers where it cannot match

Someone running HEAD pg_dump against a v19 server with many tables pays for the SAFE_INITPRIVS subquery in the per-table column ACL query on every table, although dangling rows cannot exist there. Years later, a developer changing the pg_init_privs queries cannot tell whether the macro is still needed. Nothing fails if the filter is dropped or missing from a new query site.

Message #4 and message #32 show dangling rows only come from pre-v17 servers, yet v6 to v9 apply the filter without a remoteVersion check. Message #23 objected to the per-table subquery, and Tom Lane, who raised the cost concern in message #14, gives no view on v6 or later. The only measurement, in message #26, used empty tables with no pg_init_privs rows and runs that varied between 19 and 25 minutes, so it does not exercise the new path. A version gate would limit both the cost and the scope of the code kept in HEAD.

**Question:** Why not apply SAFE_INITPRIVS only when the source server is older than v17, so HEAD dumps of modern servers skip the per-table subquery?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#4](https://www.postgresql.org/message-id/104812.1780847519%40sss.pgh.pa.us), [#14](https://www.postgresql.org/message-id/592839.1782153054%40sss.pgh.pa.us), [#23](https://www.postgresql.org/message-id/daee495807e183ea6f48c3c0844239753d7f9f6d.camel%40cybertec.at), [#26](https://www.postgresql.org/message-id/ec8f6c8008cf88577b14489b68fef971f16960ab.camel%40aon.at), [#32](https://www.postgresql.org/message-id/CAF0whuft4HjO1NhMuir-fJ-mcAWbX5m4ZG7FuARcLuxJzgK2_Q%40mail.gmail.com); `maintenance-fit-1`</sub>

## Questions for the author

1. Why does the filter drop an aclitem whose grantor alone is dangling, such as =r/16449, instead of keeping the grant with the object owner as grantor?
2. Why is SAFE_INITPRIVS not applied only when the source server is older than v17, given that message #4 says later servers cannot have dangling rows?
3. What is the pg_dump-only time with and without the patch on a database with many tables that have column ACLs and pg_init_privs rows?
4. Which branches should get the fix, and does a test run pg_dump against a v15 or v16 source server that has dangling rows?
5. Does the pg_dump fix depend on the HEAD-only putid() quoting change, and has that patch been posted?

## Touch points

- **on-disk format / pg_upgrade:** The fix changes what pg_upgrade restores; affected extension members lose their pg_init_privs row in the new cluster.
- **catalog changes:** pg_dump changes how it reads pg_init_privs and checks role OIDs against pg_roles; no catalog definition changes.
- **pg_dump / restore round-trip:** pg_dump drops dangling aclitems, so later dumps of upgraded extension members re-emit REVOKE and GRANT lines.
- **security / privileges:** The patch decides which initial privileges survive a dump; it filters by OID against pg_roles to avoid a name-matching risk (messages #17, #18).
- **backpatchability:** The author proposes a backpatch to v15, but the CF target is 19 and the range is not agreed.
- **maintenance burden ("who maintains this in 10 years"):** The SAFE_INITPRIVS macro stays in two pg_dump query sites with no server version gate.
- **performance on the hot path:** The filter subquery is added to the per-table column ACL query in dumpTable().

## Who would commit it, and when

- Owner: silent. Needs a committer who knows pg_dump's ACL and pg_init_privs handling. Tom Lane raised the cost concern and the putid() idea, but shows no view on v6 to v9.
- Priority: needs champion; audience niche; review cost low. The patch touches one pg_dump source file plus a test and fixes a hard pg_upgrade failure for clusters with extensions installed by later-dropped roles. Two reviewers report no open items, so the entry waits for a committer in this area to settle the two design points and the backpatch range.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 1 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

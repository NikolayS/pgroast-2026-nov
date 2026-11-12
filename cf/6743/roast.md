# Design pre-review: Preserve replication origin OIDs in pg_upgrade

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6743](https://commitfest.postgresql.org/patch/6743/); [thread](https://www.postgresql.org/message-id/flat/CAFPTHDa5aP8ixTZBygGAe48E1N%3DDvuPzcXikw7bNKixLMN3pVg%40mail.gmail.com), 90 of 90 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). No surviving objection is fatal, and a committer who knows the area took part in the design. Three fixable points remain. demand-1: the main benefit depends on a separate pg_commit_ts migration whose status the thread does not give. replication-ondisk-2: upgrades from clusters before 17 are now refused in cases that pass on HEAD. replication-ondisk-1: the check that justifies those refusals can be bypassed while apply workers run. A committer has to decide the pre-17 behaviour change.

Not checked: Without the code, this review could not check WAL logging of restored origin progress, locking in binary_upgrade_create_replication_origin(), or the binary-upgrade guard on the new functions. Only 3 of 90 messages were read directly.

## Prior attempts

- [Patch for migration of the pg_commit_ts directory](https://www.postgresql.org/message-id/flat/182311743703924@mail.yandex.ru) (related; Withdrawn, 2025-10-15). reason not recorded Different now: This proposal does not migrate pg_commit_ts. It preserves the roident values that such a migration needs, and it also keeps origins that are not owned by subscriptions.

## Objections a reviewer is likely to raise

### 1. Main benefit depends on a pg_commit_ts migration not yet accepted

A committer pushes both patches, but the pg_commit_ts migration does not land in the same release. Upgrades from clusters before 17 that worked before are now rejected for missing origins or relations not in 'i' or 'r' state. The roident preservation that justified this change protects no data in that release.

Messages #1 and #26, and the 0002 commit message, give one motivation: keeping roident stable so a copied pg_commit_ts stays valid. Amit Kapila notes in message #26 that the roident value only matters from PG19 onward. Message #2 says subscription OID preservation was declined before for lack of a strong motivation, and this patch borrows its motivation from a separate patch whose status the thread does not give. The other benefit, keeping origins made by external tools, has no user report in the thread.

**Question:** Is the pg_commit_ts migration patch agreed for the same release, or which users of external replication tools have reported losing origins in pg_upgrade?

<sub>fixable, level 1b, demand; messages [#1](https://www.postgresql.org/message-id/CAFPTHDa5aP8ixTZBygGAe48E1N%3DDvuPzcXikw7bNKixLMN3pVg%40mail.gmail.com), [#2](https://www.postgresql.org/message-id/OS9PR01MB12149E188221BED862E4BE9E0F5342%40OS9PR01MB12149.jpnprd01.prod.outlook.com), [#26](https://www.postgresql.org/message-id/CAA4eK1KzGSANd%3Dg1S8xpcyiwh3M-c%2BUaa4%2B447v-%2BN8chX7K3g%40mail.gmail.com), [#90](https://www.postgresql.org/message-id/CAFPTHDYCdy%2BGXTOO8ApS9oXOFWXsh8uxbB4j%2BP_3wNp3A7DBZA%40mail.gmail.com); `demand-1`</sub>

### 2. New pg_upgrade refusals for pre-17 clusters where filtering would do

An operator upgrades a PG14 subscriber whose table sync is stuck in 'd' state, or one whose origin was dropped by hand. HEAD upgrades it and restores the subscription disabled. With v22 the check refuses the upgrade, and the operator must fix the old cluster first.

These refusals exist only to keep tablesync origins and origin-less subscriptions from causing trouble after migration (messages #88, #90). For pre-17 sources, pg_subscription_rel is not migrated and subscriptions come back disabled, so origins named pg_<suboid>_<relid> carry no state the new cluster can use. Skipping those origins at dump time would keep the current pre-17 behaviour. Message #90 calls the refusal harmless, but no committer has weighed it, and the patch has no pre-17 upgrade test.

**Question:** Why reject pre-17 clusters in these states instead of leaving out tablesync origins and recreating a missing subscription origin, as HEAD does today?

<sub>fixable, level 2, on-disk format / pg_upgrade; messages [#83](https://www.postgresql.org/message-id/CABdArM7iWpoY%3DZM3Bsbbgwc3N5RbP6asS%2BZYyRzZtfC3tik4mA%40mail.gmail.com), [#88](https://www.postgresql.org/message-id/OS7PR01MB1831748150E960602D81EE47CF56D2%40OS7PR01MB18317.jpnprd01.prod.outlook.com), [#89](https://www.postgresql.org/message-id/CAJpy0uD3iXD7Qf%2BBkBosByOW3D_nQ-%2B89J%2BQ0jhwoCmsRL6uRA%40mail.gmail.com), [#90](https://www.postgresql.org/message-id/CAFPTHDYCdy%2BGXTOO8ApS9oXOFWXsh8uxbB4j%2BP_3wNp3A7DBZA%40mail.gmail.com); `replication-ondisk-2`</sub>

### 3. Tablesync origin leak check is racy for old clusters before 17

A user upgrades a PG16 subscriber that has a table in 'i' state, which the check accepts. While pg_upgrade runs, the old apply worker launches a tablesync worker, which creates origin pg_<suboid>_<relid> before pg_dumpall reads pg_replication_origin. That origin is migrated, and later DROP SUBSCRIPTION leaves it behind.

Message #85 states that apply workers can run during pg_upgrade when the source is PG16 or older. Message #88 adds the state check because migrated tablesync origins cannot be dropped once pg_subscription_rel is not carried over. The v22 commit message allows relations in 'i' state. A check taken before the dump does not bound what the dump later sees while workers keep running.

**Question:** For sources before 17, what stops a tablesync origin from being created between check_old_cluster_subscription_state() and pg_dumpall, or would it be safer not to dump pg_<suboid>_<relid> origins from such sources?

<sub>fixable, level 2, on-disk format / pg_upgrade; messages [#85](https://www.postgresql.org/message-id/OS7PR01MB18317CFD00457D42BDBE4B841F59C2%40OS7PR01MB18317.jpnprd01.prod.outlook.com), [#86](https://www.postgresql.org/message-id/CAFPTHDai3JOZ%3Du%2Bw0MPhZ4kxrBiLxh%3DH%3D8ZW9ZdqeT6NVpja_w%40mail.gmail.com), [#88](https://www.postgresql.org/message-id/OS7PR01MB1831748150E960602D81EE47CF56D2%40OS7PR01MB18317.jpnprd01.prod.outlook.com), [#90](https://www.postgresql.org/message-id/CAFPTHDYCdy%2BGXTOO8ApS9oXOFWXsh8uxbB4j%2BP_3wNp3A7DBZA%40mail.gmail.com); `replication-ondisk-1`</sub>

## Questions for the author

1. Is the pg_commit_ts migration patch agreed for the same release, and if it does not land, which users need preserved roident values?
2. Why reject pre-17 clusters with relations not in 'i' or 'r' state instead of leaving pg_<suboid>_<relid> origins out of the dump for such sources?
3. Why reject a pre-17 subscription with a missing origin instead of creating a new origin for it, as HEAD does today?
4. For sources before 17, what stops an apply worker from creating a tablesync origin between check_old_cluster_subscription_state() and pg_dumpall?
5. Which external replication tools or users have reported that pg_upgrade loses their replication origins?

## Touch points

- **on-disk format / pg_upgrade:** pg_upgrade now preserves pg_subscription.oid and roident, and adds new checks on old and new clusters.
- **WAL / replication / standby:** Replication origins and their remote_lsn are migrated, which changes where pre-17 subscriptions resume.
- **catalog changes:** pg_proc.dat gains two binary-upgrade support functions and loses binary_upgrade_replorigin_advance().
- **ABI / extension API:** The patch changes src/include/replication/origin.h and removes an existing support function.
- **pg_dump / restore round-trip:** pg_dumpall --binary-upgrade emits all replication origins as global objects, and pg_dump emits the subscription OID.
- **backpatchability:** Upgrades from released versions before 17 are now rejected in states that HEAD accepts.
- **maintenance burden ("who maintains this in 10 years"):** Origin migration relies on the pg_<suboid> and pg_<suboid>_<relid> naming conventions and on pg_upgrade checks.

## Who would commit it, and when

- Owner: qualified committer engaged. Amit Kapila steered the design in messages #26 and #51, but has not said the patch is ready. The owner needs to know logical replication origins and pg_upgrade of subscribers.
- Priority: proceed; audience niche; review cost medium. The patch helps subscribers that use pg_upgrade and need stable origin IDs, and users of external tools that create origins. It runs only in binary-upgrade mode, spans pg_dump, pg_upgrade and origin.c, and has an engaged committer and active authors.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 6 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

# Design pre-review: pg_resetwal: add --cluster-state option

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7394](https://commitfest.postgresql.org/patch/7394/); [thread](https://www.postgresql.org/message-id/flat/CAGRkXqRH%3DxzB7svv-xFRF%3D6Lexo8tX0T4XX207uPAwuLys-Kcw%40mail.gmail.com), 5 of 5 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The proposal stops at level 1b on demand. Message #1 gives one use, emptying unlogged tables after a reset, and its question about other uses has no answer in the thread. A start, an immediate stop and a second start already reach that result (already-exists-1). Four of the five non-default values have no stated use (demand-2).

Not checked: Without the code, this review could not check how startup treats each hand-set state or how the option interacts with other pg_resetwal options. Messages #4 and #5 were available only through the brief and the objections.

## Prior attempts

- [Add DB_IN_END_OF_RECOVERY_CHECKPOINT state for control file](https://www.postgresql.org/message-id/flat/CALj2ACVvy+06hv7HocOFAHEBZWxLbc-t_61axdCUq9NaR2-c0Q@mail.gmail.com) (related; Withdrawn, 2022-02-27). "This patch would bring some extra visibility for something that's not incorrect either on HEAD, as end-of-recovery checkpoints are the same things as shutdown checkpoints." ([message](https://www.postgresql.org/message-id/YhmaUxDMemv5hmba%40paquier.xyz)) Different now: P5 changed which state the server writes during the end-of-recovery checkpoint. This patch lets a user write any state offline, so startup's dependence on the stored state matters for every accepted value.

## Objections a reviewer is likely to raise

### 1. Existing idioms already reset unlogged tables after pg_resetwal

A DBA resets a crashed cluster and wants unlogged tables emptied. Today they can start the server with listen_addresses = '', run pg_ctl stop -m immediate, and start again, which runs crash recovery and resets unlogged relations. The project would then keep a six-value option, docs and tests that only save this extra restart.

Message #1 shows one use case, emptying unlogged tables, and the author asks Michael for another, which has not arrived. An immediate stop already leaves the state in-production, so the next start runs crash recovery, the same path the option reaches. Running TRUNCATE on each unlogged table is a second existing route. The existing ways cost more steps, but they do the job, so the option adds convenience and no new capability.

Precedent: [pg_replslotdata - a tool for displaying replication slot information](https://www.postgresql.org/message-id/flat/CALj2ACW0rV5gWK8A3m6_X62qH+Vfaq5hznC=i0R5Wojt5+yhyw@mail.gmail.com) (Rejected, 2022-01-17)

**Question:** What does --cluster-state do that pg_resetwal followed by a start and pg_ctl stop -m immediate cannot, and who needs it?

<sub>fixable, level 1b, doesn't fit what Postgres is about (extension / external tool / already solved); messages [#1](https://www.postgresql.org/message-id/CAGRkXqRH%3DxzB7svv-xFRF%3D6Lexo8tX0T4XX207uPAwuLys-Kcw%40mail.gmail.com), [#4](https://www.postgresql.org/message-id/CAGRkXqSPa3hzHgXLXgGvKEiWFCeTq5UoPTrxYQLVYpAXu0smCw%40mail.gmail.com); `already-exists-1`</sub>

### 2. Five of the six state values have no stated use

A DBA reads the docs and picks in-archive-recovery or shut-down-in-recovery on a standby, because the docs list them as valid. Startup does something the docs do not describe, and nobody in the thread can say what the intended use was. Each new DBState value then needs a matching option value with no user behind it.

The one example in message #1 needs only some state other than shut-down, and in-production gives that. The patch still exposes all six values, and the v2 tests cover only in-production and a bad value. Each extra value adds documented behaviour that the project must explain and keep with no demand to justify it.

**Question:** What concrete case needs each of the five values other than shut-down, and would a single "force crash recovery" switch cover the known need?

<sub>fixable, level 1b, desirability; messages [#1](https://www.postgresql.org/message-id/CAGRkXqRH%3DxzB7svv-xFRF%3D6Lexo8tX0T4XX207uPAwuLys-Kcw%40mail.gmail.com), [#4](https://www.postgresql.org/message-id/CAGRkXqSPa3hzHgXLXgGvKEiWFCeTq5UoPTrxYQLVYpAXu0smCw%40mail.gmail.com), [#5](https://www.postgresql.org/message-id/CAG7L5%2B7q7-Gq5OEWaWYx7%2BeP7Qej88k9F7BP7c7u-C2_w%3D3w6g%40mail.gmail.com); `demand-2`</sub>

## Questions for the author

1. Which use case, other than emptying unlogged tables, needs pg_resetwal to store a state other than shut-down?
2. What does --cluster-state do that pg_resetwal followed by a start, pg_ctl stop -m immediate and a second start cannot, and who needs it?
3. Which concrete case needs each of shut-down-in-recovery, shutting-down, in-crash-recovery and in-archive-recovery, or would a single switch that forces crash recovery be enough?
4. What does the next start do differently for each of the five non-default states after a reset?
5. How do in-archive-recovery and shut-down-in-recovery behave on a standby, or with standby.signal or recovery settings present?

## Touch points

- **on-disk format / pg_upgrade:** The option writes a user-chosen value into the state field of pg_control; the file format does not change.
- **WAL / replication / standby:** A non-default state makes the next start run crash recovery, and two values name standby and archive recovery states.
- **maintenance burden ("who maintains this in 10 years"):** Six documented option values map one-to-one onto the internal DBState enum and must stay valid across releases.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** A start, pg_ctl stop -m immediate and a second start already empty unlogged tables after a reset.

## Who would commit it, and when

- Owner: silent. It needs a committer who knows the startup and recovery code and how it reads the pg_control state. None has posted in the visible thread.
- Not now, show demand first: A stated use case that the start, immediate stop and start sequence cannot serve, such as the motivation behind the original suggestion or a field report. For the extra values, one concrete case per state, or a reduction to a single switch that forces crash recovery.
- Priority: needs design rethink; audience niche; review cost low. The patch is small and confined to pg_resetwal, but it serves one rare post-reset case and exposes six enum values where one is used. A single switch that forces crash recovery, with a stated use case, would be cheaper to carry.

---

[roast-full.md](roast-full.md) has 2 more objection(s) that held, 5 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

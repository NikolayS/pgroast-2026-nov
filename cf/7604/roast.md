# Design pre-review: Include schema-qualified names in publication error messages (generalised to all of LR).

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7604](https://commitfest.postgresql.org/patch/7604/); [thread](https://www.postgresql.org/message-id/flat/CAFiTN-scG7b11Jsp%2BVoDRT8ZFE84eSKLcDsSB18dZ8AaP%3DR-mw%40mail.gmail.com), 39 of 39 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No surviving objection questions the goal, and the demand objections were refuted. Three fixable design points remain at level 2: no written rule for which messages qualify names (prior-attempts-1), the unstated fate of the pushed RelationGetQualifiedRelationName() (maintenance-fit-1), and the open pg_temp versus pg_temp_N choice (maintenance-fit-3). None needs a different proposal, but a committer in this area has to settle them. The code was not read.

Not checked: Without the code, I could not check whether patch 0001 leaves one helper or two for message use, or which call sites remain unqualified. The full list of converted messages was not shown.

## Prior attempts

- [pg_get_constraintdef: Schema qualify foreign tables unless pretty printing is enabled](https://www.postgresql.org/message-id/flat/CAP53PkyRA9z5Sg2WKgTZ1NmTLcqA5VR_MRAWFcUWrjAjMwwcmw@mail.gmail.com) (related; Returned with feedback, 2022-11-30). "In other words, I think your patch is too small." ([message](https://www.postgresql.org/message-id/20220810085850.ghwvrkcqzimif5ok%40alvherre.pgsql)) Different now: P5 changed deparsed SQL at one ruleutils.c call site. This patch changes error and log message text through shared helpers across many call sites, but it defines its scope by subsystem.
- [Explicit relation name in VACUUM VERBOSE log](https://www.postgresql.org/message-id/flat/CAD21AoCtiAHCmtNQV2o8FjhEEEKSjRYCgzBOo-uzq86h96Nvhw@mail.gmail.com) (related; Returned with feedback, 2017-10-02). "Repeating the full table name in every line will be really annoying for that primary use-case." ([message](https://www.postgresql.org/message-id/14199.1503585323%40sss.pgh.pa.us)) Different now: P1 qualified names on every VACUUM VERBOSE line read in a client. This patch changes single-line error and log messages, where the name is not repeated.

## Objections a reviewer is likely to raise

### 1. Two relation-name helpers with opposite quoting rules

A later patch author needs a qualified name for an errmsg and finds RelationGetQualifiedRelationName() next to MsgQualifiedRelNameByRel() in relcache.c. They pick the quoting one inside "%s", and the double-quote bug fixed by 926627bf902 comes back. No test or build check catches it.

Message #28 placed a quoting helper in relcache.c, and message #31 pushed it. Patch 0001 adds three non-quoting helpers in the same file, and 0002 removes only get_qualified_objname(). The proposal does not say whether RelationGetQualifiedRelationName() is kept, renamed or replaced. Two near-identical helpers that differ only in quoting are the API shape most likely to be misused.

**Question:** Is RelationGetQualifiedRelationName() removed or redefined by this series, and how would a new caller know which helper fits an errmsg?

<sub>fixable, level 2, ABI / extension API; messages [#28](https://www.postgresql.org/message-id/CAA4eK1KL809TGPcK1%3DbFOs3i44Htxv9Cv88MBiO5taoSchPXGw%40mail.gmail.com), [#31](https://www.postgresql.org/message-id/CAA4eK1KsY8qppUKTZ9gRALCojJrj%2BCM9gLoDxj6CViS99S9JCA%40mail.gmail.com), [#36](https://www.postgresql.org/message-id/CALDaNm2dPWz8_GcRzvhX5pk%2BgSVnU_uDTQ9wkzZsUYRn3WeYrg%40mail.gmail.com), [#37](https://www.postgresql.org/message-id/CAHut%2BPsQ-77Mw5Ujj293kgo1GGZwfUhx90K1XgqePAHQet%2BtMw%40mail.gmail.com); `maintenance-fit-1`</sub>

### 2. Logical replication boundary gives no rule for which messages qualify names

A user runs `alter table` on a published table and gets a tablecmds.c error naming "public.t". A later error from the same command, outside logical replication, names it as "t". A future patch adds a new logical replication message with RelationGetRelationName(), and no test or check catches it.

[P5] was returned after reviewers asked for a defined rule on which call sites qualify names, backed by a documented helper; its reason is an AI summary, so it is only a pointer. Here the patch converts messages in shared files such as tablecmds.c, objectaddress.c and execReplication.c because they mention publications, not because of a general rule. The brief notes there is no statement for messages outside logical replication, and message #33 shows matview errors with the same double-quoting left as is. Message #38 lists four kinds of call sites kept unqualified by judgement.

Precedent: [pg_get_constraintdef: Schema qualify foreign tables unless pretty printing is enabled](https://www.postgresql.org/message-id/flat/CAP53PkyRA9z5Sg2WKgTZ1NmTLcqA5VR_MRAWFcUWrjAjMwwcmw@mail.gmail.com) (Returned with feedback, 2022-11-30)

**Question:** What written rule tells a future patch author whether a new message must use MsgQualifiedRelName(), and why does it stop at logical replication?

<sub>fixable, level 2, prior rejection; messages [#33](https://www.postgresql.org/message-id/CALDaNm17Vh8OccJDD97%3D88R5fSyo-BtJ4aKkfMtkGSDGsYDWEA%40mail.gmail.com), [#37](https://www.postgresql.org/message-id/CAHut%2BPsQ-77Mw5Ujj293kgo1GGZwfUhx90K1XgqePAHQet%2BtMw%40mail.gmail.com), [#38](https://www.postgresql.org/message-id/OS7PR01MB183177A958E31DC4239D23BA2F59B2%40OS7PR01MB18317.jpnprd01.prod.outlook.com); `prior-attempts-1`</sub>

### 3. pg_temp_N makes temp-table messages untestable by text

A regression test hits one of the converted messages with a temporary table. The output contains pg_temp_N, which differs by backend, so the test fails at random. The test then falls back to VERBOSITY sqlstate and stops checking the message text.

Message #36 already had to switch a publication test to sqlstate-only output for this reason. Patch 0001 extends the same behaviour to about 20 messages and keeps pg_temp_N. Message #38 asked for pg_temp, and the thread has no decision. Each future test on these paths carries the same workaround.

**Question:** Which display of temp schemas is chosen, and how do regression tests for the converted messages keep checking message text?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#36](https://www.postgresql.org/message-id/CALDaNm2dPWz8_GcRzvhX5pk%2BgSVnU_uDTQ9wkzZsUYRn3WeYrg%40mail.gmail.com), [#37](https://www.postgresql.org/message-id/CAHut%2BPsQ-77Mw5Ujj293kgo1GGZwfUhx90K1XgqePAHQet%2BtMw%40mail.gmail.com), [#38](https://www.postgresql.org/message-id/OS7PR01MB183177A958E31DC4239D23BA2F59B2%40OS7PR01MB18317.jpnprd01.prod.outlook.com); `maintenance-fit-3`</sub>

## Questions for the author

1. Does the series keep, rename or remove RelationGetQualifiedRelationName(), and how does a new caller know which helper fits a message that already supplies quotes?
2. What written rule tells a future patch author whether a new message must use MsgQualifiedRelName(), and where is that rule documented?
3. Do the converted messages print pg_temp or pg_temp_N for temporary tables, and how do regression tests keep checking the message text?
4. Which logical replication messages still print unqualified names after patch 0001, and what is the reason for each exception listed in message #38?
5. Should messages outside logical replication, such as the matview errors in message #33, follow the same rule in a later patch?

## Touch points

- **ABI / extension API:** Patch 0001 adds three MsgQualifiedRelName*() helpers in relcache.c, and patch 0002 removes get_qualified_objname().
- **backpatchability:** The text of about 20 existing messages changes, and message #19 objects to backpatching; the target is v20.
- **maintenance burden ("who maintains this in 10 years"):** Future logical replication messages must use the new helpers, and nothing fails when a new message prints a bare name.

## Who would commit it, and when

- Owner: qualified committer engaged. Amit Kapila pushed the first part (message #31) but has not reviewed the generalised patches. The owner needs to know logical replication messages and the relcache.c helper conventions.
- Priority: proceed; audience a sizeable minority; review cost low. The change helps logical replication users who have the same relation name in several schemas, and it touches only error and log paths. The three open design points are small and can be settled in the existing thread.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 6 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

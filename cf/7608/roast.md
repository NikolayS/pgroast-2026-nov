# Design pre-review: Add pg_stat_progress overview view

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7608](https://commitfest.postgresql.org/patch/7608/); [thread](https://www.postgresql.org/message-id/flat/CAFp7QwqYGVqF7LO6fDTthpaskRYNPaVXSPXKRQOrYT%3DXMe8kpw%40mail.gmail.com), 1 of 1 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: low). The decisive objection is demand-1. Message #1 names no user, tool or report that needs the overview, and a user-defined `union all` over the existing progress views returns the same rows. The cost of the view is low, so this is "not now, show demand" and no rejection. The verdict rests on absence of evidence in a one-message thread, so confidence is low.

Not checked: Without the code I could not check how the view lists the command families, whether the new TAP test catches a missing command type, or whether the view keeps the existing filtering of other users' details.

## Prior attempts

None found before 2026-11-07.

## Objections a reviewer is likely to raise

### 1. No demand shown beyond the author's wish for one overview

The view is added, and every later progress command must update its definition and docs. Monitoring tools already query the detailed pg_stat_progress_* views they care about, so few of them switch to it. The project carries the upkeep for a view that saves users one union query.

Message #1 gives the problem as an inconvenience and cites no field report, -general thread or tool that asked for this, and no other database's equivalent. A user can write the same three-column result today as a union over the seven detailed views, or over pg_stat_get_progress_info() per command, in a user view. The thread has no replies yet, so nobody else has said they need it. The cost is small, and the benefit is smaller still.

**Question:** Which monitoring tools or users need this overview today, and why does a user-defined union of the existing progress views not serve them?

<sub>fixable, level 1b, demand; messages [#1](https://www.postgresql.org/message-id/CAFp7QwqYGVqF7LO6fDTthpaskRYNPaVXSPXKRQOrYT%3DXMe8kpw%40mail.gmail.com); `demand-1`</sub>

## Questions for the author

1. Which monitoring tools or users need this overview today, and why does a user-defined `union all` over the existing pg_stat_progress_* views not serve them?
2. Why does a join of pg_stat_activity with the detailed progress views not already meet the need?
3. What makes the build or the regression tests fail when a new progress command type is added but pg_stat_progress is not updated?
4. Why does `pg_stat_progress.command` hold family names when `command` in the detailed progress views holds the command the user ran?
5. What use case needs progress_view as regclass when the command name already identifies the detailed view?

## Touch points

- **catalog changes:** The patch adds the system view pg_stat_progress in system_views.sql and bumps catversion.
- **security / privileges:** The documentation says pid, command and progress_view are visible to all users.
- **maintenance burden ("who maintains this in 10 years"):** The view lists each command family by hand, so each new progress command must also edit it.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** A user-defined `union all` over the existing pg_stat_progress_* views returns the same pid and command list.

## Who would commit it, and when

- Owner: silent. A committer who knows the progress reporting code and the statistics views in system_views.sql would have to own it. Nobody has replied in the thread.
- Not now, show demand first: A report from a monitoring tool author or from users who need one list of all progress-reporting backends, with a reason why a user-defined union of the existing views is not enough. Support from other list members in the thread would also count.
- Priority: park; audience unclear; review cost low. The patch is about 13 KiB, with one SQL view, documentation and a TAP test, and no backend changes, so review and upkeep are cheap. Message #1 shows no user or tool that needs it, so it can wait until demand is shown.

---

[roast-full.md](roast-full.md) has 2 more objection(s) that held, 2 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

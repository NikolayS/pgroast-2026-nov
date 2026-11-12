# Design pre-review: Avoid building stderr message if possible

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7106](https://commitfest.postgresql.org/patch/7106/); [thread](https://www.postgresql.org/message-id/flat/tencent_A00A556AC153E46588F0680E34ED4C43EF09%40qq.com), 3 of 3 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). No surviving objection is fatal, and no level fails. Message #3 confirms that csvlog and jsonlog never read the buffer, and the default stderr setup pays one boolean test. Three fixable points are open: the check repeated at four sites (maintenance-fit-1), the unannounced '%l' change (standards-compat-1), and a benchmark limited to one 32MB statement (demand-1). The author has not answered message #3.

Not checked: The code was not read. The claim that all four consumers build the buffer first, and that the final pfree is guarded, rests on message #3. The syslog and eventlog paths were not run.

## Prior attempts

- [Skip vacuum log report code in lazy_scan_heap() if possible](https://www.postgresql.org/message-id/flat/CAJcOf-fH5FGsF3mpPL1m5yHLeR0HdXWQp9RA=LnHu0Qd=eq9DA@mail.gmail.com) (similar idea; Rejected, 2021-12-04). "I think this mostly reduces the coverage of the relevant code without any measurable speed gain." ([message](https://www.postgresql.org/message-id/20211203011344.lkxqyzhnm6jopubh%40alap3.anarazel.de)) Different now: This patch skips formatting on a path that runs for every logged message, and message #1 shows a measured gain for csvlog-only logging. The moved code stays covered because the default stderr setup still runs it.

## Objections a reviewer is likely to raise

### 1. Deferred build at four sites turns a structural invariant into a convention

A later patch adds a log destination, or a new branch in an existing one, that reads buf without repeating the flag check. With csvlog-only or jsonlog-only configurations, that branch formats uninitialized stack memory inside the error reporting path. Normal stderr setups and regression tests never hit it, so a user finds it as corrupt output or a crash.

Today initStringInfo(&buf) runs at the top of send_message_to_server_log(), so no destination can read an unbuilt buffer. The patch copies the check-and-build block to four sites, and nothing fails at compile time or in tests if a fifth site leaves it out (message #3). The author has not replied to this point. The defect would land in elog.c, the code that must work while other things fail.

**Question:** Can the build decision live in one place, such as an accessor that owns the flag or a predicate computed up front, so that a new consumer cannot skip it?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#3](https://www.postgresql.org/message-id/6aa3ff5e.749efa0d.1234b.4ae7%40mx.google.com); `maintenance-fit-1`</sub>

### 2. Log line number %l changes meaning without documentation

An administrator runs with log_destination = 'csvlog', then switches to 'stderr' with ALTER SYSTEM and pg_reload_conf(). After upgrade, '%l' in log_line_prefix restarts near 1 instead of continuing the per-session count. Log-correlation scripts that rely on that count see gaps or reuse they did not expect, and nothing in the release notes explains why.

Message #3 showed the effect: unpatched gives LINE=11, patched gives LINE=1. '%l' is documented as the "number of the log line for each session or process, starting at 1", and the patch makes it count only messages formatted for stderr, syslog or eventlog. The commit message claims no behaviour change, and the author has not answered the question in message #3. The new behaviour may be the better one, but a committer would likely want it chosen on purpose and documented, not left as a side effect.

**Question:** Is the new '%l' counting intended, and will the docs or the commit message say that it counts only lines rendered in the stderr format?

<sub>fixable, level 2, design; messages [#3](https://www.postgresql.org/message-id/6aa3ff5e.749efa0d.1234b.4ae7%40mx.google.com); `standards-compat-1`</sub>

### 3. Gain shown only for one extreme 32MB statement

A committer pushes the patch on the strength of an 11% gain measured with a 32MB statement. Typical csvlog-only servers log statements of a few KiB and may see no measurable change. Meanwhile, everyone who later edits send_message_to_server_log() must remember the deferred-build check at each consumer of buf.

The only evidence is one 60-second, one-client pgbench run of a 32MB statement with log_min_duration_statement = 0 (message #1). No user report, field case or typical-workload number shows that csvlog-only or jsonlog-only servers lose real time to this buffer. The new rule that each consumer must build buf first applies to everyone who edits elog.c (message #3). For that trade to be worth it, the gain should show up beyond the worst case.

**Question:** What is the measured gain with csvlog or jsonlog alone on a realistic logging-heavy workload, for example many clients running ordinary-sized statements with log_min_duration_statement = 0?

<sub>fixable, level 1b, demand; messages [#1](https://www.postgresql.org/message-id/tencent_A00A556AC153E46588F0680E34ED4C43EF09%40qq.com), [#3](https://www.postgresql.org/message-id/6aa3ff5e.749efa0d.1234b.4ae7%40mx.google.com); `demand-1`</sub>

## Questions for the author

1. Can the build decision live in one place, such as an accessor that owns the flag or a predicate computed up front, so a new consumer cannot skip it?
2. Is the new '%l' counting intended, and will the docs or the commit message say that it counts only lines rendered in the stderr format?
3. What is the measured gain with csvlog or jsonlog alone when many clients run ordinary-sized statements with log_min_duration_statement = 0?
4. Has anyone run the patched code with log_destination = 'syslog' and compared the output with master?

## Touch points

- **maintenance burden ("who maintains this in 10 years"):** Each of the four consumers of buf in send_message_to_server_log() must now run the deferred-build check first.
- **performance on the hot path:** The patch changes send_message_to_server_log(), which runs for every logged message.

## Who would commit it, and when

- Owner: silent. A committer who knows the error reporting code in elog.c and the syslogger paths would have to own it. None has spoken in the thread.
- Priority: fix design first; audience niche; review cost low. The patch touches only elog.c, and the shown gain is for csvlog-only logging of a 32MB statement. A reply to message #3 and a single-place build decision would make it a short review.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 1 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

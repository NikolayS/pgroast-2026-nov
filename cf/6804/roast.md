# Design pre-review: log_postmaster_stats

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6804](https://commitfest.postgresql.org/patch/6804/); [thread](https://www.postgresql.org/message-id/flat/CAKZiRmxeTNAe_VZ9m%3Drt%3D5155mvL%3DCFsQ4ENJWhuUdgMe%3D_Udg%40mail.gmail.com), 9 of 9 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The connection-rate threshold has support in the thread, so the goal is not in question. The CPU-based log_postmaster_overloads is: message #7 asked to drop it, and v3 and v4 keep it without answering the misfire concern (standards-compat-1), the missing test (maintenance-fit-1) or the overlap with per-process OS monitoring (already-exists-1). The list needs to settle whether that GUC stays before code review makes sense. The duplicate lines and the GUC name are smaller, fixable points.

Not checked: Without the code, this review could not check how the lateness rule is computed, where the counters sit in the connection accept path, or what the TAP test covers. Only 3 of 9 messages were shown in full.

## Prior attempts

None found before 2026-05-27.

## Objections a reviewer is likely to raise

### 1. CPU-based trigger kept against reviewer advice and cannot be tested

A later patch changes the postmaster wait loop or the timing of its wakeups. The log_postmaster_overloads path then stops firing, or fires on every disk stall, and no test fails because the CPU test is commented out. Users find out during an incident, when the expected message is missing or misleading.

Robert asked in message #7 to drop the CPU checks and keep only a connection-rate threshold, partly so the test could pass reliably. In message #8 the author kept both GUCs, citing the difficulty of choosing a rate. Message #6 says CI cannot overload the postmaster, and the commit message defines overload as "75% or getting late on CPU" without saying what "late" means. Robert has not replied since v3, so no committer has accepted owning an untested, scheduler-dependent trigger.

**Question:** Can log_postmaster_overloads be split into a separate later patch, after log_postmaster_excess_connections is committed with its reliable test?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#6](https://www.postgresql.org/message-id/CAKZiRmy%2Bqqox8XSk%2Bcd%2Bc7XmbGZHHpnAHOg3D-Ho-atuQRP4fg%40mail.gmail.com), [#7](https://www.postgresql.org/message-id/CA%2BTgmob7xpq3-ds536jo5vdenGJJ_rm5-okr7G-1F%3D5sU6HF9g%40mail.gmail.com), [#8](https://www.postgresql.org/message-id/CAKZiRmx996hSedQ9BW6v2qj092tUoZZiUHWHyD9YRyhxT8gvXg%40mail.gmail.com), [#9](https://www.postgresql.org/message-id/CAKZiRmzTaWmFZWKuP-qLN5QU1p3LzYFPDtJK8%2BeONfT%2BwO3e7g%40mail.gmail.com); `maintenance-fit-1`</sub>

### 2. Boolean overload GUC hides a fixed 75% CPU rule and an undefined lateness rule

A DBA enables log_postmaster_overloads and gets "potentially overloaded" lines while the postmaster waits on a slow disk or network write. On another machine a heavy storm uses only 60% of a fast core and nothing is logged. The DBA cannot tune either case because the trigger is hard-coded.

The commit message defines overload as "75% or getting late on CPU" but does not say how late counts as late. In message #7 Robert Haas predicted these false positives and false negatives and asked to drop the CPU checks. Message #8 keeps them as the "main feature" and gives no answer on misfires, so a committer would likely see the open point as unsettled. Once the GUC ships, its meaning becomes part of the contract, and changing the 75% rule or the lateness rule later changes behaviour for existing users.

**Question:** What exact condition triggers log_postmaster_overloads, and why should 75% and the lateness rule be fixed and not settable?

<sub>fixable, level 2, new GUC; messages [#7](https://www.postgresql.org/message-id/CA%2BTgmob7xpq3-ds536jo5vdenGJJ_rm5-okr7G-1F%3D5sU6HF9g%40mail.gmail.com), [#8](https://www.postgresql.org/message-id/CAKZiRmx996hSedQ9BW6v2qj092tUoZZiUHWHyD9YRyhxT8gvXg%40mail.gmail.com), [#9](https://www.postgresql.org/message-id/CAKZiRmzTaWmFZWKuP-qLN5QU1p3LzYFPDtJK8%2BeONfT%2BwO3e7g%40mail.gmail.com); `standards-compat-1`</sub>

### 3. Two GUCs log the same stats twice per second during a storm

A site sets both log_postmaster_overloads = on and log_postmaster_excess_connections = 200. During a long connection storm the postmaster writes two lines with the same numbers every second, so the log doubles at the moment it is already under load. Log-based alert rules that count lines double their counts.

Message #9 shows the duplicate output and leaves the fix open. Both settings guard one diagnostic, with two triggers and two names, where Postgres usually has one parameter per diagnostic. The proposal also has no limit on lines per storm beyond one per second, so a storm that lasts an hour writes thousands of lines.

**Question:** Why does one diagnostic need two independent GUCs, and what does the log contain when both trigger in the same second?

<sub>fixable, level 2, design; messages [#7](https://www.postgresql.org/message-id/CA%2BTgmob7xpq3-ds536jo5vdenGJJ_rm5-okr7G-1F%3D5sU6HF9g%40mail.gmail.com), [#9](https://www.postgresql.org/message-id/CAKZiRmzTaWmFZWKuP-qLN5QU1p3LzYFPDtJK8%2BeONfT%2BwO3e7g%40mail.gmail.com); `standards-compat-2`</sub>

### 4. The name log_postmaster_excess_connections does not say it is a rate

A DBA reads the name next to max_connections, takes it as a count of open sessions, and sets it to 500. The setting means 500 new connections per second, so a real storm of 300 per second is never logged.

In message #7 Robert Haas proposed log_excessive_connection_rate because that name tells users what the setting means. The v4 name drops "rate", and the integer has no unit, while nearby connection settings count sessions. Message #8 argues that a raw rate is hard to choose but does not explain the name.

**Question:** Why is a name without "rate" or a per-second unit better than the log_excessive_connection_rate spelling suggested in message #7?

<sub>fixable, level 2, new GUC; messages [#7](https://www.postgresql.org/message-id/CA%2BTgmob7xpq3-ds536jo5vdenGJJ_rm5-okr7G-1F%3D5sU6HF9g%40mail.gmail.com), [#8](https://www.postgresql.org/message-id/CAKZiRmx996hSedQ9BW6v2qj092tUoZZiUHWHyD9YRyhxT8gvXg%40mail.gmail.com); `standards-compat-3`</sub>

### 5. Postmaster CPU checks duplicate per-process OS monitoring

A DBA turns on log_postmaster_overloads and also runs a host agent that samples per-process CPU, or pidstat on the PID from postmaster.pid. Both report the same saturation, but the core check uses a fixed 75% threshold and a one-second window the DBA cannot tune. The server log then carries a second, less flexible copy of data the OS already records.

Per-process CPU history of the postmaster is available today from standard OS tools, sampled at any interval, with the PID taken from postmaster.pid. Message #5 says the real problem is that users miss one busy core among many busy backends, which makes the OS route less visible but still possible. Robert Haas (message #7) supported a connection-rate threshold and asked to drop the CPU checks; log_postmaster_excess_connections covers what core alone can see. The 75% CPU switch mostly repeats OS accounting inside the postmaster loop.

**Question:** Which postmaster overload cases does the CPU check catch that log_postmaster_excess_connections and per-process OS CPU sampling of the postmaster PID both miss?

<sub>fixable, level 1b, doesn't fit what Postgres is about (extension / external tool / already solved); messages [#5](https://www.postgresql.org/message-id/CAKZiRmxEz9zQKHdb3VNiZVh9PT2JDBo8sivBEY0giecSAaFxrA%40mail.gmail.com), [#7](https://www.postgresql.org/message-id/CA%2BTgmob7xpq3-ds536jo5vdenGJJ_rm5-okr7G-1F%3D5sU6HF9g%40mail.gmail.com), [#8](https://www.postgresql.org/message-id/CAKZiRmx996hSedQ9BW6v2qj092tUoZZiUHWHyD9YRyhxT8gvXg%40mail.gmail.com); `already-exists-1`</sub>

## Questions for the author

1. Which postmaster overload cases does the CPU check catch that log_postmaster_excess_connections and OS sampling of the postmaster PID both miss?
2. What exact delay counts as "getting late on CPU", and why are that rule and the 75% threshold fixed?
3. Can log_postmaster_overloads move to a separate later patch, so log_postmaster_excess_connections can be committed first with its reliable test?
4. What does the log contain when both GUCs trigger in the same second, and is there any limit during an hour-long storm?
5. Why is log_postmaster_excess_connections preferred over the log_excessive_connection_rate name suggested in message #7?

## Touch points

- **new GUC:** The patch adds log_postmaster_overloads and log_postmaster_excess_connections.
- **security / privileges:** Unauthenticated connection attempts can trigger the new LOG lines, at about one line per second per GUC.
- **maintenance burden ("who maintains this in 10 years"):** The CPU trigger depends on postmaster loop timing and its test is mostly commented out.
- **performance on the hot path:** The postmaster main loop gains counters and a once-per-second getrusage() check when a GUC is on.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** The 75% CPU check overlaps per-process CPU sampling that OS tools already provide.

## Who would commit it, and when

- Owner: qualified committer engaged. Robert Haas reviewed up to message #7 but has not answered v3 or v4. The owner needs to know the postmaster main loop and its wakeup timing.
- Priority: fix design first; audience a sizeable minority; review cost low. Sites with connection storms gain from the rate threshold, and the patch is about 24 KiB, mostly in postmaster.c. The open disagreement about the CPU trigger blocks progress more than patch size does.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 2 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

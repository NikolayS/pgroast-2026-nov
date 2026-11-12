# Design pre-review: enhancing pg_basebackup speeds up to ~23Gbps (small fixes + io_uring/Direct I/O)

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7142](https://commitfest.postgresql.org/patch/7142/); [thread](https://www.postgresql.org/message-id/flat/CAKZiRmwwW-hDc3B6ERJB%2BpaX7RNSBcQLheq1KdsTf42cGuRvuA%40mail.gmail.com), 15 of 15 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). No surviving objection questions the goal, and a committer is already reviewing the server-side buffer and fadvise patches. The surviving objections are fixable design points on the client side: the exported libpq function (replication-ondisk-1), the `blackhole` rename (standards-compat-1), and Direct I/O on by default with no stated fallback (replication-ondisk-3, standards-compat-3). demand-1 asks for an io_uring measurement without MPTCP. Confidence is medium because only 3 of 15 messages are visible.

Not checked: Without the code, the short-read and truncation handling in basebackup.c, the buffer lifetime of the new libpq call, and the Direct I/O alignment and error paths could not be checked. Only 3 of 15 messages were visible, so later answers may exist.

## Prior attempts

- [parallel backup](https://www.postgresql.org/message-id/flat/CADM=JehKgobEknb+_nab9179HzGj=9EiTzWMOd2mpqr_rifm0Q@mail.gmail.com) (similar idea; Returned with feedback, 2020-07-06). reason not recorded Different now: P17 sped up base backup with parallel workers and connections. This proposal keeps the single-threaded, single-connection design and cuts per-byte CPU and I/O cost instead.

## Objections a reviewer is likely to raise

### 1. New public libpq symbol for a single in-tree consumer

A driver author finds PQgetCopyDataInternalBuf() in libpq-fe.h, keeps the returned pointer across a later libpq call, and gets COPY data overwritten when libpq refills or moves its input buffer. The corruption is silent. Once exports.txt entry 212 ships, the project cannot remove or change the function without an ABI break.

Patch 0009 adds the function to exports.txt and libpq-fe.h, with no libpq.sgml hunk. The returned buffer is valid only until the next call, which ties callers to libpq's internal buffer handling. The commit message names pg_basebackup as the only consumer, and patches 0007-0011 have no review in the thread. A committer would likely ask for a frontend-private route, or for a documented API that other COPY consumers would use.

**Question:** Why must this be an exported libpq function instead of a frontend-private interface, and which buffer lifetime rules will the documentation promise?

<sub>fixable, level 2, ABI / extension API; messages [#1](https://www.postgresql.org/message-id/CAKZiRmwwW-hDc3B6ERJB%2BpaX7RNSBcQLheq1KdsTf42cGuRvuA%40mail.gmail.com); `replication-ondisk-1`</sub>

### 2. io_uring Direct I/O benefit shown only above single-flow TCP limits

A packager adds liburing as a client build dependency and users get the Direct I/O path, but on links under about 15 Gbps they see no speedup. In run i2, without MPTCP, throughput fell from 1870 to 1695 MB/s. Every build carries the dependency and the PG_BASEBACKUP_NODIO switch, while only users on very fast links gain.

Message #1 measures the 0009 gain only with MPTCP, which is not part of this entry and belongs in its own thread. Without MPTCP, run f already saturates the link at 1818 MB/s before 0009. The author's own table [3] puts single-flow TCP at 1-10 Gbps for most environments. That leaves a small group who can use the io_uring path, while the build dependency and the extra code path land on everyone.

**Question:** Which users today get more than about 15 Gbps on one TCP flow without MPTCP, and what does 0009 give them on that path alone?

<sub>fixable, level 1b, demand; messages [#1](https://www.postgresql.org/message-id/CAKZiRmwwW-hDc3B6ERJB%2BpaX7RNSBcQLheq1KdsTf42cGuRvuA%40mail.gmail.com); `demand-1`</sub>

### 3. Renaming the blackhole target breaks scripts and mixed-version backups

An operator's benchmark script runs `pg_basebackup --target=blackhole` and fails after upgrade. A new pg_basebackup running `-t server-blackhole` against a Postgres 15-19 server sends TARGET 'server-blackhole', which the older server rejects. An older client sending 'blackhole' to a new server fails the same way.

Patch 0001 changes the TARGET value in protocol.sgml and in the pg_basebackup docs, and its commit message mentions no alias. pg_basebackup is expected to work against older servers, and BASE_BACKUP TARGET values are part of the replication protocol. The rename is cosmetic, because adding `client-blackhole` does not require removing `blackhole`.

**Question:** Will the server keep accepting `blackhole` as an alias, and how does a new pg_basebackup choose the spelling when connected to a pre-20 server?

<sub>fixable, level 2, backpatchability; messages [#1](https://www.postgresql.org/message-id/CAKZiRmwwW-hDc3B6ERJB%2BpaX7RNSBcQLheq1KdsTf42cGuRvuA%40mail.gmail.com), [#15](https://www.postgresql.org/message-id/CAKZiRmyTs0ciaPztkF_VRUp0NXhRLnd%2BAAuJBtW6102QO-rezQ%40mail.gmail.com); `standards-compat-1`</sub>

### 4. Direct I/O backup writes lack a defined fallback

An operator upgrades the client and points pg_basebackup -D at a filesystem that rejects O_DIRECT or needs a different alignment, such as some NFS or tmpfs setups. The backup fails partway, or writes take an untested path. The operator finds out during a scheduled backup run.

Patch 0010 makes io_uring Direct I/O the default for larger files. The only control is the PG_BASEBACKUP_NODIO environment variable, which is undocumented. The material gives a buffered fallback only for the tar path when posix_fallocate fails (message #1). It does not say what happens when open(O_DIRECT) or an aligned write fails, or how non-Linux builds without liburing behave. These are the writes that produce the user's backup.

**Question:** When O_DIRECT or io_uring is unavailable or rejected at run time, does pg_basebackup fall back to buffered writes on every platform, and how is that tested?

<sub>fixable, level 2, on-disk format / pg_upgrade; messages [#1](https://www.postgresql.org/message-id/CAKZiRmwwW-hDc3B6ERJB%2BpaX7RNSBcQLheq1KdsTf42cGuRvuA%40mail.gmail.com); `replication-ondisk-3`</sub>

### 5. Direct I/O control is an environment variable, not an option

After upgrade, an operator writes backups to a filesystem that rejects O_DIRECT or needs different alignment. The only documented way to turn off Direct I/O is PG_BASEBACKUP_NODIO, which is not visible in `--help` or the existing scripts. The operator finds this only when the backup fails or runs slowly.

Patch 0010 turns on io_uring Direct I/O for large files by default and adds PG_BASEBACKUP_NODIO, described in message #1 as "for experimenting". pg_basebackup controls client write behaviour with options such as --no-sync, so an environment variable departs from that practice. The proposal does not say how a build without liburing, or a non-Linux build, behaves.

**Question:** What happens on filesystems that reject O_DIRECT and on builds without liburing, and why is Direct I/O not controlled by a pg_basebackup option?

<sub>fixable, level 2, design; messages [#1](https://www.postgresql.org/message-id/CAKZiRmwwW-hDc3B6ERJB%2BpaX7RNSBcQLheq1KdsTf42cGuRvuA%40mail.gmail.com); `standards-compat-3`</sub>

## Questions for the author

1. Why must PQgetCopyDataInternalBuf() be an exported libpq function instead of a frontend-private interface, and which buffer lifetime rules will libpq.sgml promise?
2. Will the server keep accepting `blackhole` as an alias, and what does a new pg_basebackup send to a pre-20 server for `-t server-blackhole`?
3. When the target filesystem rejects O_DIRECT, or the build has no liburing, does pg_basebackup fall back to buffered writes on every platform?
4. What does the io_uring Direct I/O patch gain on a single TCP flow without MPTCP, given that run i2 was slower than run h3?
5. Why is Direct I/O on by default and controlled by the PG_BASEBACKUP_NODIO environment variable instead of a documented pg_basebackup option?

## Touch points

- **on-disk format / pg_upgrade:** see objection replication-ondisk-3: Direct I/O backup writes lack a defined fallback
- **WAL / replication / standby:** The series rewrites the server-side base backup read path in basebackup.c, including short-read and truncation handling.
- **wire protocol:** Patch 0001 renames the BASE_BACKUP TARGET value `blackhole` to `server-blackhole` in protocol.sgml.
- **ABI / extension API:** The libpq patch adds PQgetCopyDataInternalBuf() as exports.txt entry 212.
- **backpatchability:** The target rename makes a new pg_basebackup and an older server disagree on the name, in both directions.
- **maintenance burden ("who maintains this in 10 years"):** The series adds a Linux-only io_uring Direct I/O write path, a liburing client dependency and a permanent libpq export.
- **performance on the hot path:** The 256 kB sink buffer and the fadvise calls change the read path of every base backup.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** see objection demand-1: io_uring Direct I/O benefit shown only above single-flow TCP limits

## Who would commit it, and when

- Owner: qualified committer engaged. Heikki Linnakangas reviewed the server-side buffer and short-read patches. The client-side patches need someone who knows the libpq API and ABI rules, and someone who knows the pg_basebackup astreamer code.
- Priority: proceed; audience a sizeable minority; review cost high. The server-side buffer and fadvise patches have measured gains, an independent report of slow incremental backups in message #2, and active committer review, so they can go ahead on their own. The libpq export and the io_uring patches span libpq, fe_utils and pg_basebackup and need their own design answers first.

---

[roast-full.md](roast-full.md) has 1 more objection(s) that held, 7 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 15.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

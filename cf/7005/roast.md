# Design pre-review: Add configure-time choice of either port/atomics.h or <stdatomic.h> for atomic operations

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7005](https://commitfest.postgresql.org/patch/7005/); [thread](https://www.postgresql.org/message-id/flat/CA%2BhUKGKFvu3zyvv3aaj5hHs9VtWcjFAmisOwOc7aOZNc5AF3NA%40mail.gmail.com), 35 of 35 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The goal has support in the thread, but the design is not settled. The proposal offers v11 and v12 together and does not say which one is for commit. Each has an open design point: v11 has no tester or exit rule and changes the slock_t layout by build option (maintenance-fit-1, replication-ondisk-1); v12 removes the old code on untested platforms (maintenance-fit-3, standards-compat-2). Both rely on a seq_cst read that hides a tuple loss of unknown cause (maintenance-fit-2).

Not checked: Only the first 3 of 35 messages and the v1 and v2 patch prose were read; no code was read. Layout sizes under both settings, the barrier implementation and the generated code per architecture could not be checked.

## Prior attempts

None found before 2025-11-10.

## Objections a reviewer is likely to raise

### 1. Seq_cst read is a workaround for a bug whose cause is unknown

Under v12, a POWER or rv64 build pays an hwsync or two fences on each of about 200 pg_atomic_read_u32/u64 call sites, including LWLock and buffer paths. Nobody can safely relax this later, because no one knows which caller needs the ordering. A new compiler then exposes the hidden bug in some other form.

Message #35 says relaxed reads lose a join_hash tuple on rv64 with gcc 13.3 only, and the cause is not found. Message #24 also reports that a pg_read_barrier in the consumer did not help. So the stronger ordering hides a missing ordering somewhere, and the project would keep it forever against the documented "no barrier semantics" contract. The read cost is measured on aarch64 only; message #35 says it is not benchmarked on rv64 or POWER.

**Question:** Which caller relies on ordering that pg_atomic_read_u32 does not promise, and what does seq_cst cost on POWER and rv64 under a contended pgbench run?

<sub>fixable, level 2, design; messages [#24](https://www.postgresql.org/message-id/34101612-a419-4811-a672-f9e55b4fbec2%40app.fastmail.com), [#35](https://www.postgresql.org/message-id/BGXViuPJAXY5dAefuF31MhArLzQjKYoS60rlNordOBLSX7_pATL66i-2-nYMIHeyoHSGLDQ3plkUjSpW-1SPAE4y0MBdWo8JVFbUj4UKKO4%3D%40burd.me); `maintenance-fit-2`</sub>

### 2. Default-off second atomics implementation has no tester and no exit plan

A later patch adds a new pg_atomic_* operation or changes barrier use and updates only the traditional headers, which every default build uses. The stdatomic path then stops building or behaves differently. Nobody finds out until someone tries -Duse_stdatomic=yes months later.

v11 keeps arch-*.h, generic*.h and s_lock.h and adds stdatomic_impl.h beside them, so each atomics change must be made twice. The proposal names no CI task or buildfarm animal that builds the opt-in path. In message #25 Peter asks what testing the option is for and what happens when it is complete. Message #26 answers only with "cut over mid-cycle if all looks good", which gives no criteria and no date. No committer has replied since v11 was posted, and co-author Thomas Munro has not posted since message #15.

**Question:** Which CI tasks or buildfarm animals will build the opt-in path by default, and what results would trigger removing one implementation?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#15](https://www.postgresql.org/message-id/CA%2BhUKGJSx124Y1gFJi2y_eRFpO%3DUbcdi71-iEn6nmBQjbZFZ%2Bg%40mail.gmail.com), [#25](https://www.postgresql.org/message-id/649b0015-f345-42fe-b1ec-a23beadeec6e%40eisentraut.org), [#26](https://www.postgresql.org/message-id/cae320c9-6882-47c0-8d06-8ff8ab48aafb%40app.fastmail.com), [#35](https://www.postgresql.org/message-id/BGXViuPJAXY5dAefuF31MhArLzQjKYoS60rlNordOBLSX7_pATL66i-2-nYMIHeyoHSGLDQ3plkUjSpW-1SPAE4y0MBdWo8JVFbUj4UKKO4%3D%40burd.me); `maintenance-fit-1`</sub>

### 3. V12 hands atomics to compilers on untested platforms, with no fallback

After v12 is committed, a POWER, s390x or illumos buildfarm animal shows rare corruption or a hang, or an MSVC update changes /experimental:c11atomics. The project has deleted the hand-written code it would compare against. Back branches keep the old code, so fixes no longer backpatch cleanly.

In message #20 Tom asked for an alternative and not a replacement, because stdatomic codegen is not under project control. v12 removes the fallback anyway, and message #35 says the buildfarm "would be the first real test" on POWER, s390x and Solaris/illumos. MSVC still calls the needed flag experimental. A committer would likely ask for v11-style evidence on those platforms before deleting anything.

**Question:** What codegen comparison or test results on POWER, s390x and illumos show the stdatomic path matches the existing code before the old headers are removed?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#8](https://www.postgresql.org/message-id/CA%2BhUKGKfNuXYVKT7WPpKTNYTgPduzu0%3DG5yFEMju_4kbW0ybOQ%40mail.gmail.com), [#20](https://www.postgresql.org/message-id/3532551.1774275186%40sss.pgh.pa.us), [#35](https://www.postgresql.org/message-id/BGXViuPJAXY5dAefuF31MhArLzQjKYoS60rlNordOBLSX7_pATL66i-2-nYMIHeyoHSGLDQ3plkUjSpW-1SPAE4y0MBdWo8JVFbUj4UKKO4%3D%40burd.me); `maintenance-fit-3`</sub>

### 4. Build option changes slock_t layout without an extension ABI check

A user runs a server built with -Duse_stdatomic=yes or auto and loads an extension binary built against default headers. A shared-memory struct holding slock_t or pg_atomic_flag gets different offsets on each side. The extension then reads and writes the wrong bytes under load, and nothing reports a mismatch at load time.

Message #34 says slock_t becomes a pg_atomic_uint32, and the brief says pg_atomic_flag changes representation. On x86 the traditional slock_t is one byte. With v11 the layout of extension-visible shared-memory structs therefore depends on a configure choice, and with "auto" on the build host's probe result. The proposal does not say whether the option goes into the PG_MODULE_MAGIC ABI fields or whether both paths keep identical sizes. v12 avoids this because it has only one layout per major version.

**Question:** Do slock_t and pg_atomic_flag keep the same size and alignment under both settings, and if not, how does loading a mismatched extension fail?

<sub>fixable, level 2, ABI / extension API; messages [#34](https://www.postgresql.org/message-id/7gJmxtbunjsr2_04YEK5pN3XH0ZqYY8E2HQxFZFEsCUxScihXgxQyL5KNQ6nCO_X1bOLgsiCcPX1iy7sB9BSQH8MoqdB4-wgRTYAYXo0UR4%3D%40burd.me), [#35](https://www.postgresql.org/message-id/BGXViuPJAXY5dAefuF31MhArLzQjKYoS60rlNordOBLSX7_pATL66i-2-nYMIHeyoHSGLDQ3plkUjSpW-1SPAE4y0MBdWo8JVFbUj4UKKO4%3D%40burd.me); `replication-ondisk-1`</sub>

### 5. v12 turns build success on some platforms into a configure failure

A packager for a 32-bit target whose compiler emulates 64-bit atomics with locks upgrades to the release with v12. configure now stops with an error, where master builds through the existing fallback code. The packager finds out at build time, with no supported way to continue.

Message #35 says configure and meson now fail unless 32- and 64-bit atomics are always lock-free. The v1 commit message planned to let the compiler emulate 64-bit atomics with locks on armv7, so the requirement is narrower than the earlier design. Message #35 also says POWER, s390x, Solaris and illumos are untested and that "the buildfarm would be the first real test". The proposal does not list which supported platforms and compilers meet the new requirement and which lose support.

**Question:** Which platforms that master builds on today would fail the new lock-free check, and is dropping them an explicit decision?

<sub>fixable, level 2, design; messages [#1](https://www.postgresql.org/message-id/CA%2BhUKGKFvu3zyvv3aaj5hHs9VtWcjFAmisOwOc7aOZNc5AF3NA%40mail.gmail.com), [#12](https://www.postgresql.org/message-id/543920.1763917337%40sss.pgh.pa.us), [#35](https://www.postgresql.org/message-id/BGXViuPJAXY5dAefuF31MhArLzQjKYoS60rlNordOBLSX7_pATL66i-2-nYMIHeyoHSGLDQ3plkUjSpW-1SPAE4y0MBdWo8JVFbUj4UKKO4%3D%40burd.me); `standards-compat-2`</sub>

## Questions for the author

1. Which version is proposed for commit, v11 or v12, and if v11, what result and what release would trigger the move to v12?
2. Which caller relies on ordering that pg_atomic_read_u32/u64 does not promise, causing the join_hash tuple loss on rv64 with gcc 13.3?
3. What does the seq_cst read cost on POWER and rv64 under a contended pgbench run?
4. Which CI tasks or buildfarm animals will build the opt-in stdatomic path, and how does a server reject an extension built with the other slock_t layout?
5. Which platforms that master builds today would fail v12's lock-free check, and what test results exist for POWER, s390x and illumos?

## Touch points

- **ABI / extension API:** slock_t and pg_atomic_flag change representation, and pg_atomic_read_u32/u64, which extensions call, get seq_cst ordering.
- **backpatchability:** v12 removes arch-*.h, generic*.h, fallback.h and s_lock.h, which back branches keep.
- **maintenance burden ("who maintains this in 10 years"):** v11 keeps two atomics implementations in the tree; v12 depends on compiler codegen and on MSVC's /experimental:c11atomics.
- **performance on the hot path:** seq_cst reads apply to about 200 pg_atomic_read call sites and are not benchmarked on rv64 or POWER.

## Who would commit it, and when

- Owner: qualified committer engaged. Thomas Munro co-wrote the patch, and Tom Lane, Heikki Linnakangas and Peter Eisentraut commented, but none has replied since v11 and v12 were posted. The owner must know atomics, spinlocks and weak-memory architectures.
- Priority: fix design first; audience niche; review cost high. The gain is less in-tree code for maintainers; users get performance parity. The patch touches atomics, spinlocks and the build system on every architecture, and the choice between v11 and v12 and the rv64 tuple loss are open.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 5 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

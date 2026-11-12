# Design pre-review: Allow bare library names for non-superuser LOAD

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7254](https://commitfest.postgresql.org/patch/7254/); [thread](https://www.postgresql.org/message-id/flat/tencent_E6968C151EDE818A1D4CA85FA1772D668A06%40qq.com), 8 of 8 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The goal is modest and no objection is fatal. The open point is standards-compat-1: after the patch, `LOAD 'demo'` loads `$libdir/demo` for a superuser and `$libdir/plugins/demo` for other roles, with no message. Message #6 shows the case, and the answer in message #7 was not accepted in message #8. demand-1 is a secondary point: convenience is the only stated motive, and the explicit prefix works today.

Not checked: Without the code, I could not check how the patch detects a bare name, including Windows path separators, or whether dynamic_library_path is ever consulted. Only 3 of 8 messages were read directly.

## Prior attempts

None found before 2026-09-03.

## Objections a reviewer is likely to raise

### 1. Same LOAD string resolves to different files depending on role

An admin tests a setup script as superuser, where `LOAD 'demo'` loads `$libdir/demo`. The same script run by an application role, or after `SET ROLE`, loads `$libdir/plugins/demo` without any message. Before the patch that run failed with an error, as message #6 shows with auto_explain.

local_preload_libraries prepends `$libdir/plugins` for every role, so it does not depend on the role. The patch makes one LOAD string mean two paths depending on `superuser()`, so the consistency claim holds only for the restricted branch. Message #7 answers that admins should not install same-named libraries, and message #8 calls this trading a server guarantee for admin discipline. The thread did not settle the point, and nothing in the proposal detects or reports the ambiguity.

**Question:** Why not resolve bare names the same way for all roles, or raise an error when the name exists in both `$libdir` and `$libdir/plugins`?

<sub>fixable, level 2, design; messages [#6](https://www.postgresql.org/message-id/b08c5b95-f36d-412b-a936-7e710df6d24e%40uni-muenster.de), [#7](https://www.postgresql.org/message-id/tencent_170EF0BA0E2EEC98439B0B536AA352006409%40qq.com), [#8](https://www.postgresql.org/message-id/fbf5df8b-2c09-4153-bcad-c7682686d632%40uni-muenster.de); `standards-compat-1`</sub>

### 2. No user demand shown for dropping the $libdir/plugins/ prefix

A deployment where the same library name exists in $libdir and $libdir/plugins sees LOAD 'name' pick a different file per role, as message #6 shows. Admins learn this when a script that worked as superuser loads another module under a normal role. The users who gain only skip typing a fixed prefix.

The only stated need is the author's view that the prefix is inconvenient (message #3), and Japin Li called it "a bit inconvenient" (message #4). No field report or -general request is cited, and no non-superuser workflow is shown that the prefix blocks. The workaround is to write '$libdir/plugins/name', which works today and is documented. The cost of role-dependent resolution and a changed error falls on admins and scripts that never asked for the shortcut.

**Question:** Which users or tools run non-superuser LOAD of plugins today, and where has the explicit prefix caused them real trouble?

<sub>fixable, level 1b, demand; messages [#3](https://www.postgresql.org/message-id/tencent_A25AB53229C6323A784407EA76DAA1AB8508%40qq.com), [#4](https://www.postgresql.org/message-id/SY7PR01MB10921D6FCB0C8E7BBBF111DD8B6B62%40SY7PR01MB10921.ausprd01.prod.outlook.com), [#6](https://www.postgresql.org/message-id/b08c5b95-f36d-412b-a936-7e710df6d24e%40uni-muenster.de), [#8](https://www.postgresql.org/message-id/fbf5df8b-2c09-4153-bcad-c7682686d632%40uni-muenster.de); `demand-1`</sub>

## Questions for the author

1. Why not resolve bare names the same way for all roles, or raise an error when the name exists in both `$libdir` and `$libdir/plugins`?
2. Which users or tools run non-superuser LOAD of plugins today, and where has the explicit prefix caused them real trouble?
3. Does the new "could not access file" error for a missing plugin carry a different SQLSTATE than the old "access to library ... is not allowed" error?
4. Does moving the expansion into load_file() change superuser LOAD or session_preload_libraries in any way?

## Touch points

- **security / privileges:** The patch changes the path check in load_file() that limits non-superuser LOAD to `$libdir/plugins`.
- **maintenance burden ("who maintains this in 10 years"):** Role-dependent resolution of a bare LOAD name becomes documented behavior that later changes must keep.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** `LOAD '$libdir/plugins/name'` already gives non-superusers the same result today.

## Who would commit it, and when

- Owner: silent. Someone who knows the dynamic loader code in dfmgr.c and the preload library settings would have to own it. No committer is identified in the visible thread.
- Priority: fix design first; audience niche; review cost low. The patch is 3.8 KB across load.sgml, dfmgr.c and miscinit.c, and it helps only sites where non-superusers LOAD plugins. The role-dependent resolution from message #6 needs an agreed answer before code review is useful.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 3 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

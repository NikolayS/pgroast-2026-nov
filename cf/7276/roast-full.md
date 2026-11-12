# Design pre-review: Prevent object capture in CREATE/ALTER EXTENSION scripts

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7276](https://commitfest.postgresql.org/patch/7276/) (Needs review); authors: Jan Nidzwetzki (jnidzwetzki)
- Thread: [Prevent object capture in CREATE/ALTER EXTENSION scripts](https://www.postgresql.org/message-id/flat/28f66580-5d6b-4997-95f6-7d9e7917c585%40planetscale.com), started 2026-09-07
- Messages read: 7 of 7
- Patch prose read: v2-0001-Propagate-extension-script-state-to-parallel-work.patch, v2-0002-Prefer-trusted-candidates-when-resolving-names-in.patch, 0001-Propagate-extension-script-state-to-parallel-workers.patch, 0002-Prefer-trusted-candidates-when-resolving-names-in-ex.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

The problem is stated and demand is shown, so levels 1a and 1b pass. Four fixable level 2 objections survived: security-1 (an ordinary role can add its own objects to a required extension and have them trusted), security-2 (a lookup path that skips the check fails silently), standards-compat-1 and standards-compat-2. None requires a different proposal, but the trust rule sits in core name resolution and needs a committer who knows that code.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The cost of the trust check and the pg_depend lookup, the completeness of the filtered lookup paths, and the plancache handling could not be judged without the code. The answer on ALTER EXTENSION ... ADD may also depend on details of the membership test.

## The proposal, restated

**Problem.** A CREATE EXTENSION or ALTER EXTENSION ... UPDATE script runs as the invoking role or as the bootstrap superuser, so an object another role planted in the extension's schema can capture a reference and run that role's code with elevated rights. Pinned search_path and @extschema@ qualification do not help, because the plant sits in the script's own schema. Hosted platforms that run extension scripts as superuser for users are the main case.

**Proposed design.** While creating_extension is set, relation, type, function and operator lookups skip untrusted objects, for qualified and search_path names. An object is trusted if it is in pg_catalog, owned by a superuser or by the role running the script, or belongs to the extension or one it requires. Cached plans record whether they were analyzed in a script, and 0001 passes the script state to parallel workers.

**User-visible surface.**
- Extension scripts that reference an object owned by another ordinary role, even schema-qualified, now fail as if the object did not exist.
- New error detail saying a candidate was ignored, and a hint naming the ignored candidate when argument types do not match.
- Overload resolution inside scripts can pick a different function or operator than before.

**What the author claims.**
- Closes object capture through planted overloads, operators, domains and tables during script execution.
- Ordinary parsing outside extension scripts is unchanged.
- The trust rule accepts built-ins, the extension's own objects and required extensions' objects, so normal scripts keep working.
- References to objects owned by other ordinary roles should be rare.
- Complements the opt-in "extensions with an owned schema" proposal, which only helps new extensions.

**Thread state.** Two reviewers reproduced the bug and report that v1 fixes it without finding problems; no committer has replied. The author posted v2 to also check schema-qualified relation and type names, and the CommitFest entry is in Needs review.

## 2. Prior attempts

### Fix search_path for maintenance commands (related)

- **Link:** [https://commitfest.postgresql.org/patch/4367/](https://commitfest.postgresql.org/patch/4367/)
- **Thread:** [https://www.postgresql.org/message-id/flat/e44327179e5c9015c8dda67351c04da552066017.camel@j-davis.com](https://www.postgresql.org/message-id/flat/e44327179e5c9015c8dda67351c04da552066017.camel@j-davis.com)
- **Date:** decided 2024-01-18; started 2023-05-26
- **Outcome:** Withdrawn (cf)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (AI summary of the linked thread; check it there):** Jeff Davis withdrew the patch after a reviewer reported that it no longer applied to HEAD, saying it was not yet clear this was the right change. He had already noted that it forces search_path in maintenance commands but not in other contexts, and that his broader proposals to fix search_path for functions got little agreement. Earlier concerns about breaking functions that depend on search_path were raised in messages not shown. Nathan Bossart supported committing it.
- **Reason (quote):** "I am withdrawing this patch from the CF until it's more clear that this is what we really want to do." ([message](https://www.postgresql.org/message-id/b7244fd967fbec96ead8d4546e1d1e93e5eb82b7.camel%40j-davis.com))
- **What is different this time:** P3 forced search_path to "pg_catalog, pg_temp" during maintenance commands. This proposal keeps search_path and filters candidates by owner during extension scripts, so it also blocks plants in the script's own schema.


## 3. Objections a reviewer is likely to raise

### 1. Membership in a required extension can be granted by an ordinary role

fixable · level 2, rubric item 7 · security / privileges · `security-1`

An ordinary role installs a trusted extension that another extension requires, so it owns that extension. It creates f(text) in the shared schema and runs alter extension ... add function f(text). A superuser or platform later installs the dependent extension, and its script resolves f('abc') to the planted function because it is a member of a required extension.

The trust rule in message #1 accepts any member of a required extension, "regardless of who owns them". alter extension add needs only ownership of the extension and of the object. A non-superuser who installs a trusted extension owns it, so membership is a property an attacker can set. The hosted-platform case the proposal targets is where unprivileged users install trusted extensions on their own.

Thread: [#1 (Jan Nidzwetzki, 2026-09-07)](https://www.postgresql.org/message-id/28f66580-5d6b-4997-95f6-7d9e7917c585%40planetscale.com)

**Question:** How does the trust rule stop an ordinary role that owns a required extension from adding its own objects to it and having them trusted?

Counter-check: held. A non-superuser who installs a trusted extension owns it, and alter extension ... add needs only ownership of the extension and of the object. The rule in message #1 trusts members of required extensions regardless of owner, and the thread does not address this route.

### 2. Each lookup path must opt in, and a missed path fails silently

fixable · level 2, rubric item 9 · security / privileges, maintenance burden ("who maintains this in 10 years") · `security-2`

A later patch adds a new lookup route, or changes an existing one, without calling the trust check. Scripts on that route go back to resolving planted objects. No test fails, and the gap shows up as a security report against a guarantee the project now documents.

The protection depends on every relevant lookup site calling the check. v1 already missed the qualified branches of RangeVarGetRelidExtended() and LookupTypeNameExtended() (message #7). Message #1 leaves operator classes, collations, text search objects, casts and conversions unfiltered and says a route may have been missed. Once the guarantee is committed, the project has to keep it complete in all of these places.

Thread: [#1 (Jan Nidzwetzki, 2026-09-07)](https://www.postgresql.org/message-id/28f66580-5d6b-4997-95f6-7d9e7917c585%40planetscale.com), [#7 (Jan Nidzwetzki, 2026-09-22)](https://www.postgresql.org/message-id/3f6db46e-3175-40de-9001-ccde3a8b05f2%40planetscale.com)

**Question:** What makes a new or changed name-lookup path fail loudly, in tests or by construction, when it skips the trust check during extension scripts?

Counter-check: held. Message #7 shows v1 missed the qualified relation and type branches after two reviews found no problem. Message #1 lists lookup paths that remain unfiltered and says a route may have been missed, and nothing shown makes a skipped check fail loudly.

### 3. Existing scripts can stop working on update, and there is no escape hatch

fixable · level 2, rubric item 6 · design, backpatchability, maintenance burden ("who maintains this in 10 years") · `standards-compat-1`

An extension script references an object owned by an ordinary role, for example a function of a superuser=false extension that is optional or only a transitive dependency. After the server upgrade, ALTER EXTENSION ... UPDATE fails with "does not exist". On a hosted platform the user cannot change object ownership or edit the script, so the update stays blocked.

Message #1 names this behaviour change and calls such references "rare", but gives no survey of contrib or third-party scripts. The proposal has no control-file option or other way to mark an object as trusted. The only fixes are to reassign ownership to a superuser or to add a requires entry, and neither is open to the user who hits the error. The commit message carries Reported-by, so a committer would also likely ask whether this is meant for back branches.

Thread: [#1 (Jan Nidzwetzki, 2026-09-07)](https://www.postgresql.org/message-id/28f66580-5d6b-4997-95f6-7d9e7917c585%40planetscale.com)

**Question:** Which existing extension scripts reference objects of ordinary roles outside their requires list, and how does a user unblock an update when one fails?

Counter-check: held. Message #1 confirms that a schema-qualified reference to another ordinary role's object now fails, and offers only "should be rare" as evidence. The proposal names no way for a user to unblock a failed update, and the brief lists the affected extensions as unknown.

### 4. The requires list becomes a name-visibility rule

fixable · level 2, rubric item 12 · design, maintenance burden ("who maintains this in 10 years") · `standards-compat-2`

Extension A requires B, and B requires C, where C is a superuser=false extension owned by an ordinary role. A's script calls a C function by its schema-qualified name. With this patch the call fails, although A installs only when C is present. The extension author finds out from a user's failed CREATE EXTENSION.

Today requires controls install order and @extschema:name@ substitution. Under 0002 it also decides which objects a script can see, and only for direct entries, as message #1 says. Extension authors would have to list transitive and conditional dependencies to keep scripts working. No documentation hunk is shown, so the new meaning of requires does not yet appear in the extension-authoring docs.

Thread: [#1 (Jan Nidzwetzki, 2026-09-07)](https://www.postgresql.org/message-id/28f66580-5d6b-4997-95f6-7d9e7917c585%40planetscale.com)

**Question:** Why is trust limited to direct requires entries, and where would the new meaning of requires be documented for extension authors?

Counter-check: held. Message #1 says objects of an extension outside the direct requires list are untrusted, so the A, B, C scenario follows from the stated rule. The thread gives no reason for stopping at direct entries.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | **yes** | The trust rule reads pg_depend, and 0002 touches namespace.c and pg_operator.c; no catalog change is described. |
| ABI / extension API | **yes** | 0002 changes namespace.h, parse_type.h, plancache.h and extension.h, which extensions include. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | **yes** | CachedPlanSource records whether a plan was analyzed inside an extension script, which affects plan reuse. |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | **yes** | The patch adds an ownership-based trust rule to stop privilege escalation through planted objects in extension scripts. |
| backpatchability | **yes** | The commit message carries Reported-by for a security issue, but the change can break existing extension scripts. |
| maintenance burden ("who maintains this in 10 years") | **yes** | Each relation, type, function and operator lookup path and each cache must apply the trust check from now on. |
| performance on the hot path | **yes** | The check sits in name lookup and plancache code; the cost outside extension scripts is not measured. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. How does the trust rule stop an ordinary role that owns a required extension from adding its own objects with ALTER EXTENSION ... ADD and having them trusted?
2. What makes a new or changed name-lookup path fail loudly, in tests or by construction, when it skips the trust check during an extension script?
3. Which contrib or third-party extension scripts reference objects owned by ordinary roles outside their requires list, and how does a user unblock a failed update?
4. Why does trust stop at direct requires entries, and where would the docs tell extension authors about this new meaning of requires?
5. Is this meant for back branches or only for master, given that it can make existing update scripts fail?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows namespace.c, parser name resolution and plancache, and who is willing to sign for a security guarantee about extension scripts.
- **Liability:** A documented guarantee that extension scripts never resolve to untrusted objects, kept complete across every current and future lookup path and cache, with no end date. Any missed path becomes a security report.
- **Author follow-up:** The author posted v2 in message #7 to cover schema-qualified relation and type names, which shows follow-up during review.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs champion
- **Audience:** a sizeable minority
- **Maintenance burden:** high
- **Review cost:** high
- **Why:** Hosted platforms that run extension scripts as superuser benefit, and two reviewers reproduced the capture. The 83.5 KB patch changes namespace.c, the parser and plancache, and no committer has replied in the visible thread.

## Appendix

### Refuted in the counter-check

- `maintenance-fit-1` Every current and future lookup path must remember the trust check. Refuted (not specific): Duplicate of security-2, which states the same scenario and the same question to the author. The remark that no committer has replied belongs in committability and does not support the verdict.
- `maintenance-fit-2` Script-only resolution mode leaves the wider capture problem open. Refuted (no failure scenario): The scenario depends on a future design that nobody has proposed, and message #1 states the script-only scope and its reason. The existing defenses 7eeb1d9861b and b9b21acc766 are also script-only, and P3 was withdrawn on its author's own doubt, so it does not support this.
- `maintenance-fit-3` Existing extension updates can start failing, with no backpatch path. Refuted (not specific): Duplicate of standards-compat-1, which covers the broken update, the missing survey, the missing escape hatch and the back-branch question. Supported branches staying exposed is the current state, so the proposal does not cause it.

### Review angles

- prior-attempts: 0 objection(s). No precedent records a reason that applies here. P3, P9 and P11 deal with search_path escalation in maintenance commands: P9 was reverted for timing in the release cycle, P11 records no reason, and P3 was withdrawn without a firm objection. P2 and P7 are different ideas. Message #1 cites the owned-schema proposal and the sandboxing thread but reports no rejection of this approach.
- standards-compat: 2 objection(s). The SQL surface is unchanged: no new syntax, keywords, GUCs or catalog objects. pg_dump and pg_upgrade are not affected by the design as described.
- replication-ondisk: 0 objection(s). The proposal changes name lookup only while an extension script runs; it adds no on-disk format, WAL record, catalog column or wire protocol message. Plan cache and parallel worker state stay in memory, so I found no persistent or cross-node state to object to; any ABI or backpatch effect of the header changes cannot be judged without the code.
- security: 2 objection(s). Two objections raised. The error hint naming an ignored candidate tells the script runner only about an object the planting role already knows, so it does not leak information.
- maintenance-fit: 3 objection(s). Hot-path cost for non-users looks small from the prose, since the checks apply only while creating_extension is set. The size of the plancache flag cost and the pg_depend lookups could not be checked without code or numbers.
- already-exists: 0 objection(s). No existing feature covers this. Revoking CREATE on the target schema fails when the planting role is the non-superuser installing a trusted extension. Schema-qualifying names and adding casts depends on third-party script authors, and the owned-schema proposal is uncommitted and opt-in. No extension or hook can filter core name lookup.
- demand: 0 objection(s). Demand is real for a security fix: a named external reporter, a hazard the docs already warn about, and two reviewers who reproduced the capture. Cost falls only on extension scripts, and hosted platforms that run scripts as superuser are a large group.

### Precedents retrieved

- P1: [Prevent extension creation in temporary schemas](https://commitfest.postgresql.org/patch/1969/) (cf, Returned with feedback, 2019-03-07) - judged not relevant; reason summarised from its thread
- P2: [Add checks on search_path for IMMUTABLE and SECURITY DEFINER functions](https://commitfest.postgresql.org/patch/3684/) (cf, Returned with feedback, 2022-10-12) - used in the review; reason summarised from its thread
- P3: [Fix search_path for maintenance commands](https://commitfest.postgresql.org/patch/4367/) (cf, Withdrawn, 2024-01-18) - used in the review; reason summarised from its thread
- P4: [Add CREATE SCHEMA ... LIKE support](https://commitfest.postgresql.org/patch/6481/) (cf, Returned with feedback, 2026-04-06) - judged not relevant; reason summarised from its thread
- P5: [Tab completion for CREATE SCHEMA AUTHORIZATION](https://commitfest.postgresql.org/patch/3252/) (cf, Returned with feedback, 2021-12-03) - judged not relevant; reason summarised from its thread
- P6: [Support % wildcard in extension upgrade scripts](https://commitfest.postgresql.org/patch/3654/) (cf, Returned with feedback, 2023-08-01) - judged not relevant; reason summarised from its thread
- P7: [Allow excluding pg_temp or pg_catalog from search_path.](https://commitfest.postgresql.org/patch/4508/) (cf, Withdrawn, 2023-11-17) - used in the review; reason summarised from its thread
- P8: [allow building trusted languages without the untrusted versions](https://commitfest.postgresql.org/patch/3647/) (cf, Returned with feedback, 2022-08-01) - judged not relevant; reason summarised from its thread
- P9: [Revert MAINTAIN privilege and pg_maintain predefined role.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=151c22deee66a3390ca9a1c3675e29de54ae73fc) (revert, Reverted, 2023-07-07) - used in the review
- P10: [Support % wildcard in extension upgrade scripts](https://commitfest.postgresql.org/patch/3892/) (cf, Withdrawn, 2022-09-14) - judged not relevant
- P11: [Revert "Fix search_path to a safe value during maintenance operations."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2fcc7ee7affe2fb708d7b2727e93981216b5d2f8) (revert, Reverted, 2023-06-10) - used in the review
- P12: [Support % wildcard in extension upgrade script filenames](https://commitfest.postgresql.org/patch/4489/) (cf, Returned with feedback, 2024-02-01) - judged not relevant
- P13: [Various small unrelated doc patches: plpgsql, schemas, permissions, oidvector](https://commitfest.postgresql.org/patch/4581/) (cf, Returned with feedback, 2024-10-07) - judged not relevant
- P14: [Strip Windows newlines from extension script files manually.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6cfb3a33746990602c790199adc2debb2c4bbb87) (revert, Partially reverted, 2024-10-28) - judged not relevant
- P15: [psql: add schema size information to \dn+](https://commitfest.postgresql.org/patch/2010/) (cf, Withdrawn, 2019-02-25) - judged not relevant
- P16: [pg_get_constraintdef: Schema qualify foreign tables unless pretty printing is enabled](https://commitfest.postgresql.org/patch/3811/) (cf, Returned with feedback, 2022-11-30) - judged not relevant
- P17: [Revert "Forbid DROP SCHEMA on temporary namespaces"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=65192e02441cedd106b6abebe0036fb8cc124fb3) (revert, Reverted, 2020-01-08) - judged not relevant
- P18: [revert "warn of SECURITY DEFINER schemas for non-sql_body funcs"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f77becbe5a99c9853b1ae7f22aa15babd01368b7) (revert, Reverted, 2022-09-28) - judged not relevant
- P19: [Use "protocol options" name instead of "protocol extensions" everywhere](https://commitfest.postgresql.org/patch/5341/) (cf, Rejected, 2025-03-11) - judged not relevant
- P20: [CREATEROLE and role ownership hierarchies](https://commitfest.postgresql.org/patch/3506/) (cf, Returned with feedback, 2022-04-12) - judged not relevant
- P21: [USAGE privilege on PUBLICATION](https://commitfest.postgresql.org/patch/3641/) (cf, Withdrawn, 2024-01-15) - judged not relevant
- P22: [extension patch of CREATE OR REPLACE TRIGGER](https://commitfest.postgresql.org/patch/2034/) (cf, Returned with feedback, 2019-09-30) - judged not relevant
- P23: [Revert "Ignore object files generated by ecpg test suite on Windows"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2093eb4d4c2b55962df613fdd6f605ce4dc67217) (revert, Reverted, 2015-03-09) - judged not relevant
- P24: [Revert "Avoid race condition between "GRANT role" and "DROP ROLE"".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=8abbbbae610cd62faad4f9ec18e7cfa908593796) (revert, Reverted, 2025-09-16) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 13; wall time: 149 s; cost reported by the CLI: $1.98
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 7 from cache, 1 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 5 of 24 kept
- Tone check: 83 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:35:42.042Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

# Design pre-review: Prevent object capture in CREATE/ALTER EXTENSION scripts

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7276](https://commitfest.postgresql.org/patch/7276/); [thread](https://www.postgresql.org/message-id/flat/28f66580-5d6b-4997-95f6-7d9e7917c585%40planetscale.com), 7 of 7 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). The problem is stated and demand is shown, so levels 1a and 1b pass. Four fixable level 2 objections survived: security-1 (an ordinary role can add its own objects to a required extension and have them trusted), security-2 (a lookup path that skips the check fails silently), standards-compat-1 and standards-compat-2. None requires a different proposal, but the trust rule sits in core name resolution and needs a committer who knows that code.

Not checked: The cost of the trust check and the pg_depend lookup, the completeness of the filtered lookup paths, and the plancache handling could not be judged without the code. The answer on ALTER EXTENSION ... ADD may also depend on details of the membership test.

## Prior attempts

- [Fix search_path for maintenance commands](https://www.postgresql.org/message-id/flat/e44327179e5c9015c8dda67351c04da552066017.camel@j-davis.com) (related; Withdrawn, 2024-01-18). "I am withdrawing this patch from the CF until it's more clear that this is what we really want to do." ([message](https://www.postgresql.org/message-id/b7244fd967fbec96ead8d4546e1d1e93e5eb82b7.camel%40j-davis.com)) Different now: P3 forced search_path to "pg_catalog, pg_temp" during maintenance commands. This proposal keeps search_path and filters candidates by owner during extension scripts, so it also blocks plants in the script's own schema.

## Objections a reviewer is likely to raise

### 1. Membership in a required extension can be granted by an ordinary role

An ordinary role installs a trusted extension that another extension requires, so it owns that extension. It creates f(text) in the shared schema and runs alter extension ... add function f(text). A superuser or platform later installs the dependent extension, and its script resolves f('abc') to the planted function because it is a member of a required extension.

The trust rule in message #1 accepts any member of a required extension, "regardless of who owns them". alter extension add needs only ownership of the extension and of the object. A non-superuser who installs a trusted extension owns it, so membership is a property an attacker can set. The hosted-platform case the proposal targets is where unprivileged users install trusted extensions on their own.

**Question:** How does the trust rule stop an ordinary role that owns a required extension from adding its own objects to it and having them trusted?

<sub>fixable, level 2, security / privileges; messages [#1](https://www.postgresql.org/message-id/28f66580-5d6b-4997-95f6-7d9e7917c585%40planetscale.com); `security-1`</sub>

### 2. Each lookup path must opt in, and a missed path fails silently

A later patch adds a new lookup route, or changes an existing one, without calling the trust check. Scripts on that route go back to resolving planted objects. No test fails, and the gap shows up as a security report against a guarantee the project now documents.

The protection depends on every relevant lookup site calling the check. v1 already missed the qualified branches of RangeVarGetRelidExtended() and LookupTypeNameExtended() (message #7). Message #1 leaves operator classes, collations, text search objects, casts and conversions unfiltered and says a route may have been missed. Once the guarantee is committed, the project has to keep it complete in all of these places.

**Question:** What makes a new or changed name-lookup path fail loudly, in tests or by construction, when it skips the trust check during extension scripts?

<sub>fixable, level 2, security / privileges; messages [#1](https://www.postgresql.org/message-id/28f66580-5d6b-4997-95f6-7d9e7917c585%40planetscale.com), [#7](https://www.postgresql.org/message-id/3f6db46e-3175-40de-9001-ccde3a8b05f2%40planetscale.com); `security-2`</sub>

### 3. Existing scripts can stop working on update, and there is no escape hatch

An extension script references an object owned by an ordinary role, for example a function of a superuser=false extension that is optional or only a transitive dependency. After the server upgrade, ALTER EXTENSION ... UPDATE fails with "does not exist". On a hosted platform the user cannot change object ownership or edit the script, so the update stays blocked.

Message #1 names this behaviour change and calls such references "rare", but gives no survey of contrib or third-party scripts. The proposal has no control-file option or other way to mark an object as trusted. The only fixes are to reassign ownership to a superuser or to add a requires entry, and neither is open to the user who hits the error. The commit message carries Reported-by, so a committer would also likely ask whether this is meant for back branches.

**Question:** Which existing extension scripts reference objects of ordinary roles outside their requires list, and how does a user unblock an update when one fails?

<sub>fixable, level 2, design; messages [#1](https://www.postgresql.org/message-id/28f66580-5d6b-4997-95f6-7d9e7917c585%40planetscale.com); `standards-compat-1`</sub>

### 4. The requires list becomes a name-visibility rule

Extension A requires B, and B requires C, where C is a superuser=false extension owned by an ordinary role. A's script calls a C function by its schema-qualified name. With this patch the call fails, although A installs only when C is present. The extension author finds out from a user's failed CREATE EXTENSION.

Today requires controls install order and @extschema:name@ substitution. Under 0002 it also decides which objects a script can see, and only for direct entries, as message #1 says. Extension authors would have to list transitive and conditional dependencies to keep scripts working. No documentation hunk is shown, so the new meaning of requires does not yet appear in the extension-authoring docs.

**Question:** Why is trust limited to direct requires entries, and where would the new meaning of requires be documented for extension authors?

<sub>fixable, level 2, design; messages [#1](https://www.postgresql.org/message-id/28f66580-5d6b-4997-95f6-7d9e7917c585%40planetscale.com); `standards-compat-2`</sub>

## Questions for the author

1. How does the trust rule stop an ordinary role that owns a required extension from adding its own objects with ALTER EXTENSION ... ADD and having them trusted?
2. What makes a new or changed name-lookup path fail loudly, in tests or by construction, when it skips the trust check during an extension script?
3. Which contrib or third-party extension scripts reference objects owned by ordinary roles outside their requires list, and how does a user unblock a failed update?
4. Why does trust stop at direct requires entries, and where would the docs tell extension authors about this new meaning of requires?
5. Is this meant for back branches or only for master, given that it can make existing update scripts fail?

## Touch points

- **catalog changes:** The trust rule reads pg_depend, and 0002 touches namespace.c and pg_operator.c; no catalog change is described.
- **ABI / extension API:** 0002 changes namespace.h, parse_type.h, plancache.h and extension.h, which extensions include.
- **planner behaviour change:** CachedPlanSource records whether a plan was analyzed inside an extension script, which affects plan reuse.
- **security / privileges:** The patch adds an ownership-based trust rule to stop privilege escalation through planted objects in extension scripts.
- **backpatchability:** The commit message carries Reported-by for a security issue, but the change can break existing extension scripts.
- **maintenance burden ("who maintains this in 10 years"):** Each relation, type, function and operator lookup path and each cache must apply the trust check from now on.
- **performance on the hot path:** The check sits in name lookup and plancache code; the cost outside extension scripts is not measured.

## Who would commit it, and when

- Owner: silent. Someone who knows namespace.c, parser name resolution and plancache, and who is willing to sign for a security guarantee about extension scripts.
- Priority: needs champion; audience a sizeable minority; review cost high. Hosted platforms that run extension scripts as superuser benefit, and two reviewers reproduced the capture. The 83.5 KB patch changes namespace.c, the parser and plancache, and no committer has replied in the visible thread.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 3 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

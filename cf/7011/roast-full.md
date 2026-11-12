# Design pre-review: pg_hosts: Add pg_hosts_file_rules()

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7011](https://commitfest.postgresql.org/patch/7011/) (Needs review); authors: Zsolt Parragi (zsolt.parragi)
- Thread: [pg_hosts: Add pg_hosts_file_rules()](https://www.postgresql.org/message-id/flat/CAN4CZFNoqV6cwOH5UfatRwbvcw3BooxW8QyVECkzZmExtjmfqA%40mail.gmail.com), started 2026-07-08
- Messages read: 9 of 9
- Patch prose read: v5-0001-Add-pg_hosts_file_rules-to-inspect-the-hosts-conf.patch, 0001-Add-pg_hosts_file_rules-to-inspect-the-hosts-configu.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium)

No objection survived the critic. The only objection raised, maintenance-fit-1, was refuted because it depends on a future pg_hosts.conf format change that the thread mentions only as an idea. The view copies the shipped pg_hba_file_rules design, and the thread settled the rule_number column and the behaviour on builds without SNI. Confidence is medium because the code was not read and only 3 of 9 messages are shown.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, the privileges on the function, the handling of a missing or included file, and the parser changes in be-secure-common.c could not be checked. Messages #4 to #9 were not shown, so the brief's summary of them is taken as given.

## The proposal, restated

**Problem.** Postgres has pg_hba_file_rules for pg_hba.conf but no matching view for pg_hosts.conf, the hosts configuration file that maps SNI host names to SSL certificate settings. The author noticed the gap while working on a configuration file format proposal. Administrators cannot check from SQL whether the current file parses.

**Proposed design.** The patch adds a set-returning function and a system view over it, built the same way as pg_hba_file_rules. The view parses the current file on disk, not what the server last loaded, and shows one row per entry with an error column for bad lines. On builds without SNI support, such as LibreSSL, the file is not parsed and the view is always empty.

**User-visible surface.**
- New system view pg_hosts_file_rules, readable only by superusers by default.
- New function pg_hosts_file_rules() in pg_proc, used by the view definition.
- View columns: file_name, line_number, hostnames text[], ssl_certificate, ssl_key, ssl_ca, passphrase_command, passphrase_command_reload, error.
- The rule_number column from v1 was dropped in later versions because pg_hosts.conf is unordered.

**What the author claims.**
- The view gives pg_hosts.conf the same inspection tool that pg_hba_file_rules gives pg_hba.conf.
- Users can check planned file changes before a reload, or diagnose a failed load.
- Leaving the view empty on builds without SNI avoids telling users a file is valid when SNI cannot work.

**Thread state.** Daniel Gustafsson supported the idea, reviewed several versions, and called it nearly ready for v20 after two nits, which v4 addressed. solai v tested v3 without issues; v5 fixed an EXEC_BACKEND test failure and no one has replied since.

## 2. Prior attempts

### pg_hba_file_settings (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/723/](https://commitfest.postgresql.org/patch/723/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CAJrrPGerH4jiwpcXT1-46QXUDmNp2QDrG9+-Tek_xC8APHShYw@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAJrrPGerH4jiwpcXT1-46QXUDmNp2QDrG9+-Tek_xC8APHShYw@mail.gmail.com)
- **Date:** decided 2016-10-03; started 2016-08-15
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Michael Paquier (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Michael Paquier moved it to the next CF with no reviews, then marked it returned with feedback. He questioned the use of jsonb for the options column, since the options are not nested and a simple name=value array would do. Vitaly Burovoy argued for jsonb to make filtering easier, but the thread ended without a revised patch.
- **Reason (quote):** "If they were nested I'd see a case for it but it seems to me that as presented this is just an overkill. In short, I think that this patch needs a bit of rework, so I am marking it as returned with feedback." ([message](https://www.postgresql.org/message-id/CAB7nPqQ562wDzD7AqTMDsi13gzHYp%3D-me%2Brj0gPWEEx7yH59OQ%40mail.gmail.com))
- **What is different this time:** P6 was an early pg_hba view that stalled over a jsonb options column. This proposal uses flat text and bool columns and copies the pg_hba_file_rules design that Postgres later shipped.

### catalog view to pg_hba.conf file (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/125/](https://commitfest.postgresql.org/patch/125/)
- **Thread:** [https://www.postgresql.org/message-id/flat/F40B0968DB0A904DA78A924E633BE78645FE29@SYDEXCHTMP2.au.fjanz.com](https://www.postgresql.org/message-id/flat/F40B0968DB0A904DA78A924E633BE78645FE29@SYDEXCHTMP2.au.fjanz.com)
- **Date:** decided 2015-07-07; started 2014-03-14
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Peter Eisentraut, Stephen Frost (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Reviewers were unconvinced that the mapping of pg_hba.conf onto a table was right: Peter Eisentraut found the keyword columns, compare_method and options representation confusing and questioned why a processed view was better than the raw file if it could not answer "what auth method applies to user X from address Y". Stephen Frost shared the concern about splitting up keywords and asked for a lookup function as well. The author agreed to rework the view and add such a function for 9.6, but no new patch arrived in the July commitfest, so it was returned with feedback.
- **Reason (quote):** "Ultimately, I don't see how this is better than just showing the raw file. I don't see a sensible way to compute answers to questions such as "What is the authentication method for user X from IP address Y." ([message](https://www.postgresql.org/message-id/5547DB0A.2020904%40gmx.net))
- **What is different this time:** P1 showed the loaded pg_hba.conf, and reviewers found its mapping of keywords and options onto columns confusing. pg_hosts.conf entries are flat and unordered, and this view parses the file on disk as pg_hba_file_rules does.


## 3. Objections a reviewer is likely to raise

No objection held up strongly enough in the counter-check to list here.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | **yes** | The patch adds a pg_proc.dat entry and a system view in system_views.sql, so it needs a catversion bump. |
| ABI / extension API | **yes** | The patch changes src/include/libpq/hba.h, a header that extensions can include. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | **yes** | The view shows ssl_key paths and passphrase_command text, and is readable only by superusers by default. |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | The view has one column per pg_hosts.conf field, so a new field in the file needs a new column. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Does system_views.sql revoke EXECUTE on pg_hosts_file_rules() from public, as it does for pg_hba_file_rules()?
2. Should the docs warn that granting SELECT on the view exposes passphrase_command text and private key paths?
3. What does the view return when pg_hosts.conf is missing: no rows, an error row, or a thrown error?
4. If pg_hosts.conf supports include directives, do entries from included files appear with their own file_name and line_number?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Daniel Gustafsson supported the idea in message #2 and reviewed several versions; the owner needs to know the SSL and SNI code and the hba file parsing code.
- **Liability:** One superuser-only view and function whose columns must follow the pg_hosts.conf format in every later release, plus the rule that the view stays empty on builds without SNI.
- **Author follow-up:** The author posted five versions in response to review and moved the entry to the next CommitFest, and v5 fixed an EXEC_BACKEND test failure.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch is about 21 KB, touches eight files and reuses the pg_hba_file_rules mechanism in hbafuncs.c. Only administrators who use SNI with pg_hosts.conf benefit, the cost falls only on people who query the view, and v5 has no reply yet.

## Appendix

### Refuted in the counter-check

- `maintenance-fit-1` Fixed per-setting columns while the pg_hosts.conf format may grow. Refuted (no failure scenario): The scenario depends on a future format change that the material only mentions as an idea in message #3. The documented columns map one to one to today's entries, as the fixed columns of pg_hba_file_rules do, and Postgres adds system view columns in major releases routinely.

### Review angles

- prior-attempts: 0 objection(s). P1 and P6 were earlier pg_hba attempts at a file-inspection view. Their recorded reasons were a jsonb options column and a confusing mapping of hba keywords to columns, and neither applies to the flat, unordered pg_hosts.conf. The view copies the shipped pg_hba_file_rules design, the thread (messages #2, #4, #7) settled the rule_number and LibreSSL questions, and P2 concerns file comments, which this proposal does not expose.
- standards-compat: 0 objection(s). The view copies pg_hba_file_rules and pg_ident_file_mappings in naming, superuser-only default, error column and on-disk parsing, so it follows existing practice and changes nothing for existing users. The thread settled the empty result on builds without SNI and the dropped rule_number column (messages #2-#4). Fixed per-setting columns may need changes if the config rework adds pg_hosts.conf options, but changing system view columns between major versions is normal. Without the code, I could not check how a missing file or include files appear.
- replication-ondisk: 0 objection(s). The proposal holds no persistent or cross-node state. It adds one pg_proc entry and one system view, both created by initdb and never dumped by pg_dump or carried over by pg_upgrade, and there is no WAL, on-disk format, wire protocol or replication change. A standby reads its own local pg_hosts.conf the same way pg_hba_file_rules reads pg_hba.conf, and the hba.h change targets a new major version only.
- security: 0 objection(s). No security objection. The view is superuser-only by default, like pg_hba_file_rules, and a superuser can already read pg_hosts.conf and its passphrase settings. The code is not shown, so two points stay unchecked: whether system_views.sql revokes EXECUTE on pg_hosts_file_rules() from public, and whether the docs warn that granting the view exposes passphrase_command text.
- maintenance-fit: 1 objection(s). Apart from this point, the design follows pg_hba_file_rules and pg_ident_file_mappings. It adds no hot-path cost and no new dependency, and a committer who knows the area (message #2, #7) is engaged. So the maintenance and fit case against it is weak.
- already-exists: 0 objection(s). No existing feature gives this result. pg_read_file() returns the raw pg_hosts.conf text but does not run the server's parser, and a reload applies changes instead of checking them first. The patch reuses the pg_hba_file_rules mechanism in hbafuncs.c, so it extends an existing one. An extension would have to duplicate the core parser.
- demand: 0 objection(s). I have no demand objection. The material shows no field reports, but the view copies pg_hba_file_rules and pg_ident_file_mappings for a sibling configuration file, and a committer who knows the area (message #2, #7) supports it. The cost is one superuser-only view and function, paid only by people who query it.

### Precedents retrieved

- P1: [catalog view to pg_hba.conf file](https://commitfest.postgresql.org/patch/125/) (cf, Returned with feedback, 2015-07-07) - used in the review; reason summarised from its thread
- P2: [Add inline comments to the pg_hba_file_rules view](https://commitfest.postgresql.org/patch/4546/) (cf, Withdrawn, 2023-10-10) - used in the review; reason summarised from its thread
- P3: [Document hostssl related concepts better](https://commitfest.postgresql.org/patch/5549/) (cf, Withdrawn, 2025-03-09) - judged not relevant; reason summarised from its thread
- P4: [Sample configuration files](https://commitfest.postgresql.org/patch/745/) (cf, Returned with feedback, 2016-09-30) - judged not relevant; reason summarised from its thread
- P5: [Configurable file mode mask](https://commitfest.postgresql.org/patch/1069/) (cf, Returned with feedback, 2017-03-21) - judged not relevant; reason summarised from its thread
- P6: [pg_hba_file_settings](https://commitfest.postgresql.org/patch/723/) (cf, Returned with feedback, 2016-10-03) - used in the review; reason summarised from its thread
- P7: [Configurable location for extension .control files](https://commitfest.postgresql.org/patch/170/) (cf, Returned with feedback, 2015-07-30) - judged not relevant; reason summarised from its thread
- P8: [hostorder and failover_timeout for libpq](https://commitfest.postgresql.org/patch/1631/) (cf, Returned with feedback, 2019-02-03) - judged not relevant; reason summarised from its thread
- P9: [Revert rules.out to non-broken state. I will with difficulty hold my tongue about the sloppiness of (a) submitting and (b) committing this file.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=d8e582e23688ad2fad7b2d7107ee0d60becb799e) (revert, Reverted, 2000-10-08) - judged not relevant
- P10: [Support UTF16/32 host variable in ECPG](https://commitfest.postgresql.org/patch/1337/) (cf, Returned with feedback, 2017-11-28) - judged not relevant
- P11: [Make it possible to use RETURNING together with conditional DO INSTEAD rules, such as for partitioning setups](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Views_and_Rules) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P12: [Allow VIEW/RULE recompilation when the underlying tables change](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Views_and_Rules) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P13: [Support pg_ident mapping for LDAP](https://commitfest.postgresql.org/patch/3314/) (cf, Withdrawn, 2022-08-17) - judged not relevant
- P14: [lax reloptions parsing for views](https://commitfest.postgresql.org/patch/3340/) (cf, Returned with feedback, 2022-03-31) - judged not relevant
- P15: [Add pg_ownerships and pg_privileges system views](https://commitfest.postgresql.org/patch/5033/) (cf, Withdrawn, 2025-01-04) - judged not relevant
- P16: [Set host(no)ssl settings from initdb](https://commitfest.postgresql.org/patch/2378/) (cf, Returned with feedback, 2020-09-07) - judged not relevant
- P17: [Map WAL segment files on PMEM as WAL buffers](https://commitfest.postgresql.org/patch/3181/) (cf, Returned with feedback, 2022-08-01) - judged not relevant
- P18: [pg_stat_walwrites system view](https://commitfest.postgresql.org/patch/1003/) (cf, Returned with feedback, 2017-11-27) - judged not relevant
- P19: [Let people set host(no)ssl settings from initdb](https://commitfest.postgresql.org/patch/2916/) (cf, Rejected, 2021-03-09) - judged not relevant
- P20: [Implementing OCSP Stapling for Server certificate status verification](https://commitfest.postgresql.org/patch/4849/) (cf, Returned with feedback, 2024-10-08) - judged not relevant
- P21: [Add checkpoint stats of snapshot and mapping files of pg_logical dir](https://commitfest.postgresql.org/patch/3389/) (cf, Withdrawn, 2022-10-11) - judged not relevant
- P22: [Reload SSL certificates on SIGHUP](https://commitfest.postgresql.org/patch/259/) (cf, Returned with feedback, 2015-10-03) - judged not relevant
- P23: [Add system views pg_permissions and pg_ownerships](https://commitfest.postgresql.org/patch/3033/) (cf, Returned with feedback, 2022-04-08) - judged not relevant
- P24: [Log SSL certificate verification errors](https://commitfest.postgresql.org/patch/1374/) (cf, Returned with feedback, 2018-03-02) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 14; wall time: 85 s; cost reported by the CLI: $1.56
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 5 from cache, 3 summarised now, 0 failed; quotes: 3 verified, 0 dropped; relevance pass: done, 3 of 24 kept
- Tone check: 53 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:14:22.569Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

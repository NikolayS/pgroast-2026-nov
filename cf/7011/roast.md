# Design pre-review: pg_hosts: Add pg_hosts_file_rules()

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7011](https://commitfest.postgresql.org/patch/7011/); [thread](https://www.postgresql.org/message-id/flat/CAN4CZFNoqV6cwOH5UfatRwbvcw3BooxW8QyVECkzZmExtjmfqA%40mail.gmail.com), 9 of 9 messages read; no code read.

## Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium). No objection survived the critic. The only objection raised, maintenance-fit-1, was refuted because it depends on a future pg_hosts.conf format change that the thread mentions only as an idea. The view copies the shipped pg_hba_file_rules design, and the thread settled the rule_number column and the behaviour on builds without SNI. Confidence is medium because the code was not read and only 3 of 9 messages are shown.

Not checked: Without the code, the privileges on the function, the handling of a missing or included file, and the parser changes in be-secure-common.c could not be checked. Messages #4 to #9 were not shown, so the brief's summary of them is taken as given.

## Prior attempts

- [pg_hba_file_settings](https://www.postgresql.org/message-id/flat/CAJrrPGerH4jiwpcXT1-46QXUDmNp2QDrG9+-Tek_xC8APHShYw@mail.gmail.com) (similar idea; Returned with feedback, 2016-10-03). "If they were nested I'd see a case for it but it seems to me that as presented this is just an overkill. In short, I think that this patch needs a bit of rework, so I am marking it as returned with feedback." ([message](https://www.postgresql.org/message-id/CAB7nPqQ562wDzD7AqTMDsi13gzHYp%3D-me%2Brj0gPWEEx7yH59OQ%40mail.gmail.com)) Different now: P6 was an early pg_hba view that stalled over a jsonb options column. This proposal uses flat text and bool columns and copies the pg_hba_file_rules design that Postgres later shipped.
- [catalog view to pg_hba.conf file](https://www.postgresql.org/message-id/flat/F40B0968DB0A904DA78A924E633BE78645FE29@SYDEXCHTMP2.au.fjanz.com) (similar idea; Returned with feedback, 2015-07-07). "Ultimately, I don't see how this is better than just showing the raw file. I don't see a sensible way to compute answers to questions such as "What is the authentication method for user X from IP address Y." ([message](https://www.postgresql.org/message-id/5547DB0A.2020904%40gmx.net)) Different now: P1 showed the loaded pg_hba.conf, and reviewers found its mapping of keywords and options onto columns confusing. pg_hosts.conf entries are flat and unordered, and this view parses the file on disk as pg_hba_file_rules does.

## Objections a reviewer is likely to raise

No objection held up in the counter-check.

## Questions for the author

1. Does system_views.sql revoke EXECUTE on pg_hosts_file_rules() from public, as it does for pg_hba_file_rules()?
2. Should the docs warn that granting SELECT on the view exposes passphrase_command text and private key paths?
3. What does the view return when pg_hosts.conf is missing: no rows, an error row, or a thrown error?
4. If pg_hosts.conf supports include directives, do entries from included files appear with their own file_name and line_number?

## Touch points

- **catalog changes:** The patch adds a pg_proc.dat entry and a system view in system_views.sql, so it needs a catversion bump.
- **ABI / extension API:** The patch changes src/include/libpq/hba.h, a header that extensions can include.
- **security / privileges:** The view shows ssl_key paths and passphrase_command text, and is readable only by superusers by default.
- **maintenance burden ("who maintains this in 10 years"):** The view has one column per pg_hosts.conf field, so a new field in the file needs a new column.

## Who would commit it, and when

- Owner: qualified committer engaged. Daniel Gustafsson supported the idea in message #2 and reviewed several versions; the owner needs to know the SSL and SNI code and the hba file parsing code.
- Priority: proceed; audience niche; review cost low. The patch is about 21 KB, touches eight files and reuses the pg_hba_file_rules mechanism in hbafuncs.c. Only administrators who use SNI with pg_hosts.conf benefit, the cost falls only on people who query the view, and v5 has no reply yet.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 1 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

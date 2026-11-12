# Design pre-review: REVOKE's CASCADE protection doesn't work with inherited table owners

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6961](https://commitfest.postgresql.org/patch/6961/); [thread](https://www.postgresql.org/message-id/flat/CAOYmi%2B%3DKTLd%2BXsEP%3DTDiZ48iVf-CEc7JrZd5uhWPYWKEfOgyyQ%40mail.gmail.com), 7 of 7 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). Nothing that survived questions the fix itself: the problem is reproducible, and the old behaviour leaves ACL entries that pg_dump cannot restore. The one surviving objection, prior-attempts-1, is about back-patching to 14. A REVOKE that succeeds today would fail in a minor release, and the thread neither weighs that nor gives release-note text. That is a stable-branch policy decision for a committer who knows the privilege code.

Not checked: The code was not read. This review cannot confirm that aclmask_direct() gives the intended result for every grantor case, including SUPERUSER and PUBLIC, or that no legitimate REVOKE now fails.

## Prior attempts

None found before 2026-06-24.

## Objections a reviewer is likely to raise

### 1. Back-patched REVOKE behavior change can break existing scripts

An admin's deployment script runs `revoke all on admin_table from bob`, where bob's grant option comes through role membership or SUPERUSER. After a minor update to 14.x through 18.x, that statement fails with "dependent privileges exist". The migration aborts, and the admin finds out only on the first deploy after the update.

Message #2 proposes back-patching "all the way down", and the commit message says Backpatch-through: 14. The thread does not discuss whether a REVOKE that used to succeed should start failing in a minor release, and no release-note text is given. [P1] is a privileges change back-patched to 14 that was reverted because it "broke existing behavior". Here the old behavior leaves ACLs that pg_dump cannot restore, which is a stronger case for the fix, but a committer would likely still weigh master-only against back-patching.

Precedent: [Revert b2a459edf "Fix GRANTED BY support in REVOKE ROLE statements"](https://www.postgresql.org/message-id/flat/17346-f72b28bd1a341060@postgresql.org) (Reverted, 2021-12-30)

**Question:** Why should existing REVOKE RESTRICT scripts fail in minor releases rather than only on master, and what will the release notes tell affected users?

<sub>fixable, level 2, backpatchability; messages [#2](https://www.postgresql.org/message-id/CAOYmi%2Bn_hw%3DSC5V1i3BmqfZPfPBRaUSJc%2BBeOXEKDwRue%2BWYrg%40mail.gmail.com), [#7](https://www.postgresql.org/message-id/179074273441.6180.2515281464552050662%40mail.gmail.com); `prior-attempts-1`</sub>

## Questions for the author

1. Why should the stricter REVOKE reach stable branches 14 through 18 and not only master, given that scripts that succeed today will fail after a minor update?
2. What release-note text will tell users that REVOKE without CASCADE can now fail, and that REVOKE ... CASCADE now removes dependent grants it used to leave?
3. Do other grant-option checks that call aclmask() have the same flaw as recursive_revoke()?
4. How can an administrator find ACL entries that this bug already orphaned in an existing cluster before running pg_dump or pg_upgrade?

## Touch points

- **pg_dump / restore round-trip:** The fix stops REVOKE from leaving ACL entries that pg_dump output cannot recreate on restore.
- **security / privileges:** recursive_revoke() in acl.c changes which grant options count when REVOKE checks for dependent privileges.
- **backpatchability:** The commit message says Backpatch-through: 14, and REVOKE without CASCADE starts failing in cases that succeed today.

## Who would commit it, and when

- Owner: silent. The patch needs a committer who knows the ACL and grant-option code in acl.c and the back-patch rules for behaviour changes. None has replied in the visible thread.
- Priority: needs champion; audience niche; review cost low. The patch changes one call in acl.c and adds regression tests. It has two LGTMs but no committer reply after three CommitFests, and only a committer can settle the back-patch question.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 4 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

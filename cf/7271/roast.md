# Design pre-review: Check output_plugin_libraries in pg_createsubscriber

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7271](https://commitfest.postgresql.org/patch/7271/); [thread](https://www.postgresql.org/message-id/flat/OS9PR01MB12149618ED793133C85A61A34F5B52%40OS9PR01MB12149.jpnprd01.prod.outlook.com), 46 of 46 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No objection survived at levels 1a or 1b, and the goal has support from two committers in the thread. The one surviving objection, standards-compat-1, is a fixable design point: the frontend parse of output_plugin_libraries can stop the tool on a value the server accepts today. A committer who knows this code can settle that point, together with the backpatch question.

Not checked: The v7 code patch prose is not in the material, so the final error text, the skip conditions and the parse-failure branch could not be checked. Whether the frontend and the server match a quoted or path-qualified pgoutput entry the same way is also unchecked.

## Prior attempts

None found before 2026-09-04.

## Objections a reviewer is likely to raise

### 1. Client-side list parsing can reject settings the server accepts

A DBA sets output_plugin_libraries = 'pgoutput, test_decoding, ' on the publisher. The server accepts this value and pgoutput slots work. With the patch, pg_createsubscriber stops with a fatal parse error in --dry-run and in the real run, so a conversion that worked before now fails.

The patch re-parses a server GUC in the frontend with SplitGUCList, so the tool's view of the setting can differ from the server's. Message #29 shows that the trailing-comma value reaches the parse-failure branch. Message #31 calls the server behaviour a bug and plans a fix, but the material does not say whether that fix lands in the same release. The tool also turns a parse failure into a hard error, though the run would succeed today.

**Question:** Will the server-side fix that rejects empty list elements land before or with this patch, and if not, why should a parse failure stop the tool instead of skipping the check?

<sub>fixable, level 2, design; messages [#20](https://www.postgresql.org/message-id/CAOYmi%2BnFptx-1xTxMqatHgRuLAc3JKZo--FzXZvKcrpCjBc%3DBQ%40mail.gmail.com), [#29](https://www.postgresql.org/message-id/CAHGQGwG81iHN46mY_19Bne5RuKfAjH4dOnQ7-8i%3D7KYJk_0rAg%40mail.gmail.com), [#30](https://www.postgresql.org/message-id/TYRPR01MB128459BA258A78D03C5365B2EF5B92%40TYRPR01MB12845.jpnprd01.prod.outlook.com), [#31](https://www.postgresql.org/message-id/CAOYmi%2Bkrun4a8ACxcP4MtvPU6SMrowwENSvh7yzc5mhEJ85k_g%40mail.gmail.com); `standards-compat-1`</sub>

## Questions for the author

1. Will the server fix that rejects values like 'pgoutput, test_decoding, ' land with this patch, and if not, why should a parse failure stop the tool instead of skipping the check?
2. Is the patch now master only, or does the author still ask for a backpatch to v17 and v18?
3. Should the tool report a skipped check, for example at debug level with -v, when it cannot read output_plugin_libraries?
4. Does the frontend match pgoutput the same way the server does when the list entry is quoted or carries a path?
5. Can the CF entry carry the v7 code patch so that cfbot tests it, since the entry now shows only the doc patch?

## Touch points

- **security / privileges:** The tool reads a security allow list through pg_settings and skips the check silently when the role cannot see it.
- **backpatchability:** The CF entry is tagged Backport and the author proposes v17 and v18, while three reviewers favour master only.
- **maintenance burden ("who maintains this in 10 years"):** pg_createsubscriber.c parses the list with SplitGUCList, so frontend and server parsing must stay compatible.

## Who would commit it, and when

- Owner: qualified committer engaged. Fujii Masao and Jacob Champion take part in the thread; the owner needs to know pg_createsubscriber and the output_plugin_libraries setting from commit 226e49cb.
- Priority: proceed; audience niche; review cost low. Only users who run pg_createsubscriber against a publisher with pgoutput removed from the list gain, but the patch is small and touches one tool. Two committers are engaged and a reviewer tested v7.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.

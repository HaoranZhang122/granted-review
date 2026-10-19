# The demo forms

A complete round trip of the fictional review in `../` (an online shop moves its orders database). Open the HTML files in a browser. They were built with `link_base` set to `../`, so that their links reach `../report.md` from this folder.

| File | What it is |
|---|---|
| `acme-db-move_R1.0_2026-10-19.html` | Round 1 as the AI issued it: 13 cards, no answers. Try it: answer, then click **Seal & send**. |
| `acme-db-move_R1.0_2026-10-19_answered.html` | The same form as a (fictional) reviewer saved it: 9 items confirmed, 4 open. |
| `DECISIONS_after_R1.md` | What `granted.py next-round` wrote from those answers. |
| `acme-db-move_R2.0_2026-10-19.html` | Round 2: the 9 confirmed items in one table, and the 4 open cards, each with the reviewer's previous answer and the AI's reply. |

The empty and the answered R1.0 are the same form, so they share the answers that your browser keeps: if you try the empty one, the answered one then offers your answers instead (click "use the answers in the file" to see the saved ones). R2.0 keeps its answers apart. To start again, clear the site data of the page in your browser.

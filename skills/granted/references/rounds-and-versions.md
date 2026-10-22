# Rounds, versions and files

## Version labels

| Label | Meaning |
|---|---|
| `R1.0` | The first form of round 1. |
| `R1.1`, `R1.2`, … | Revisions of round 1. They fix wording or layout only. The answers of the round are carried over (`build --carry`), except on a card whose chosen option got a new label: that card asks again. |
| `R2.0` | The first form of round 2: the confirmed items in one table, plus one card for each open item. |
| `R<n>.<m>` | Round n, revision m. |

**Rules:**
- A revision never changes the content or the options of a card that the reader answered. A change of content starts a new round.
- An option keeps its key in every version and round. A changed option gets a new key. `check` compares each card set with the previous issued one and warns when an option label changes.
- The form id is `<slug>-<version>-<date>` (the UTC date of the build). A draft has the id `<slug>-<version>-<date>-draft`. Each review also has a random `review_id` (in `granted.json`). The browser keeps the answers under the review id and the form id, so answers from different reviews, versions and drafts never mix.
- Each card in a form carries a fingerprint of its content. `receive` compares it with the issued card set, so an answer always belongs to the card set that the form was built from. (The fingerprint does not prove that nobody edited the visible text of a returned file by hand. Keep returned files as the user sent them.)
- A revision offers the answers that the reader typed into an earlier revision of the same round. This is automatic for every earlier revision in the log. Add other form ids with `--prev-form-id`.
- Each saved form records its version, its round and, if it started from an earlier file, that file's name, sha256 and version (`carried`, `history`).

## Status after a round

`next-round` gives each item one status. The choice flags `confirms`, `defers` and `needs_reason` in `granted.json` decide what a choice means (see `card-schema.md`).

| Status | When |
|---|---|
| Confirmed | The reader selected an option (not Other), or a choice that `confirms` (by default: agree, defer), or marked an info card as noted. The item is not flagged and the reader was not unsure. |
| Open | No answer. |
| Open | Only a comment, with no option, assessment or "noted". |
| Open | A choice that does not confirm (by default: revise, disagree, need more information). |
| Open | Option "Other". |
| Open | Flagged for discussion. |
| Open | The reader was unsure. |
| Open | A confirmed item that the reader reopened in the confirmed table. |
| Open | An item that was open, but is missing from the form or from the returned file. |
| Open (your call) | A confirmed item with a comment that asks a question or changes the action. `next-round` marks every confirmed item with a comment, also comments on rows of the confirmed table. Read them all, and pass `--open <ID>` where needed. |

Defer counts as confirmed: the decision is "not now", and the confirmed table says "Deferred. Nothing is done now." If the reader defers an item that blocks other work, say so in the next round.

## The folder

```
review/
  granted.json                      the config
  <slug>_R1.0_<date>.html           working copy of an issued form (give this path to the reader)
  <slug>_R1.0_DRAFT.html            the newest draft preview (not logged, replaced by each --draft)
  REVIEW_LOG.md                     every issued and returned file with its sha256 (generated)
  registry.json                     the same, machine-readable (append-only)
  DECISIONS_after_R1.md / .json     the status of every item after round 1
  DECISIONS_after_R1.amend1.*       an amendment (next-round --amend)
  cards/R1.0/*.json                 the card set of each version (read-only after issue)
  cards/R1.0.sha256                 the checksums of that card set
  cards/R1.0.config.json            the config as it was when R1.0 was issued (for rebuild)
  cards/R2.0/_confirmed.json        the confirmed items shown in round 2
  figs/                             figures (embedded into the forms)
  R1/issued/                        frozen copies of the issued forms (read-only archives)
  R1/received/                      frozen copies of the returned forms (read-only)
  R1/answers/                       the answers extracted from each returned form
  R2/ …                             the same for round 2
```

The log holds paths relative to the review folder only, so you can share the folder. Relative links in the cards resolve from the working copy. The frozen copies are archives: their relative links can break.

## Rebuild an issued form byte for byte

The log keeps every input of each build: the card set, a frozen copy of the config, the build time (UTC), the note, the carried file, the form ids offered for restore and the checksums of the figures and the template. To check an issued form:

```
VM rebuild --dir review --version R1.1 --out /tmp/rebuild.html
```

It prints IDENTICAL or DIFFERENT, and for DIFFERENT the likely cause (a changed figure, template or tool version).

## How the form saves

- Until a save writes them into the form file, the browser keeps every change in its local storage, under the review id and the form id. A save into the form file deletes that copy (a downloaded copy does not), and the form removes copies older than 30 days whenever a Granted form opens. While the form offers newer browser answers, it does not store new changes, and a save keeps the offered copy until the reader chooses. In some browsers every local HTML file can read this storage. If the storage is blocked, the form says so.
- If the browser has newer answers than the file, the form shows the answers in the file and offers the others in a bar at the top. They come back only when the reader clicks **Use the browser answers**.
- The file carries a Content-Security-Policy: the page cannot make any request by itself (fetch, images, fonts, frames, form posts), and only the form's own script runs. Links that the reader clicks are not covered.
- **Save** writes the answers into the HTML file. Chrome and Edge write into the open file (the first save asks for the file). Other browsers and the VS Code preview download an updated copy: the form then says "Copy downloaded", not "Saved", because the open file did not change.
- A save removes any code that a viewer injected (for example the styles of the VS Code preview), so the saved file contains only the form and the answers. A script that a viewer injects is removed without a warning. A file that was changed in a way that can affect the answers (an extra data block, a second copy of a block tag, or an HTML comment before the form's script) shows a warning and none of its answers; Save then writes only what the reader answers there, in the builder's shape. `receive` and `answers` refuse a file without that shape, and `receive` also refuses a file whose visible text, options, links or styles differ from the issued form.
- The answers are in `<script type="application/json" id="review-state">`. `receive` and `answers` read them.
- **Seal & send** shows what is missing (no answer, reason missing, unsure, flagged, accepted without reading), saves, and copies a Markdown summary for the chat, in one click.

## Problems and fixes

| Problem | What to do |
|---|---|
| The reader saved into Downloads | `receive --find` finds it. Or give the file with `--file`. |
| The reader sends two saved files of one round | `receive` both (`--find` receives every new save). `next-round` uses the newer save. If the older one has an answer that the newer one lacks, it stops and lists both: ask the reader, then use `--answers`. |
| A save arrives after `next-round` closed its round | `receive` logs it, says that the round is closed and lists the changed answers. If the next round is not issued: `next-round --amend --answers <its answers .json>`. Otherwise ask the reader to reopen those items in the confirmed table of the current form. |
| The returned file was never saved | `receive` refuses it. Ask the reader to open the same file in the same browser, click **Use the browser answers**, then Save: the answers are in the browser storage for 30 days. |
| `receive` says the file does not have the shape that the form writes | Someone changed the file outside the form. Ask the reader to open it, answer the cards again (the form then shows none of the file's answers) and click Save, or send a fresh copy of the issued form. |
| `receive` says that what the reader saw differs from the issued form | Someone changed the visible text of the file. Do not use `--force`. Send the reader a fresh copy of the issued form. |
| `receive` says an answer is not a choice of its card | The file was changed by hand. Do not use `--force`. Ask the reader to answer in the issued form. |
| A tool says that the card files changed after the form was issued | Put the issued cards back (their checksums are in `cards/<version>.sha256`). Make changes in a new revision or round. |
| The reader answered the draft | `receive` refuses it. Ask the reader to answer the issued form. A draft keeps its answers apart, so nothing is lost in the issued form. |
| The reader answered an older revision | `receive` the file (it keeps its own version). If the reader has finished, run `next-round`: it uses the newest save of the round, from any revision. If they go on answering, issue the next revision: `revise`, then `build --carry <the received file>`. |
| You missed a comment after `next-round` ran | `next-round --amend --open ID`, as long as the next round is not issued. Otherwise ask in the round after it. |
| The form has a fault | Fix the cards (or the template) and issue a revision with `revise` and `build --carry`. Never edit an issued file. |

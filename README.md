<div align="center">

<img src="assets/logo.svg" width="96" alt="Vermilion seal">

# Vermilion · 朱批

**Review long AI reports like an emperor.**

Your AI writes the memorial. You hold the vermilion brush.

[![Claude Code skill](https://img.shields.io/badge/Claude%20Code-skill-c8361d)](#install)
[![Zero dependencies](https://img.shields.io/badge/dependencies-0-2e7d4f)](#how-it-works)
[![One HTML file](https://img.shields.io/badge/form-one%20HTML%20file-1f1a17)](#how-it-works)
[![MIT](https://img.shields.io/badge/license-MIT-6b625a)](LICENSE)

<a href="assets/vermilion-demo.mp4"><img src="assets/vermilion-demo.gif" width="860" alt="Vermilion in 25 seconds: a long AI report becomes one review form, the reviewer rules on each card, and round 2 holds only the open items"></a>

<sub>25 seconds. Click for the MP4.</sub>

</div>

---

## Why

AI agents now write long reports: audits, migration plans, design reviews, research summaries. A typical one has 10–50 items where **you** must decide, approve or correct something.

Reading the report is fine. **Giving feedback is the painful part.** You scroll, you quote, you type "for item 7, I think…" into the chat. You lose track of what you answered. The agent misreads which item you meant. And next week nobody can say what was decided, or on which version of the text.

In imperial China, ministers sent long memorials (奏章) to the throne. The emperor did not write essays back. He wrote a short decision on each one in vermilion ink: the 朱批. **Vermilion gives you that role.** The agent turns its report into one card per item, each with the evidence, the options and its recommendation. You rule on each card in a form. The agent reads your answers exactly, replies to your comments, and asks again only about what is still open.

**The result: better decisions and better feedback, with less effort.** You see every decision, and nothing is skipped. Each answer has a reason where it needs one. The agent gets structured answers instead of a paragraph to interpret.

> **Ultracode mode is highly recommended.** For a long report, let the agent run a multi-agent workflow: one writer per card, an independent verifier per card that checks it against the report, and a consistency pass. Writers that condense long reports drop hedges and change numbers. Verifiers catch this. See [references/ultracode.md](skills/vermilion/references/ultracode.md).

## How it works

```
 long report ──► cards (one item = one card) ──► R1.0 review form (one HTML file)
                                                    │  you answer, then Seal & send
                                                    ▼
       R2.0 form ◄── next-round ◄── receive (freeze + extract + log)
 (confirmed table + only the open cards, each with the AI's reply) ──► … until nothing is open
```

**For you, the reviewer**
- **One card per decision.** Each card has a one-line summary, the evidence (expand it when you doubt), figures, the options with their costs and risks, and one recommended option.
- **Rule fast.** <kbd>A</kbd> accepts the recommendation. <kbd>1</kbd>–<kbd>9</kbd> select an option. <kbd>J</kbd>/<kbd>K</kbd> move between cards. <kbd>N</kbd> jumps to the next unanswered card. <kbd>F</kbd> flags a card. <kbd>C</kbd> opens a comment. A vermilion seal (准 Approved, 改 Revise, 驳 Rejected, 缓 Deferred, 问 Question) stamps each card that you decide.
- **Say how sure you are.** Sure, leaning or unsure. If you are unsure, the agent explains more in the next round.
- **Reasons where they matter.** If you disagree, ask for a change or select "Other", the form asks you for one sentence of reason.
- **Seal & send.** One click lists what is unanswered, unsure or without a reason, saves your answers into the file and copies a short summary for the chat. Then you tell the agent "saved". It finds the file by itself if you saved it in the form's folder, Downloads or Desktop; otherwise give it the path.
- **Round 2 is short.** It shows a table of everything that is confirmed (you can reopen a row) and a card only for each open item. Each open card has your previous answer and the agent's reply to your comment.

**For the agent**
- A zero-dependency Python CLI (`vermilion.py`): `init`, `check`, `build`, `receive`, `next-round`, `revise`, `rebuild`, `answers`, `status`, `selftest`.
- `check` validates the cards, the links and the figures. It also gives a plain-language score (about 80% ASD-STE100, optional).
- `receive` refuses a file of another review, a draft preview, an unsaved file or a file whose cards differ from the issued form.
- `next-round` sorts every item into Confirmed or Open, with a reason. It writes `DECISIONS_after_R<n>.md` and drafts the next card set, with a `TODO(AI)` in each place where a reply is due. Nothing can drop out of the review without a decision.

**For the record**
- **The answers live inside the HTML file.** In Chrome and Edge, the first Save opens a Save dialog: choose the form file itself and confirm Replace, and later saves write into it directly. Other browsers and the VS Code preview download an updated copy.
- **Every issued and returned file is frozen** (a read-only copy) and logged with its sha256 in `REVIEW_LOG.md`. Each card carries a fingerprint of its text, and `receive` checks it against the issued form, so each answer traces to the card set that you were given.
- **Versions** are `R<round>.<revision>`. Option letters never change their meaning. A revision (R1.1) fixes wording and keeps your answers. A new round (R2.0) asks only what is open.
- **Private by design.** There is no server, no network call and no CDN. The form is one file on your disk.

## Install

**As a Claude Code plugin:**

```
/plugin marketplace add HaoranZhang122/vermilion-review
/plugin install vermilion@vermilion-review
```

**Or as a plain skill:** copy `skills/vermilion/` into `~/.claude/skills/` (for all projects) or into `.claude/skills/` (for one project).

You need Python 3.8 or later. To run the optional self-test, you also need Chrome, Chromium or Edge.

## Use it

Ask your agent:

```
Use vermilion to turn docs/migration-report.md into a review form. ultracode
```

The agent writes the cards, checks them, and gives you a path such as `review/acme-db-move_R1.0_2026-10-19.html`. Open it in your browser and answer. Then click **Seal & send** and tell the agent "saved". It reads your answers, replies to your comments, and sends you round 2 if something is still open.

**Try it now, with no agent:** clone the repo and open the files in [`skills/vermilion/examples/demo/forms/`](skills/vermilion/examples/demo/forms/) in your browser (GitHub itself shows HTML files as source code). They hold a fictional database migration review: round 1 empty, round 1 as a reviewer returned it, and round 2.

## Commands

| Command | What it does |
|---|---|
| `vermilion.py init --dir review --slug NAME --title TEXT` | Create the review folder and its config. |
| `vermilion.py check --dir review --version R1.0` | Validate the cards. Report the plain-language score. |
| `vermilion.py build --dir review --version R1.0 [--draft]` | Build a preview (`--draft`), or issue the form: a working copy, a frozen copy and a log entry. |
| `vermilion.py receive --dir review --find` | Receive every new save of this review in the review folder, ~/Downloads and ~/Desktop, oldest first (or give `--file SAVED.html`). Freeze each one, extract its answers (`.md` and `.json`) and log them. |
| `vermilion.py next-round --dir review [--dry-run] [--open ID ...]` | Classify the answers. Write the decisions file and draft the next round. |
| `vermilion.py revise --dir review --from R1.0 --to R1.1` | Copy a card set for a revision. |
| `vermilion.py build --dir review --version R1.1 --carry SAVED.html` | Issue a revision that keeps the answers of the round. |
| `vermilion.py rebuild --dir review --version R1.0 --out FILE` | Rebuild an issued form from the log and compare it byte for byte. |
| `vermilion.py answers SAVED.html` | Print the answers of any form. Logs nothing. |
| `vermilion.py status --dir review` | Show the log. |
| `vermilion.py selftest FORM.html` | Run the form's built-in self-test in a headless browser. |

The details are in [SKILL.md](skills/vermilion/SKILL.md) and [references/](skills/vermilion/references/): the card schema, how to write cards, plain-language rules, rounds and versions, and the Ultracode recipe.

## FAQ

**Which browser?** In Chrome or Edge, the first Save asks where to save: choose the form file itself. Firefox, Safari and the VS Code preview work too: Save downloads an updated copy. Give that copy to the agent.

**I closed the tab without saving.** Open the same file in the same browser. The form offers the answers that the browser kept.

**Can I trust what the agent says it decided?** The decisions file lists every item with its status and the reason. Each answer traces to a frozen copy of the file that you saved, with its sha256. Your comments go to the agent as your words. The agent reads every comment and replies on each open card in the next round.

**Does it work with other agents (Codex, Cursor, Gemini CLI)?** The CLI is plain Python and SKILL.md is plain Markdown. Any agent that can read a file and run a command can follow it.

**Is my data sent anywhere?** No. The form makes no network requests. Figures are embedded in the file.

## Author

Haoran Jack Zhang · [@HaoranZhang122](https://github.com/HaoranZhang122)

## License

MIT. See [LICENSE](LICENSE).

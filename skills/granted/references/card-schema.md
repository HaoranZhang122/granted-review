# Card schema (schema 1)

A **memorial** is one review form. It has a config file, `granted.json`, and one JSON file for each card.

```
review/
  granted.json            the config of the form (one per review)
  cards/R1.0/D1.json      one file per card; the file name is the card id
  cards/R1.0/F2.json
  figs/cutover.svg        figures that the cards use (embedded into the form)
```

Files in a card folder whose name starts with `_` are not cards. `next-round` writes `_confirmed.json` there.

## The config: `granted.json`

| Key | Required | Meaning |
|---|---|---|
| `schema` | yes | Always `1`. |
| `slug` | yes | A short name, `[a-z0-9-]+`. It starts the form id and the file names. |
| `review_id` | no | 8 random hex characters that `init` writes. The browser keeps the answers under it, so two reviews with the same slug never mix. |
| `title` | yes | The heading of the form. |
| `short_title` | no | The title in the top bar. Keep it short (≤ 28 characters). |
| `intro_html` | no | The text at the top of the form. |
| `sources` | no | A list of `{label, href}`: the full report, the plan, the evidence folder. A relative `href` resolves from the review folder (where the working copy of the form is). |
| `link_base` | no | A prefix for **every** relative link: in the cards, in `[src: …]` citations and in `sources`. Use it when all the linked files are in another folder, for example `../`. It must be empty, a relative path or an http(s) URL. |
| `figs_dir` | no | The folder of the figures, relative to the review folder. Default `figs`. A figure must be inside this folder. |
| `priorities` | no | The priority scale, highest first. Default `Critical, High, Medium, Low`. |
| `labels` | no | A list of `{name, meaning}`: the evidence labels that the cards use. The form shows them as a legend. |
| `sections` | yes | A list of `{key, title, intro, kind}`. `kind` is the default kind of the cards in the section. |
| `choices` | no | The assessment choices of a `review` card: `{value, label, seal, seal_en, tone, confirms, accept, defers, needs_reason}`. See below. Never change a `value` after you issue a form. |
| `seal_style` | no | `cjk` (default: the seal shows a Chinese character and an English word) or `latin` (English only). |

### Choices

Each choice says what it means. The form and `next-round` use these flags, not the values:

| Flag | Meaning | Default choices that have it |
|---|---|---|
| `confirms` | This answer confirms the item (it is not asked again). | agree, defer |
| `accept` | The **Accept** button and the key A select this choice on a review card without options. At most one choice. | agree |
| `defers` | The decision is "not now". The confirmed table says "Deferred. Nothing is done now." | defer |
| `needs_reason` | The form asks for a comment, and `Seal & send` lists the card if the comment is empty. | modify, disagree, info |
| `tone` | The colour of the seal: empty (vermilion, for approval only), `amber`, `ink`, `gray`, `blue`, `green` or `purple`. A choice without a tone gets vermilion only if it confirms; otherwise ink. | modify amber, disagree ink, defer gray, info blue |

If you use the default values (`agree`, `modify`, `disagree`, `defer`, `info`), missing flags come from the defaults. The values `other` and `noted` are reserved.

### What must not change after the first issue

`check` (and so `build`) refuses these changes, because the earlier forms, answers and decisions depend on them: `review_id`, `slug`, a removed choice value, a choice whose `confirms`, `accept` or `defers` flag changed, and a removed, renamed or re-kinded section. Titles, labels, `intro_html`, `sources` and `labels` can change. Each issued version also keeps a frozen copy of the config (`cards/<version>.config.json`), and the tool reads the cards of that version with it.

## A card

```json
{
  "id": "D1",
  "section": "decisions",
  "order": 1,
  "kind": "decision",
  "title": "Select the cutover method for the database move",
  "summary": "Three cutover methods are possible. The plan recommends logical replication, because it gives the shortest downtime and a fast rollback.",
  "priority": "High",
  "tag": "Cutover",
  "badges": [{"text": "OWNER-ONLY", "tone": "red", "tip": "Only you can approve the downtime."}],
  "blocks": [
    {"title": "The problem", "html": "<p>…</p>"},
    {"title": "What we found", "html": "<ul><li>… [src: notes/load_test.md]</li></ul>"}
  ],
  "figures": [{"file": "cutover.svg", "caption": "The three methods on one time line."}],
  "options": [
    {"key": "a", "label": "Big-bang copy in a 4-hour window", "detail_html": "<p>…</p>"},
    {"key": "b", "label": "Logical replication, then a 5-minute switch", "detail_html": "<p>…</p>", "recommended": true},
    {"key": "c", "label": "Dual writes from the application", "detail_html": "<p>…</p>"}
  ],
  "recommendation_html": "<p>Option b, because …</p>",
  "facts": [{"label": "Blocks", "value": "D3, F4"}, {"label": "Decide by", "value": "Before the load test"}],
  "related": ["D3", "F4"],
  "sources": [{"label": "Load test notes", "href": "notes/load_test.md"}]
}
```

| Field | Required | Rules |
|---|---|---|
| `id` | yes | `[A-Za-z0-9][A-Za-z0-9._-]*`, at most 64 characters. It must be the same as the file name without `.json`. Never reuse an id for a different item. Some words are reserved (for example `constructor`, `toString`). |
| `section` | yes | A `key` from `sections`. |
| `order` | no | A number. The sort order inside the section. |
| `kind` | no | `decision`, `review` or `info`. The default comes from the section. |
| `title` | yes | A statement of the item, not a question. ≤ 90 characters. |
| `summary` | yes | One or two sentences: the problem, then the proposed action. The reader sees this line when the card is closed. |
| `priority` | no | A value from `priorities`. |
| `tag` | no | A short label, for example the item number in the source report. |
| `badges` | no | `{text, tone, tip}`. `tone` is `red`, `amber`, `blue`, `green`, `purple` or `gray`. |
| `blocks` | no | The body of the card: `{title, html}` in reading order. Usual titles: The problem · What we found · Why it matters · Proposed action. |
| `figures` | no | `{file, caption}`. The file is in `figs_dir`. PNG, JPEG, SVG, WebP or GIF. |
| `tables_html` | no | Tables and examples, shown after the figures. |
| `options` | `decision`: yes | `{key, label, detail_html, recommended}`. A key is 1–8 letters or digits. A `decision` card has exactly one `recommended` option. A `review` card can also have options. The form adds an "Other" option. |
| `recommendation_html` | no | Why the recommended option is the best one. |
| `action` | no | Plain text: what will happen if the reader agrees. The decisions file uses it in "What will happen". Without it, the tool takes the last block whose title starts with "Proposed action", "Action", "Fix", "Next steps" or "Recommendation". |
| `facts` | no | `{label, value}` rows, for example Blocks, Owner, Decide by, Cost. |
| `related` | no | Ids of other cards. They become links. |
| `sources` | no | Strings or `{label, href}`. Shown as links at the end of the card. |

**Kinds:**
- `decision`: the reader selects one option (or "Other" and a comment).
- `review`: the reader selects one choice from `choices` (agree, revise, disagree, defer, need more information). Options are optional.
- `info`: the reader only confirms that they read it ("Noted") or writes a comment.

**HTML.** The builder keeps only these tags: `p ul ol li b strong i em code pre table thead tbody tr th td br span sub sup blockquote h4 small a`. It removes every other tag and every attribute except `colspan`, `rowspan` and a safe `href` (http, https, mailto, a relative path or `#id`).

**Source citations.** Write `[src: path/file.md]` or `[src: a.md; b.csv]` after a claim. The form makes each path a link, with `link_base` before it. A URL also works: `[src: https://example.com/page]`.

## Fields for round 2 and later

`next-round` writes these fields. You (the AI) complete them before you build the next round.

| Field | Written by | Meaning |
|---|---|---|
| `previous` | `next-round` | The reader's answer in the last round: `{version, option, option_label, choice, choice_label, comment, flag, confidence}`. The form shows it in a box at the top of the card. |
| `status_reason` | `next-round` | Why the card is open again, for example "You selected Other" or "You flagged it". |
| `reply_html` | you | Your reply to the reader's comment. Answer each question. Say what you changed. `check` fails while it still contains `TODO(AI)`. |
| `disposition` | you | What you did with the answer: `changed`, `answered`, `explained` or `declined`. The form shows it next to your reply. `check` warns while it is empty. |
| `changes_html` | you | What changed in the card since the last round, if anything. |

**Option keys never change.** If an option changes its meaning, give it a new key (for example `d`) and remove the old option. An option that the reader proposed in a comment becomes a new option with a new key.

`_confirmed.json` holds the confirmed items. The form shows them in one table at the top, and the reader can reopen a row. It is only the display: the decisions files decide what is confirmed, so `check` refuses a `_confirmed.json` whose items or decisions differ from them. You may edit the `action` texts.

```json
{"round": 2, "items": [
  {"id": "F1", "title": "…", "decision": "Agree: do the proposed action", "action": "…", "since": "R1.0", "comment": ""}
]}
```

## The answers inside a saved form

The form saves the answers in `<script type="application/json" id="review-state">`:

```json
{
  "formId": "my-review-R1.0-2026-10-18", "formVersion": "R1.0", "round": "1",
  "savedAt": "…", "updatedAt": "…",
  "meta": {"reviewer": "…", "date": "…", "general": "…"},
  "cards": {"D1": {"opt": "b", "comment": "…", "flag": false, "conf": "sure", "how": "accept", "updatedAt": "…"},
            "F2": {"resp": "modify", "comment": "…", "conf": "unsure"}},
  "reopen": {"F1": {"reopen": true, "comment": "…"}},
  "opened": {"D1": true},
  "carried": {"fromFile": "…", "fromSha256": "…", "fromVersion": "R1.0"},
  "history": [ … ]
}
```

`opt` is an option key or `other`. `resp` is a choice value. `conf` is `sure`, `leaning` or `unsure`. `how` is `accept` when the reader used the Accept button. `opened` lists the cards whose details the reader opened, so `receive` can report "accepted without opening the details".

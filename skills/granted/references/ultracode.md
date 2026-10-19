# Ultracode: write and verify the cards with many agents

For a short report, one agent can write the cards. For a long report (more than about 15 items), or when a wrong card is expensive, split the work: independent writers, independent verifiers that try to refute each card, and one consistency pass. In Claude Code, turn on **Ultracode** (put the word `ultracode` in your prompt, or turn it on for the session) so that the agent orchestrates this with the Workflow tool. Ultracode needs dynamic workflows (see `/config`), and some setups do not offer it. Without it, and in Codex, do the same phases one after another, as SKILL.md says.

## The recipe

| Phase | Agents | Job |
|---|---|---|
| 1. Extract | 1 | List every item of the report that needs the reader: id, kind, section, title, the lines of the report it comes from. |
| 2. Completeness | 1 | Read the report again and find items that the list misses or merges by mistake. |
| 3. Write | 1 per card | Write `cards/R1.0/<id>.json` from the report lines only, by `writing-cards.md` (and `plain-language.md` if asked). |
| 4. Verify | 1 per card (2 for decision cards) | Compare the card with the report. Try to refute each claim, number and option. Fix errors in the card. Report what was fixed. |
| 5. Consistency | 1 | Read all cards together: the same term for the same thing, correct `related` ids, no duplicate items, sensible priorities. Run `check`. |
| 6. Build and test | 1 | `build --draft`, `selftest`, open the draft in a browser (Playwright if available), answer one card of each kind, save, and run `answers` on the saved file. |

For round 2 and later, use the same shape on the open cards: one agent per card writes `reply_html` from the reader's comment, and a verifier checks that the reply answers every question and adds no unsupported claim.

**Why a separate verifier:** a writer that condenses a long report tends to drop hedges, change numbers and merge options. A fresh agent that checks only one card against the source finds these errors.

## A workflow template

Pass `args` as `{ "report": "docs/report.md", "review": "review", "version": "R1.0", "skill": "<folder of SKILL.md>" }`.

```js
export const meta = {
  name: 'granted-cards',
  description: 'Write and verify Granted review cards from a long report',
  phases: [{ title: 'Extract' }, { title: 'Write' }, { title: 'Verify' }, { title: 'Consistency' }],
}
const ITEMS = { type: 'object', required: ['items'], properties: { items: { type: 'array', items: {
  type: 'object', required: ['id', 'section', 'kind', 'title', 'source'],
  properties: { id: { type: 'string' }, section: { type: 'string' }, kind: { type: 'string', enum: ['decision', 'review', 'info'] },
                title: { type: 'string' }, source: { type: 'string', description: 'where in the report: headings or line numbers' },
                replaces: { type: 'array', items: { type: 'string' }, description: 'ids of listed items that this item splits or replaces' } } } } } }
const VERDICT = { type: 'object', required: ['id', 'fixed', 'remaining'], properties: {
  id: { type: 'string' }, fixed: { type: 'array', items: { type: 'string' } }, remaining: { type: 'array', items: { type: 'string' } } } }
const REF = `${args.skill}/references`
const RULES = `Rules: ${REF}/card-schema.md, ${REF}/writing-cards.md and ${REF}/plain-language.md. Never invent a number or a claim.`
const card = id => `${args.review}/cards/${args.version}/${id}.json`

phase('Extract')
const first = await agent(`Read ${args.report} and ${args.review}/granted.json. List every item that needs the reader's decision or review. ${RULES}`, { schema: ITEMS })
const missed = await agent(`Here is a list of review items from ${args.report}: ${JSON.stringify(first.items)}. Read the report again. ` +
  `Return ONLY items that are missing, or new items for items that the list merged by mistake (set 'replaces' to the merged ids).`, { schema: ITEMS })
const replaced = new Set(missed.items.flatMap(m => m.replaces || []))
// keep a new item if it is new, or if it replaces an item (a split part can keep the old id)
const items = [...first.items.filter(f => !replaced.has(f.id)), ...missed.items.filter(m => replaced.has(m.id) || !first.items.some(f => f.id === m.id))]
log(`${items.length} items: ${missed.items.length} from the completeness pass, ${replaced.size} replaced`)

const verify = (it, lens) => agent(`Verify ${card(it.id)} against ${args.report} (${it.source}). Lens: ${lens}. ` +
  `Try to refute every claim, number, option and recommendation. Fix each error in the file. Do not add claims that the report does not support. ${RULES}`,
  { label: `verify:${it.id}:${lens}`.slice(0, 60), phase: 'Verify', schema: VERDICT })
const done = await pipeline(items,
  it => agent(`Write the card ${it.id} (${it.kind}, section ${it.section}) into ${card(it.id)}. Source: ${it.source} of ${args.report}. ${RULES}`,
              { label: `write:${it.id}`, phase: 'Write' }),
  (_, it) => verify(it, 'facts: numbers, names, hedges and evidence labels'),
  (v, it) => it.kind === 'decision' ? verify(it, 'options: overlap, honest costs and risks, one sound recommendation').then(v2 => [v, v2]) : [v])

phase('Consistency')
const report = await agent(`Read every card in ${args.review}/cards/${args.version}/. Make terms consistent, fix related ids, merge duplicates, check priorities. ` +
  `Then run: python3 ${args.skill}/scripts/granted.py check --dir ${args.review} --version ${args.version} and fix every error. Report what you changed.`)
return { items: items.length, verify: done.filter(Boolean).flat(), report }
```

After the workflow, read its result, build a draft, open it, and only then issue the form.

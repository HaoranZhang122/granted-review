# Plain language for the cards (about 80% ASD-STE100)

ASD-STE100 (Simplified Technical English) is a controlled language for technical documents. It makes text easier to read for people who do not speak English as a first language, and it removes ambiguity. These rules apply its writing rules to review cards. They do not replace the official specification, and they do not check words against the official STE dictionary.

**"80%"** means: at least 80% of the sentences on each card pass every automatic check of `granted.py check`. The other sentences can break a rule only when strict STE would lose precision. Run `check --plain 80` to enforce it.

## A. Hard rules

1. **Change the words, not the facts.** In a rewrite, every number, unit, sign, identifier, code span, citation and evidence label stays the same. Do not add or remove a fact. Do not make a claim stronger or weaker. Keep each hedge ("about", "at least", "not tested").
2. **No semicolons in prose.** Use two sentences or a list.
3. **No sentence longer than 30 words**, not counting citations.
4. **Options keep their meaning.** A rewritten option describes the same choice, with the same costs and effects.

## B. Writing rules

1. **Sentence length.** Up to 20 words in an instruction, up to 25 words in a description. A number, a code span or an identifier counts as one word.
2. **One topic per sentence.** Put a series of items, steps or conditions into a list.
3. **Paragraphs.** Up to six sentences. The most important information comes first.
4. **Active voice.** Say who does the action: "The script removes…", "The plan recommends…", "You must decide…". Use the passive only when the agent is unknown or does not matter.
5. **Simple verb forms.** Simple present, simple past, simple future ("will"), imperative, infinitive. Avoid "-ing" verb forms: write "when the job runs", not "running the job". Technical names that end in -ing are permitted (logging, caching, testing).
6. **Modal verbs.** "must" for a requirement, "can" for a possibility, "will" for the future. Avoid should, would, could, might and may.
7. **Conditions first.** "If the switch fails, roll back", not "Roll back if the switch fails".
8. **No contractions, idioms or phrasal verbs.** "find", not "find out". "do", not "carry out".
9. **Keep the small words.** Do not drop "the", "a" or "that" to save space.
10. **Noun clusters.** Up to three nouns in a row, technical names excepted.
11. **One word, one meaning.** Pick one term for each thing and use it on every card. List the terms in the config intro if the review is long.
12. **Abbreviations.** Write the full name the first time on each card.
13. **Explain jargon briefly** where the reader cannot know it. The explanation must not add a new claim.

## C. Plain words

| Avoid | Use |
|---|---|
| utilize | use |
| approximately, ~ (in prose) | about |
| however | but |
| therefore, hence, consequently | thus, so, as a result |
| additionally, furthermore, moreover | also |
| ensure | make sure |
| demonstrate, indicate | show |
| obtain | get |
| commence, initiate | start |
| terminate | stop |
| sufficient | enough |
| numerous | many |
| prior to | before |
| subsequent to | after |
| in order to | to |
| due to | because of |
| facilitate | help |
| regarding, with respect to | about |
| via | through, with |
| e.g., i.e., etc. | for example, that is, and other |
| respectively | write the pairs out |

## D. Example

**Before (43 words):**
> Due to the fact that the restore procedure has never been executed, it should be noted that the recovery time objective cannot be guaranteed; therefore, it is recommended that a restore test be performed monthly prior to the migration.

**After:**
> Nobody has tested the restore. Thus we do not know how long a recovery takes. The plan recommends a restore test each month. The first test must be before the migration.

## E. What the checker flags

`granted.py check` flags a sentence with: `LEN` (> 25 words), `SEMI` (semicolon), `MODAL` (should, would, could, might, may, shall), `PASSIVE` (a form of "be" + a past participle), `ING` (an -ing word that is not in the list of technical names), `WORD` (a word from table C) and `CONTR` (a contraction). The checks are heuristics: a flagged sentence can be correct, and a sentence without flags can still be unclear. Read the flagged sentences, and fix the ones that are really hard to read.

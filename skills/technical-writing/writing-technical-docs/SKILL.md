---
name: writing-technical-docs
description: "Trigger: README, CONTRIBUTING, guide, runbook, wiki or Notion page, architecture doc, doc tone or voice review. Write docs for people in plain, reasoned prose."
license: Apache-2.0
metadata:
  author: "AlessandroRLM"
  version: "1.0"
---

## Activation Contract

Load when writing or reviewing documentation a person reads: README, CONTRIBUTING, guides, runbooks, architecture and wiki pages. It sets the voice, not the structure. Not for files an agent reads as instructions (AGENTS.md, skills, prompts): those stay terse and imperative.

## Hard Rules

- State each rule together with its reason or its consequence, joined in one sentence or in the next one with a connector ("because", "so", "therefore"). A rule alone reads as an order; the reason is what the reader keeps.
- Write connected sentences. Never stack short fragments ("Do not skip them." "Run once per clone."). If three sentences in a row have fewer than eight words, join them.
- Name the tool or what it does (the hook, CI, the tests, the linter). Never use team shorthand or an internal label as if it were common vocabulary ("gate", "the guard"). Do not translate the technical vocabulary the reader already uses: terms of the platform and its interface (re-run, summary, dispatch, flag, dry-run) and the names of tools, commands and objects stay in their original language.
- Say what concretely happens when the rule is not followed, instead of only forbidding it.
- State costs and limits plainly, without softening or selling them.
- Use the voice the project already uses. In Spanish default to impersonal ("Los hooks se instalan una vez por clon"), neutral and professional, without regional forms.
- Bold only opens a paragraph with the rule that the paragraph explains. Never bold for emphasis inside a sentence.
- A table holds parallel data. Do not turn an explanation into a table or a checklist. Add a sentence before a table only when it tells the reader something the heading and the columns do not; never a sentence that only announces the table ("The table lists...").
- Do not write maxims or slogans ("X is the source of truth"), dramatic contrasts ("it is not X, it is Y"), rhetorical questions, announcements of what comes next, or em dashes.
- A contributing guide is short: a welcome line, the steps to send a change, and links to where the system is explained. It does not explain how the system works.

## Decision Gates

| Document | Voice |
|---|---|
| Architecture or system page | Impersonal, present tense. Each paragraph: the rule, then why, then what follows from it |
| Contributing guide | Impersonal. Steps in the order they are done, each with what it prevents |
| Runbook | Numbered steps with the exact command, then how to read the result and what to do for each outcome |
| README of a component | What it is, how it is used, and the limits a user will hit |

## Execution Steps

1. Read two or three existing pages of the project and match their person, tense and vocabulary.
2. Draft each paragraph as rule, reason, consequence.
3. Replace every internal label with the name of the tool or a plain description.
4. Read the draft aloud. Rewrite any sentence a colleague would not say when explaining it to a new teammate.
5. Check the draft against `references/examples.md`.

## Output Contract

Writing: the final text. Reviewing: a list of `passage | rule broken | proposed text`; say "no findings" when there are none.

## References

- `references/examples.md`: before and after pairs.
- `references/sources.md`: sources and how their material is used.

---
name: writing-data-metadata
description: "Trigger: table description, column description, data dictionary, glossary term, catalog entry, metadata review. Write metadata a data consumer can use alone."
license: Apache-2.0
metadata:
  author: "AlessandroRLM"
  version: "1.0"
---

## Activation Contract

Load when writing or reviewing what a data consumer reads to find and understand data: descriptions of tables, views, columns and datasets, glossary terms and catalog entries, in YAML, SQL `COMMENT` or a catalog. Not for code comments, docstrings or commits: a developer reads those.

## Hard Rules

- Write for a consumer with no access to the code, the issue tracker or the team. Every description stands alone.
- A definition MUST be singular, say what the item is (not only what it is not), be a full phrase or sentence rather than a label, use only abbreviations the reader already knows, and name other items instead of defining them inside.
- A definition SHOULD give the core meaning, be precise and concise, never define a term with itself, and reuse the wording and structure of related definitions.
- Never invent facts about the data. When the grain, source, allowed values or unit are unknown, leave a visible placeholder (`<allowed values>`) and ask.
- Never include work state or implementation: ticket or decision-record ids, review codes, dated measurements, "still" or "until X ships", function, macro, model or pipeline names, how the code computes the value, citations of books or standards. Move them to code comments, test descriptions or decision records.
- Define a business concept once, in the glossary when one exists. A column description says what the column holds in its table and names the concept.
- Keep each description within its platform's limit (BigQuery column: 1,024 characters).

## Decision Gates

| Item | Write |
|---|---|
| Table or view | The grain first ("One row per ..."), what a row represents, and what it excludes that a consumer could expect |
| Column | One definition sentence; after it, only consumer notes: what an empty value means, unit or currency, allowed values, how to filter or join |
| Glossary term | The concept's definition, independent of any table; what it includes and excludes; the synonyms consumers search for; its owner where the catalog has a field for it |
| Sensitivity or classification | Nothing here: declare it with the platform's classification (policy tag, aspect, label) |
| Rationale, history, computation | Nothing here: code comment, test description or decision record |

## Execution Steps

1. Classify each item with the Decision Gates.
2. Write the definition sentence first and check it against `references/definition-rules.md`.
3. Add consumer notes only of the kinds the gate allows.
4. Strip every never-include item and say where it moved.
5. Reread as the consumer: could they use the data correctly with this text alone?

## Output Contract

Writing: the final text per item, ready to paste. Reviewing: a list of `item | rule broken | proposed text`; say "no findings" when there are none.

## References

- `references/definition-rules.md`: definition rules with their source and modality.
- `references/metadata-fundamentals.md`: governance and security, technical and business metadata, glossary and description.
- `references/examples.md`: before and after pairs.
- `references/sources.md`: sources and how their material is used.

# Definition rules

The rules for the definition sentence of any metadata item, in our own words. Each rule names its source and keeps its modality: a requirement is a must, a recommendation is a should. See `sources.md` for the full references.

## Requirements (ISO/IEC 11179-4:2004, clause 4.1)

| Rule | In practice |
|---|---|
| Use the singular | "A shipment that ...", not "Shipments that ..." |
| Say what the thing is, not only what it is not | "Excludes returns" alone defines nothing |
| Write a full phrase or sentence that describes the item | Not a bare label or the name reworded |
| Use only abbreviations a consumer already knows | Spell out internal acronyms |
| Do not define other items inside the definition | Name the other item; its definition lives elsewhere |

## Recommendations (ISO/IEC 11179-4:2004, clause 4.2)

| Rule | In practice |
|---|---|
| Give the core meaning | What makes this item this item |
| Be precise and leave one reading only | No term with two senses in the same context |
| Be concise | No filler, no repetition of the table's description |
| Make it understandable on its own | No need to read other definitions, code or tickets |
| Leave out why it exists, how a process uses it and how it is produced | Those go to code comments, tests or decision records |
| Do not define a term with itself | Nor with a term that is defined by it |
| Word related definitions the same way | Sibling columns read alike |
| Fit the kind of item | A table, a column and a glossary term need different content |

The standard also expects the terms a definition uses to be defined in an associated glossary.

## Plain language (The Ohio State University)

- Write in plain language, without jargon.
- Leave examples, procedures, usage and rationale out of the definition. This makes the ISO recommendation about rationale and usage a must.
- Do not use the term inside its own definition.
- Define a metric with two parts: how it is calculated and what it means in plain words.

## Notes after the definition

The rules above govern the definition. A catalog with a single description field also has to tell the consumer how to use the item, so a description is the definition sentence followed, when needed, by consumer notes of a few kinds only: what an empty value means, the unit or currency, the allowed values, and how to filter or join. A note that explains why the item exists or how it is computed is not a consumer note.

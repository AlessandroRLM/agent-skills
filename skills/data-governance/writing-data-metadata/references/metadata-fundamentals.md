# Metadata fundamentals

Full references and how their material is used: `sources.md`.

## Governance and security

They are different concepts that complement each other (Practical Lakehouse Architecture, O'Reilly 2024, chapter 6, "Data (and AI) Governance and Security in Lakehouse Architecture"):

- Data governance is "an umbrella term that comprises various standards, rules, and policies to ensure that all data management processes follow formal guidelines". It "provides guidance for both data and metadata ingestion, integration, storage, ownership, consumption, and sharing".
- Data security "deals with data protection—wherever it resides and whenever users access it", so that "only the right people have the right levels of access to the right data".

Writing metadata is governance. Deciding who may read a column, and declaring its sensitivity for that purpose, is security.

The same chapter lists ten processes: metadata management, compliance and regulations, data and model quality, lineage, data and AI asset sharing, data ownership, auditing and monitoring, access management, data protection, and handling sensitive data.

## Technical and business metadata

Chapter 4 of the same book ("Data Catalogs") separates technical metadata from business metadata. Only its headings were read:

- Technical metadata describes structure: names, types, partitions, keys.
- Business metadata gives the meaning that the technical names do not carry.

A catalog serves search and discovery, classification, governance and security, and lineage. The description of an item is business metadata: it is what a consumer reads to decide whether the item is the one they need.

## Glossary and description

- A glossary term defines a business concept once, independent of any table. ISO/IEC 11179-4 expects definitions to rely on an associated glossary for the terms they use.
- A term holds more than its definition: what the concept includes and excludes (the edge cases people ask about), the synonyms consumers search for, and who owns the definition. Synonyms serve search and discovery; the owner is ownership metadata, and catalogs usually keep it in its own field.
- A table or column description says what the item holds and names the concepts it uses.
- Catalogs connect the two. Google Cloud Knowledge Catalog links a glossary term to a column with a `definition` link, and keeps other metadata as aspects on the table, a column or a link.
- Without a glossary, the column description carries the definition sentence, and the same concept is worded the same way everywhere it appears.

## The consumer

The reader of metadata is whoever looks for data: an analyst, another team, an agent. They read it where the data lives (the warehouse, the catalog), not in the repository, and most of them cannot open the issue tracker. dbt states the purpose: good documentation "will help downstream consumers discover and understand the datasets you curate for them".

## Where implementation detail goes

| Detail | Home |
|---|---|
| How the value is computed, which function or model builds it | Code comment next to the logic |
| The rule a test proves | The test's own description |
| Why a design was chosen | Decision record |
| Who asked for it, when, in which ticket | The issue tracker and the commit |

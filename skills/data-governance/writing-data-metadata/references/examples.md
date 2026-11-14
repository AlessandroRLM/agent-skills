# Before and after

Each pair names the rule the first version breaks. Names and ids are invented.

## Implementation detail in an id column

- Before: "The order id, derived in staging from the source order key (macro build_order_key)."
- Breaks: never-include (model layer, macro name, how the code computes it).
- After: "Identifier of the order. Unique in this table."

## Ticket reference

- Before: "Debit minus credit, in company currency (DATA-142)."
- Breaks: never-include (ticket id).
- After: "Debit minus credit of the journal item, in the company's currency."

## Citation

- Before: "Load date of the record's tie to its key (see the modeling book, section 4.3)."
- Breaks: never-include (citation); stand-alone recommendation (it needs the reader to know the method).
- After: "When the record was first loaded under its current identifier."

## Circular definition

- Before: "order_status: The status of the order."
- Breaks: no-self-definition recommendation; plain-language rule.
- After: "Stage the order is at in the source system: draft, confirmed, done or cancelled."

## Definition by negation

- Before: "Shipments that are not receipts or internal transfers."
- Breaks: say-what-it-is requirement.
- After: "One row per delivery of goods to a customer. Receipts and transfers between warehouses are not included."

## Definition embedded in another

- Before: "Net sales, which is gross sales minus discounts and returns, for the line."
- Breaks: do-not-define-other-items requirement.
- After: "Net sales of the line." With "net sales" defined once in the glossary.

## Consumer note mixed with rationale

- Before: "Promised date. Empty because the source only sets it when the order has a commitment date, which we decided not to infer."
- Breaks: rationale recommendation.
- After: "Date promised to the customer for the delivery. Empty when the order has no promised date."

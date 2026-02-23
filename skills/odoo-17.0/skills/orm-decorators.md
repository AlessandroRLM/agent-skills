---
name: odoo-orm-decorators
description: >
  Focused guide to the most commonly misused Odoo 17.0 @api decorators.
  Covers correct usage and critical gotchas for: @api.depends,
  @api.depends_context, @api.constrains, @api.onchange, @api.model,
  and @api.model_create_multi. These six are responsible for the vast
  majority of compute, validation, and create bugs in custom modules.
  Trigger: When writing compute methods, field validation, onchange handlers,
  default_get overrides, or create() overrides in Odoo 17.0 Python code.
license: Apache-2.0
metadata:
  author: AlessandroRLM
  version: "1.0"
---

## When to Use

Use this skill when:

- A computed field is not updating when its source fields change
- A `@api.constrains` method is not blocking invalid saves
- An `@api.onchange` is not firing or behaving inconsistently
- You are overriding `create()` and unsure which decorator to use
- You need to understand the exact difference between onchange and constrains

---

## The Six Decorators — Quick Reference

| Decorator                 | When it runs                              | Receives                | Primary use                 |
| ------------------------- | ----------------------------------------- | ----------------------- | --------------------------- |
| `@api.depends`            | On dependency field write                 | recordset (multi)       | Compute field values        |
| `@api.depends_context`    | On context key change                     | recordset (multi)       | Context-sensitive compute   |
| `@api.constrains`         | After every create/write on listed fields | recordset (multi)       | Validate and block save     |
| `@api.onchange`           | User changes field in form UI             | virtual record (single) | Update related fields in UI |
| `@api.model`              | Always (class-level, no record)           | empty recordset         | default_get, class helpers  |
| `@api.model_create_multi` | On create() call                          | empty recordset         | Override create() in 17.0   |

---

## Critical Patterns

### `@api.depends` — The Most Misused

**Rule:** List EVERY field whose value affects the result. Missing a dependency = stale computed values with no error.

```python
# ✅ CORRECT — all three fields that affect the result are listed
@api.depends('price_unit', 'quantity', 'discount')
def _compute_price_subtotal(self):
    for line in self:
        price = line.price_unit * (1 - line.discount / 100)
        line.price_subtotal = price * line.quantity

# ✅ CORRECT — traverse relational fields with dot-notation
@api.depends('partner_id.country_id.name')
def _compute_partner_country(self):
    for rec in self:
        rec.partner_country = rec.partner_id.country_id.name or ''

# ✅ CORRECT — trigger on One2many children changes
@api.depends('line_ids.price_subtotal')
def _compute_amount_total(self):
    for order in self:
        order.amount_total = sum(order.line_ids.mapped('price_subtotal'))
```

**Gotchas:**

```python
# ❌ WRONG — missing dependency → stale value when quantity changes
@api.depends('price_unit')          # forgot 'quantity'!
def _compute_subtotal(self):
    for rec in self:
        rec.subtotal = rec.price_unit * rec.quantity  # qty changes → no recompute

# ❌ WRONG — writing to a NON-computed field inside depends
@api.depends('state')
def _compute_label(self):
    for rec in self:
        rec.computed_field = rec.state   # ✅ fine
        rec.other_regular_field = 'x'    # ❌ NEVER — side effects, potential infinite loops

# ❌ WRONG — not assigning in every branch (required for store=True fields)
@api.depends('partner_id')
def _compute_label(self):
    for rec in self:
        if rec.partner_id:
            rec.label = rec.partner_id.name
        # ❌ missing else → stale value when partner is cleared

# ✅ CORRECT — always assign in all code paths
@api.depends('partner_id')
def _compute_label(self):
    for rec in self:
        rec.label = rec.partner_id.name if rec.partner_id else ''
```

---

### `@api.depends_context` — Context-Sensitive Compute

**Rule:** Use when the result depends on runtime environment (user, company, language), not stored field values. Fields using this **cannot** be `store=True`.

```python
# ✅ CORRECT — price differs by company currency
@api.depends('price_unit')
@api.depends_context('company')
def _compute_price_company(self):
    for rec in self:
        rate = self.env.company.currency_id.rate
        rec.price_company = rec.price_unit * rate

# ✅ CORRECT — label varies by UI language
@api.depends_context('lang')
def _compute_translated_name(self):
    for rec in self:
        rec.translated_name = rec.with_context(
            lang=self.env.context.get('lang')
        ).name
```

**Gotcha:**

```python
# ❌ WRONG — store=True with depends_context — context varies per user/session
price_company = fields.Monetary(
    compute='_compute_price_company',
    store=True,    # ❌ cannot store context-dependent values
)

# ✅ CORRECT
price_company = fields.Monetary(
    compute='_compute_price_company',
    store=False,   # always real-time
)
```

---

### `@api.constrains` — The Validation Decorator

**Rule:** Always loop over `self`. Always raise `ValidationError` (never `UserError`). Always list the fields that, when changed, should trigger the check.

```python
from odoo.exceptions import ValidationError

# ✅ CORRECT
@api.constrains('date_from', 'date_to')
def _check_dates(self):
    """Ensure date_from is before date_to."""
    for rec in self:
        if rec.date_from and rec.date_to and rec.date_from > rec.date_to:
            raise ValidationError(
                _("Start date (%s) must be before end date (%s).") % (
                    rec.date_from, rec.date_to
                )
            )

# ✅ CORRECT — cross-record check; always exclude self with ('id', '!=', rec.id)
@api.constrains('employee_id', 'date_from', 'date_to')
def _check_no_overlap(self):
    for rec in self:
        overlap = self.search([
            ('employee_id', '=', rec.employee_id.id),
            ('id', '!=', rec.id),           # ← REQUIRED to exclude current record
            ('date_from', '<=', rec.date_to),
            ('date_to', '>=', rec.date_from),
        ])
        if overlap:
            raise ValidationError(_("Leave request overlaps with an existing one."))
```

**Gotchas:**

```python
# ❌ WRONG — UserError instead of ValidationError
@api.constrains('qty')
def _check_qty(self):
    raise UserError("...")          # wrong exception class for constrains

# ❌ WRONG — empty decorator → method NEVER runs
@api.constrains()
def _check_something(self): ...    # no field listed = never triggered

# ❌ WRONG — not looping (breaks silently on multi-record writes)
@api.constrains('amount')
def _check_amount(self):
    if self.amount < 0:            # self may be multiple records
        raise ValidationError(...)

# ❌ WRONG — listing a field you don't actually read in the method
@api.constrains('name')            # listed, but never used inside
def _check_amount(self):
    for rec in self:
        if rec.amount < 0:         # 'amount' changes → check does NOT rerun
            raise ValidationError(...)

# ✅ CORRECT — list what you validate, loop always
@api.constrains('amount')
def _check_amount(self):
    for rec in self:
        if rec.amount < 0:
            raise ValidationError(_("Amount must be positive."))
```

---

### `@api.onchange` — UI-Only, Never Guaranteed

**Rule:** `@api.onchange` runs ONLY in the browser form. It does NOT run during programmatic `create()` or `write()`. If logic must be enforced on save, use `@api.constrains` or override `create`/`write` as well.

```python
# ✅ CORRECT — update UI fields when partner changes
@api.onchange('partner_id')
def _onchange_partner_id(self):
    if self.partner_id:
        self.street = self.partner_id.street
        self.country_id = self.partner_id.country_id
        if not self.partner_id.email:
            return {
                'warning': {
                    'title': _("Missing Email"),
                    'message': _("This customer has no email address on file."),
                }
            }
    else:
        self.street = False
        self.country_id = False

# ✅ CORRECT — return domain update to client
@api.onchange('product_id')
def _onchange_product_id(self):
    if self.product_id:
        self.price_unit = self.product_id.list_price
        return {
            'domain': {
                'uom_id': [('category_id', '=', self.product_id.uom_id.category_id.id)]
            }
        }
```

**Gotchas:**

```python
# ❌ WRONG — using self.id in a domain search inside onchange
@api.onchange('name')
def _onchange_name(self):
    existing = self.search([('name', '=', self.name), ('id', '!=', self.id)])
    # ❌ self.id is a NewId object (not an integer) — domain comparison breaks

# ❌ WRONG — assuming onchange runs on programmatic create
# self.env['my.model'].create({'partner_id': p.id})
# → _onchange_partner_id does NOT fire → street/country will NOT be filled
# ✅ Fix: set those values in default_get() or override create()

# ❌ WRONG — raising ValidationError in onchange to block save
@api.onchange('qty')
def _onchange_qty(self):
    if self.qty < 0:
        raise ValidationError(...)  # unreliable — does not always block save

# ✅ CORRECT pattern: warning in onchange, hard enforcement in constrains
@api.onchange('qty')
def _onchange_qty(self):
    if self.qty < 0:
        return {'warning': {'title': _("Invalid"), 'message': _("Qty must be positive.")}}

@api.constrains('qty')
def _check_qty(self):
    for rec in self:
        if rec.qty < 0:
            raise ValidationError(_("Qty must be positive."))
```

---

### `@api.model` vs `@api.model_create_multi`

**Rule:** `@api.model` is for class-level helpers that don't operate on a specific recordset. `@api.model_create_multi` is the **only** correct decorator for overriding `create()` in Odoo 17.0.

```python
# ✅ CORRECT — @api.model for default_get and class helpers
@api.model
def default_get(self, fields_list):
    """Inject custom defaults."""
    res = super().default_get(fields_list)
    if 'state' in fields_list:
        res['state'] = 'draft'
    if 'company_id' in fields_list:
        res['company_id'] = self.env.company.id
    return res

@api.model
def _get_default_stage(self):
    """Return first available stage."""
    return self.env['project.task.type'].search([], limit=1)
```

```python
# ✅ CORRECT — create() override in Odoo 17.0
@api.model_create_multi
def create(self, vals_list):
    """Assign sequence and post creation message."""
    for vals in vals_list:
        if not vals.get('name'):
            vals['name'] = self.env['ir.sequence'].next_by_code('my.model') or _('New')
    records = super().create(vals_list)
    for record in records:
        record.message_post(body=_("Record created."))
    return records
```

**Gotchas:**

```python
# ❌ WRONG — @api.model for create() is the Odoo 16 and older pattern
@api.model
def create(self, vals):            # single dict — does not support batch
    return super().create(vals)

# ❌ WRONG — modifying vals_list after super() has no effect
@api.model_create_multi
def create(self, vals_list):
    records = super().create(vals_list)
    for vals in vals_list:
        vals['name'] = 'ignored'   # ❌ records already created
    return records

# ❌ WRONG — calling super() inside the loop (creates records multiple times!)
@api.model_create_multi
def create(self, vals_list):
    for vals in vals_list:
        super().create([vals])     # ❌ creates one-by-one AND then...
    return super().create(vals_list)   # ❌ creates them ALL AGAIN

# ✅ CORRECT — modify vals BEFORE super(), call super() exactly ONCE
@api.model_create_multi
def create(self, vals_list):
    for vals in vals_list:
        vals['processed'] = False
    return super().create(vals_list)
```

---

## Decision Tree

```
Field derives its value from other fields?
  → @api.depends (list ALL dependencies, assign in EVERY branch)
  → Also add @api.depends_context if result depends on lang/company/user

Need to BLOCK save when data is invalid?
  → @api.constrains + raise ValidationError (loop over self, list correct fields)
  → NEVER rely on @api.onchange alone to block saves

Need to UPDATE form fields when user types in the UI?
  → @api.onchange (remember: does NOT run on programmatic create/write)
  → Add @api.constrains too if the rule must be enforced on save

Writing default_get or a class-level helper?
  → @api.model

Overriding create() in Odoo 17.0?
  → @api.model_create_multi (modify vals_list BEFORE super, call super ONCE)
```

---

## onchange vs constrains — Side-by-Side

|                              | `@api.onchange`             | `@api.constrains`                  |
| ---------------------------- | --------------------------- | ---------------------------------- |
| When it runs                 | User edits field in browser | After every `create()` / `write()` |
| Runs on programmatic create? | ❌ No                       | ✅ Yes                             |
| Can modify other fields?     | ✅ Yes                      | ❌ No                              |
| Can reliably block save?     | ⚠️ No                       | ✅ Yes (raise ValidationError)     |
| `self.id` is a real integer? | ❌ No (NewId object)        | ✅ Yes                             |
| Can safely search the DB?    | ⚠️ With caution             | ✅ Yes                             |
| Correct way to signal error  | Return `{'warning': {...}}` | `raise ValidationError(...)`       |

---

## Resources

- **Templates**: See [assets/](assets/) for method stubs for each decorator
- **Documentation**: See [references/](references/) for the official Odoo 17.0 ORM Decorators reference

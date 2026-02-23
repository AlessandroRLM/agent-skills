---
name: odoo-orm-fields
description: >
  Complete reference for all Odoo 17.0 field types and their options.
  Covers scalar fields (Char, Text, Integer, Float, Monetary, Boolean, Date,
  Datetime, Selection, Html, Binary), relational fields (Many2one, One2many,
  Many2many), and special field modes (compute, related, store, copy, index,
  required, readonly, tracking, groups).
  Trigger: When declaring any field on a model, choosing between field types,
  configuring compute/related/store options, or understanding field parameters.
license: Apache-2.0
metadata:
  author: AlessandroRLM
  version: "1.0"
---

## When to Use

Use this skill when:

- Declaring any new field on a model
- Choosing between field types (e.g. Float vs Monetary, Char vs Text)
- Deciding whether a computed field should be `store=True` or `store=False`
- Using `related=` to mirror a field from a linked model
- Configuring `index`, `copy`, `tracking`, `groups`, `required`, `readonly`
- Working with Many2one, One2many, or Many2many relationships

---

## Critical Patterns

### Pattern 1: Scalar Field Types

```python
from odoo import fields, models


class MyModel(models.Model):
    _name = 'my.model'
    _description = 'Field Types Reference'

    # ── Char ──────────────────────────────────────────────────────────────────
    # Single-line text. Maps to VARCHAR in PostgreSQL.
    # Use for: names, codes, references, emails, phone numbers.
    name = fields.Char(
        string='Name',        # UI label (defaults to field name if omitted)
        required=True,        # NOT NULL at DB level
        size=64,              # max character length (no DB enforcement by default)
        trim=True,            # strip leading/trailing whitespace on save (default True)
        translate=True,       # enable per-language translations
        index=True,           # add DB index (use on fields searched often)
        copy=False,           # exclude from record duplication
        default='',           # default value
    )

    # ── Text ──────────────────────────────────────────────────────────────────
    # Multi-line plain text. Maps to TEXT in PostgreSQL. No size limit.
    # Use for: notes, descriptions, comments.
    description = fields.Text(string='Description', translate=True)

    # ── Html ──────────────────────────────────────────────────────────────────
    # Rich text with HTML content (rendered by Odoo's HTML editor).
    # Maps to TEXT. Always sanitized by Odoo before save.
    # Use for: email bodies, product descriptions, website content.
    body = fields.Html(string='Content', translate=True, sanitize=True)

    # ── Integer ───────────────────────────────────────────────────────────────
    # Whole numbers. Maps to INTEGER in PostgreSQL.
    sequence = fields.Integer(string='Sequence', default=10)
    priority = fields.Integer(default=0)

    # ── Float ─────────────────────────────────────────────────────────────────
    # Decimal numbers. Maps to NUMERIC(precision, scale).
    # Use for: quantities, weights, percentages, rates.
    # digits=(total_digits, decimal_places)
    quantity = fields.Float(string='Quantity', digits=(16, 4))
    discount = fields.Float(string='Discount (%)', digits=(5, 2), default=0.0)

    # ── Monetary ──────────────────────────────────────────────────────────────
    # Decimal for currency amounts. ALWAYS pair with a currency_field.
    # Stores in the company's currency. Displays with currency symbol in UI.
    # Use for: prices, totals, costs — never use Float for money.
    price = fields.Monetary(
        string='Unit Price',
        currency_field='currency_id',   # REQUIRED — name of the Currency field
    )
    currency_id = fields.Many2one(
        'res.currency',
        related='company_id.currency_id',   # usually derived from company
        store=True,
    )

    # ── Boolean ───────────────────────────────────────────────────────────────
    # True/False. Maps to BOOLEAN in PostgreSQL.
    is_active = fields.Boolean(string='Active', default=True)
    is_confirmed = fields.Boolean(default=False)

    # ── Date ──────────────────────────────────────────────────────────────────
    # Calendar date (no time). Maps to DATE in PostgreSQL.
    # In Python: fields.Date.today(), fields.Date.context_today(self)
    deadline = fields.Date(string='Deadline')
    birth_date = fields.Date(string='Date of Birth')

    # ── Datetime ──────────────────────────────────────────────────────────────
    # Date + time stored in UTC. Maps to TIMESTAMP in PostgreSQL.
    # UI converts to user's timezone automatically.
    # In Python: fields.Datetime.now()
    scheduled_at = fields.Datetime(string='Scheduled At')
    confirmed_at = fields.Datetime(string='Confirmed On', readonly=True, copy=False)

    # ── Selection ─────────────────────────────────────────────────────────────
    # Fixed list of (value, label) pairs. Maps to VARCHAR.
    # Use for: status fields, type enumerations.
    state = fields.Selection(
        selection=[
            ('draft', 'Draft'),
            ('confirmed', 'Confirmed'),
            ('done', 'Done'),
            ('cancelled', 'Cancelled'),
        ],
        string='Status',
        default='draft',
        required=True,
        tracking=True,       # log changes in chatter (needs mail.thread)
    )

    # ── Binary ────────────────────────────────────────────────────────────────
    # Stores raw bytes (files, images). Maps to BYTEA in PostgreSQL.
    # For images, use fields.Image instead (auto-resizes).
    attachment = fields.Binary(string='Attachment', attachment=True)
    name_of_attachment = fields.Char(string='Attachment Filename')

    # ── Image ─────────────────────────────────────────────────────────────────
    # Binary field with automatic resizing.
    # max_width/max_height: resize on save (keeps aspect ratio).
    avatar = fields.Image(string='Avatar', max_width=128, max_height=128)
    image_1920 = fields.Image(max_width=1920, max_height=1920)  # Odoo convention
```

### Pattern 2: Relational Field Types

```python
    # ── Many2one ──────────────────────────────────────────────────────────────
    # N records of this model → 1 record of another model.
    # Stores the foreign key ID in this table.
    # Use for: partner, product, company, user references.
    partner_id = fields.Many2one(
        comodel_name='res.partner',   # target model
        string='Customer',
        required=True,
        index=True,                   # ALWAYS index Many2one fields used in searches
        ondelete='restrict',          # what happens when partner is deleted:
                                      # 'restrict' = block deletion (safest)
                                      # 'cascade' = delete this record too
                                      # 'set null' = set field to False
        domain=[('customer_rank', '>', 0)],   # filter dropdown options
        context={'show_vip': True},           # extra context for dropdown
    )

    company_id = fields.Many2one(
        'res.company',
        string='Company',
        default=lambda self: self.env.company,
        index=True,
    )

    user_id = fields.Many2one(
        'res.users',
        string='Responsible',
        default=lambda self: self.env.user,
        index=True,
    )

    # ── One2many ──────────────────────────────────────────────────────────────
    # 1 record of this model → N records of another model (reverse of Many2one).
    # Does NOT store anything in this table — reads from the child model.
    # REQUIRES the child model to have a Many2one back to this model.
    line_ids = fields.One2many(
        comodel_name='sale.order.line',   # child model
        inverse_name='order_id',           # the Many2one field on the child
        string='Order Lines',
        copy=True,                         # copy children when duplicating parent
    )

    # ── Many2many ─────────────────────────────────────────────────────────────
    # N records of this model ↔ N records of another model.
    # Creates a join table automatically (or you can specify one).
    tag_ids = fields.Many2many(
        comodel_name='project.tags',
        string='Tags',
        # Optional: specify join table + column names explicitly
        # (required if two Many2many fields point to the same model pair)
        # relation='project_task_tag_rel',
        # column1='task_id',
        # column2='tag_id',
    )

    # Many2many to res.users (very common)
    follower_ids = fields.Many2many(
        'res.users',
        'my_model_user_rel',    # explicit join table name (avoid auto-name collision)
        'model_id',
        'user_id',
        string='Followers',
    )
```

### Pattern 3: Computed, Related, and Default Fields

```python
    # ── Computed field (store=False) ──────────────────────────────────────────
    # Recalculated every time it is read. Not stored in DB.
    # Use when: value changes frequently, no need to search/sort by it.
    display_label = fields.Char(
        string='Label',
        compute='_compute_display_label',
        # No store= → defaults to False
    )

    # ── Computed field (store=True) ───────────────────────────────────────────
    # Written to DB when dependencies change. Can be searched and sorted.
    # Use when: value is searched in domains, used in ORDER BY, or shown in list views.
    amount_total = fields.Monetary(
        string='Total Amount',
        currency_field='currency_id',
        compute='_compute_amount_total',
        store=True,    # stored → searchable, sortable, visible in export
        index=True,    # add index if used in search domains often
    )

    # ── Related field ─────────────────────────────────────────────────────────
    # Mirrors a field from a linked model. Shorthand for a simple compute.
    # store=True to cache the value in DB (useful for search/order).
    partner_email = fields.Char(
        related='partner_id.email',
        string='Customer Email',
        store=True,      # store to allow searching
        readonly=True,   # related fields are readonly by default
    )

    partner_country_id = fields.Many2one(
        related='partner_id.country_id',
        store=True,
        readonly=True,
    )

    # ── Default values ────────────────────────────────────────────────────────
    # Static default
    state = fields.Selection(..., default='draft')
    priority = fields.Integer(default=0)

    # Dynamic default via lambda
    company_id = fields.Many2one('res.company', default=lambda self: self.env.company)
    user_id = fields.Many2one('res.users', default=lambda self: self.env.user)
    date = fields.Date(default=fields.Date.today)           # method reference
    created_at = fields.Datetime(default=fields.Datetime.now)
```

---

## Decision Tree

```
Single-line text?               → fields.Char
Multi-line plain text?          → fields.Text
Rich HTML content?              → fields.Html
Whole number?                   → fields.Integer
Decimal (non-monetary)?         → fields.Float (specify digits=)
Currency amount?                → fields.Monetary (+ currency_field= REQUIRED)
True/False toggle?              → fields.Boolean
Date only (no time)?            → fields.Date
Date + time?                    → fields.Datetime
Fixed list of choices?          → fields.Selection
File or raw bytes?              → fields.Binary (attachment=True for large files)
Photo / avatar?                 → fields.Image (max_width/max_height)

Link to one other record?       → fields.Many2one (add index=True)
List of child records?          → fields.One2many (needs inverse Many2one)
Tags / multiple records?        → fields.Many2many

Derived from calculation?       → compute= (store=False if not searched)
Needs to be searched/sorted?    → compute= + store=True + index=True
Mirror field from related model?→ related= (store=True if searched)
Log change in chatter?          → tracking=True (requires mail.thread)
```

---

## Code Examples

### Example 1: Monetary Field — Full Setup

```python
class SaleOrder(models.Model):
    _name = 'sale.order'

    # Always define currency_id alongside Monetary fields
    currency_id = fields.Many2one(
        'res.currency',
        related='company_id.currency_id',
        store=True,
        readonly=True,
    )
    company_id = fields.Many2one(
        'res.company',
        default=lambda self: self.env.company,
        required=True,
        index=True,
    )

    # Now Monetary fields can reference currency_id
    amount_untaxed = fields.Monetary(currency_field='currency_id', store=True)
    amount_tax = fields.Monetary(currency_field='currency_id', store=True)
    amount_total = fields.Monetary(currency_field='currency_id', store=True)
```

### Example 2: Selection Field — Extending Choices in Inherited Model

```python
# Extend the state selection of sale.order with a custom state
class SaleOrder(models.Model):
    _inherit = 'sale.order'

    state = fields.Selection(
        selection_add=[
            ('pending_approval', 'Pending Approval'),
        ],
        ondelete={'pending_approval': 'set default'},  # REQUIRED in 17.0 when adding states
    )
```

### Example 3: Many2many Write Commands (used in create/write vals)

```python
# ORM write commands for Many2many and One2many:
# (4, id)       → link existing record id
# (3, id)       → unlink (remove from relation, don't delete)
# (5,)          → clear all links
# (6, 0, [ids]) → replace all links with new list of ids
# (0, 0, vals)  → create new record and link

# Example: set tag_ids to exactly these two records
record.write({'tag_ids': [(6, 0, [tag1.id, tag2.id])]})

# Example: add one tag without removing others
record.write({'tag_ids': [(4, new_tag.id)]})

# Example: remove one tag
record.write({'tag_ids': [(3, old_tag.id)]})

# Example: clear all tags
record.write({'tag_ids': [(5,)]})
```

### Example 4: Field Visibility & Access Modifiers

```python
# groups= restricts field visibility to specific user groups
internal_cost = fields.Float(
    string='Internal Cost',
    groups='base.group_system',   # only visible to admins
)

# States-based readonly/required (Odoo 17: use XML invisible/required attrs instead)
# In Python, use readonly/required for absolute rules:
confirmed_at = fields.Datetime(readonly=True, copy=False)

# tracking= levels (requires _inherit = ['mail.thread'])
state = fields.Selection([...], tracking=True)          # always tracked
amount = fields.Monetary(tracking=10)                   # tracked with priority 10
```

---

## Commands

```bash
# After adding/changing fields, upgrade the module to apply schema changes
./odoo-bin -d mydb -u my_module

# Check column type in PostgreSQL
psql -d mydb -c "\d my_model" | grep column_name

# Check Many2many join table was created
psql -d mydb -c "\dt *my_module*"

# Check field metadata in ORM (shell)
./odoo-bin shell -d mydb -e "print(env['my.model']._fields['partner_id'])"
```

---

## Resources

- **Templates**: See [assets/](assets/) for a field declaration cheatsheet template
- **Documentation**: See [references/](references/) for the official Odoo 17.0 ORM — Fields reference

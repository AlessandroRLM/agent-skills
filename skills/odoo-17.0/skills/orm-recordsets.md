---
name: odoo-orm-recordsets
description: >
  Complete guide to Odoo 17.0 recordset operations: creating records (create),
  reading (search, browse, read, name_search), updating (write), deleting (unlink),
  recordset manipulation (mapped, filtered, sorted, exists, ids), domain syntax,
  environment (env, sudo, with_context, with_company, with_user), and
  transactional patterns (savepoints, cr.execute).
  Trigger: When querying, creating, modifying, or deleting records in Python;
  working with self.env; writing search domains; or chaining recordset operations.
license: Apache-2.0
metadata:
  author: AlessandroRLM
  version: "1.0"
---

## When to Use

Use this skill when:

- Writing any code that reads from or writes to the database
- Building search domains for `search()`, `search_count()`, or view filters
- Using `self.env` to access other models or user/company context
- Chaining `mapped()`, `filtered()`, `sorted()` on a recordset
- Needing to run code as superuser (`sudo()`) or in a different company
- Handling transactions with savepoints or raw SQL cursors

---

## Critical Patterns

### Pattern 1: The Domain Language

Domains are lists of conditions joined by implicit AND. Operators must be prefixed before the conditions they join.

```python
# ── Condition tuple ────────────────────────────────────────────────────────────
# ('field_name', 'operator', value)
# Operators: =, !=, <, >, <=, >=, like, ilike, =like, =ilike,
#            in, not in, child_of, parent_of

# ── Simple AND (default) ───────────────────────────────────────────────────────
domain = [
    ('state', '=', 'sale'),
    ('amount_total', '>', 1000),
    ('user_id', '=', self.env.uid),
]
# SQL: state = 'sale' AND amount_total > 1000 AND user_id = <uid>

# ── OR with '|' prefix operator ───────────────────────────────────────────────
# '|' applies to the next TWO conditions after it
domain = ['|',
    ('state', '=', 'sale'),
    ('state', '=', 'done'),
]
# SQL: state = 'sale' OR state = 'done'

# ── NOT with '!' prefix operator ──────────────────────────────────────────────
domain = ['!', ('state', '=', 'cancelled')]
# SQL: NOT (state = 'cancelled')   → equivalent to ('state', '!=', 'cancelled')

# ── Complex: (A AND B) OR (C AND D) ──────────────────────────────────────────
domain = ['|',
    '&', ('state', '=', 'sale'), ('amount_total', '>', 1000),
    '&', ('state', '=', 'done'), ('user_id', '=', self.env.uid),
]

# ── Text search operators ─────────────────────────────────────────────────────
('name', 'ilike', 'john')        # case-insensitive LIKE %john%
('name', 'like', 'John')         # case-sensitive LIKE %John%
('name', '=ilike', 'john%')      # case-insensitive exact pattern
('ref', '=like', 'SO%')          # case-sensitive exact pattern (= SQL LIKE)

# ── Relational field operators ────────────────────────────────────────────────
('partner_id', '=', partner.id)                  # exact Many2one match
('partner_id.country_id', '=', country.id)       # traverse relation
('tag_ids', 'in', [tag1.id, tag2.id])            # any of these tags
('tag_ids', 'not in', [tag.id])                  # none of these tags
('child_ids', 'child_of', parent.id)             # all children of parent

# ── Date comparisons ──────────────────────────────────────────────────────────
from odoo import fields
today = fields.Date.today()
domain = [('deadline', '<', today)]

# ── Empty / not empty ─────────────────────────────────────────────────────────
('partner_id', '=', False)     # is empty (null)
('partner_id', '!=', False)    # is not empty
```

---

### Pattern 2: `search()` — Query Records

```python
# search(domain, offset=0, limit=None, order=None) → recordset
# Returns a recordset. Empty recordset if nothing found (not None).

# Basic search
orders = self.env['sale.order'].search([('state', '=', 'sale')])

# With limit and order
recent = self.env['sale.order'].search(
    [('state', '=', 'sale')],
    limit=10,
    order='date_order desc',    # SQL ORDER BY — must be a stored field
)

# Count only (generates SELECT COUNT(*) — much faster)
count = self.env['sale.order'].search_count([('state', '=', 'sale')])

# ✅ Use search_count when you only need the number, not the records
# ❌ NEVER: count = len(self.env['sale.order'].search([...])) — loads all records

# search_read → returns list of dicts (avoids loading full ORM objects)
# Useful for large datasets where you need specific fields only
data = self.env['sale.order'].search_read(
    domain=[('state', '=', 'sale')],
    fields=['name', 'partner_id', 'amount_total'],
    limit=100,
    order='date_order desc',
)
# Returns: [{'id': 1, 'name': 'S001', 'partner_id': [5, 'Azure'], 'amount_total': 500.0}, ...]
```

### Pattern 3: `browse()` — Fetch by Known IDs

```python
# browse(ids) → recordset — does NOT hit the DB immediately (lazy)
# DB query happens only when you access a field

# Single record
partner = self.env['res.partner'].browse(42)
print(partner.name)   # DB hit happens here

# Multiple records
partners = self.env['res.partner'].browse([1, 2, 3])

# ✅ Use browse when you already KNOW the IDs (e.g. from context or parameters)
# ❌ Don't use browse with user-supplied IDs without validation — use search() instead
```

### Pattern 4: `create()` — Insert Records

```python
# Single record
order = self.env['sale.order'].create({
    'partner_id': partner.id,
    'state': 'draft',
})

# ✅ PREFERRED in 17.0: create multiple records in one SQL call
orders = self.env['sale.order'].create([
    {'partner_id': p1.id, 'state': 'draft'},
    {'partner_id': p2.id, 'state': 'draft'},
])
# Returns a recordset with both records

# Many2many write commands inside create vals
task = self.env['project.task'].create({
    'name': 'My Task',
    'project_id': project.id,
    'tag_ids': [(4, tag1.id), (4, tag2.id)],    # link existing tags
    'tag_ids': [(6, 0, [tag1.id, tag2.id])],    # replace all tags (alternative)
})

# One2many create children inline
order = self.env['sale.order'].create({
    'partner_id': partner.id,
    'order_line': [(0, 0, {'product_id': prod.id, 'product_uom_qty': 1.0})],
})
```

### Pattern 5: `write()` — Update Records

```python
# Write on single record
order.write({'state': 'sale', 'note': 'Approved'})

# ✅ Write on entire recordset — one UPDATE query for all records
orders.write({'state': 'cancelled'})

# Update relational fields
record.write({
    'tag_ids': [(4, new_tag.id)],          # add one tag
    'tag_ids': [(3, old_tag.id)],          # remove one tag
    'tag_ids': [(5,)],                      # clear all tags
    'tag_ids': [(6, 0, [t1.id, t2.id])],  # replace all tags
    'line_ids': [(0, 0, {'name': 'New Line'})],  # create child
    'line_ids': [(2, line.id)],            # delete child record
    'line_ids': [(1, line.id, {'qty': 5})],# update existing child
})

# ❌ NEVER write in a loop — one UPDATE per record is very slow
for order in orders:
    order.write({'state': 'sale'})   # N SQL queries

# ✅ Write once on the recordset
orders.write({'state': 'sale'})      # 1 SQL query
```

### Pattern 6: `unlink()` — Delete Records

```python
# Delete a single record
record.unlink()

# Delete entire recordset in one call
old_records.unlink()

# ✅ Best practice: filter before unlink
stale = self.env['mail.message'].search([
    ('date', '<', cutoff),
    ('message_type', '=', 'notification'),
])
stale.unlink()
```

---

### Pattern 7: Recordset Manipulation Methods

```python
# ── mapped(field_or_func) ─────────────────────────────────────────────────────
# Extract field values. Returns a list for scalar fields, recordset for relational.

names = orders.mapped('name')                     # ['SO001', 'SO002'] — list of str
partners = orders.mapped('partner_id')            # recordset of res.partner (deduplicated)
subtotals = lines.mapped('price_subtotal')        # [100.0, 200.0] — list of floats
total = sum(lines.mapped('price_subtotal'))       # sum via mapped + sum()

# Lambda for computed extraction
discounted = orders.mapped(lambda o: o.amount_total * 0.9)

# ── filtered(field_or_func) ───────────────────────────────────────────────────
# Filter in Python — no new DB query. Returns a recordset.

active_orders = orders.filtered('active')                            # truthy field
sale_orders = orders.filtered(lambda o: o.state == 'sale')          # lambda
high_value = orders.filtered(lambda o: o.amount_total > 1000)
mine = orders.filtered(lambda o: o.user_id == self.env.user)

# Chain filtered + mapped
partner_names = orders.filtered('active').mapped('partner_id.name')

# ── sorted(key, reverse=False) ────────────────────────────────────────────────
# Sort in Python — no new DB query. Returns a new recordset.

by_date = orders.sorted('date_order')
by_date_desc = orders.sorted('date_order', reverse=True)
by_lambda = orders.sorted(lambda o: o.amount_total)

# ── exists() ─────────────────────────────────────────────────────────────────
# Filter out records that no longer exist in DB (e.g. deleted by another process)
records = records.exists()

# ── ids property ─────────────────────────────────────────────────────────────
# Get list of integer IDs
id_list = records.ids      # [1, 2, 3]

# ── ensure_one() ─────────────────────────────────────────────────────────────
# Raises ValueError if recordset has ≠ 1 record. Use in methods expecting single rec.
def action_do_something(self):
    self.ensure_one()   # raises if called on 0 or 2+ records
    return {'type': 'ir.actions.act_window', 'res_id': self.id, ...}

# ── Set operations ────────────────────────────────────────────────────────────
combined = record_a | record_b           # union (no duplicates)
common = record_a & record_b             # intersection
diff = record_a - record_b               # difference
in_a = record_a in record_b             # membership check
```

---

### Pattern 8: Environment — `self.env`

```python
# ── Access other models ───────────────────────────────────────────────────────
partners = self.env['res.partner'].search([('customer_rank', '>', 0)])
partner = self.env['res.partner'].browse(partner_id)

# ── Current user and company ──────────────────────────────────────────────────
user = self.env.user                  # res.users record
uid = self.env.uid                    # integer user ID
company = self.env.company            # res.company record
companies = self.env.companies        # all companies in multi-company context
lang = self.env.lang                  # e.g. 'en_US'

# ── Context ───────────────────────────────────────────────────────────────────
ctx = self.env.context                # dict — READ ONLY
value = self.env.context.get('my_key', default_value)

# ── sudo() — bypass access rights ────────────────────────────────────────────
# Use sparingly. Always add a comment explaining WHY.
# NEVER expose sudo() results back to user without re-checking rights.
record.sudo().write({'internal_field': value})           # bypass write rights
count = self.env['hr.payslip'].sudo().search_count([])   # bypass read rights

# ── with_context() — add/change context keys ─────────────────────────────────
# Returns a new environment — does NOT modify self.env
records_fr = self.env['res.partner'].with_context(lang='fr_FR').browse(ids)
no_recompute = self.with_context(no_recompute=True)

# ── with_user() — run as another user ────────────────────────────────────────
other_env = self.env.with_user(other_user)
other_env['sale.order'].create({...})   # creates as other_user

# ── with_company() — switch company ───────────────────────────────────────────
company_env = self.env.with_company(other_company)

# ── ref() — get record by XML ID ──────────────────────────────────────────────
admin = self.env.ref('base.user_admin')
group = self.env.ref('base.group_user')
```

### Pattern 9: Transactions and Raw SQL

```python
import logging
_logger = logging.getLogger(__name__)


def _heavy_data_migration(self):
    """Batch update without loading ORM objects."""
    # ✅ Use parameterized queries — NEVER format SQL with user input
    self.env.cr.execute("""
        UPDATE my_module_model
        SET state = %s,
            processed_at = NOW()
        WHERE state = %s
          AND create_date < %s
          AND company_id = %s
    """, ('done', 'draft', cutoff_date, self.env.company.id))

    rows_updated = self.env.cr.rowcount
    _logger.info("Updated %d records to done.", rows_updated)

    # Invalidate ORM cache after raw SQL writes
    self.env['my.module.model'].invalidate_model(['state', 'processed_at'])

# ── Savepoints — partial rollback ─────────────────────────────────────────────
def _process_with_savepoint(self, records):
    """Process each record independently — skip failures."""
    for record in records:
        try:
            with self.env.cr.savepoint():
                record._do_risky_operation()
        except Exception as e:
            _logger.warning("Failed to process record %s: %s", record.id, e)
            # Savepoint rolled back — continue with next record
```

---

## Decision Tree

```
Query records matching conditions?       → search(domain)
Count records?                           → search_count(domain) — NOT len(search())
Fetch specific fields as dicts?          → search_read(domain, fields)
Fetch by known IDs?                      → browse(ids)
Create one record?                       → create({vals})
Create many records at once?             → create([vals1, vals2, ...])
Update all records in set?               → recordset.write({vals})
Update one field on one record?          → record.field = value (triggers write)
Delete records?                          → recordset.unlink()
Extract field values?                    → mapped('field')
Filter in Python (no DB)?               → filtered(lambda)
Sort in Python (no DB)?                 → sorted('field')
Remove deleted from set?                → exists()
Ensure single record?                   → ensure_one()
Run as superuser?                       → sudo() + comment why
Change language/context?               → with_context(lang='xx_XX')
Write raw SQL?                          → env.cr.execute(sql, params) + invalidate_model
Partial rollback on failure?            → env.cr.savepoint()
```

---

## Commands

```bash
# Open shell to test recordset operations interactively
./odoo-bin shell -d mydb

# Inside shell — example session:
# env['sale.order'].search([('state','=','sale')], limit=3).mapped('name')
# env['res.partner'].search_count([('customer_rank','>',0)])
# env.ref('base.user_admin').name

# Check SQL executed (enable in odoo.conf: log_level = debug_sql)
# Then run your operation and inspect logs

# Analyze slow queries
# psql -d mydb -c "SELECT query, mean_time FROM pg_stat_statements ORDER BY mean_time DESC LIMIT 5;"
```

---

## Resources

- **Templates**: See [assets/](assets/) for domain builder snippets and common search patterns
- **Documentation**: See [references/](references/) for the official Odoo 17.0 ORM — Recordsets reference

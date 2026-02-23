---
name: odoo-performance
description: >
  Performance best practices for Odoo 17.0 development. Covers prefetch,
  N+1 query prevention, index usage, batch operations, computed field
  storage strategy, SQL vs ORM tradeoffs, and cache management.
  Trigger: When writing loops over recordsets, designing compute fields with store,
  optimizing slow views or reports, or any code that processes many records at once.
license: Apache-2.0
metadata:
  author: AlessandroRLM
  version: "1.0"
---

## When to Use

Use this skill when:

- Writing any `for rec in self:` loop (N+1 query risk)
- Designing computed fields and deciding `store=True` vs `store=False`
- Adding fields that will be used in `search()` domains or `order=` clauses
- Writing code that creates, writes, or reads many records at once
- Experiencing slow list views, kanban views, or report generation
- Choosing between ORM operations and raw SQL for heavy data processing

---

## Critical Patterns

### Pattern 1: Avoid N+1 — Use Prefetch and `mapped()`

```python
# ❌ BAD — triggers one SQL query per record (N+1 problem)
def bad_compute(self):
    for rec in self:
        rec.partner_name = rec.partner_id.name  # hits DB each iteration

# ✅ GOOD — Odoo's ORM auto-prefetches Many2one targets in the same query batch
# Just write compute normally; prefetch happens automatically when iterating self
@api.depends('partner_id.name')
def _compute_partner_name(self):
    for rec in self:
        rec.partner_name = rec.partner_id.name  # prefetch batch handles this

# ✅ GOOD — Use mapped() to fetch a field across a recordset in one query
all_names = self.mapped('partner_id.name')

# ✅ GOOD — filtered() operates in Python, no additional SQL
active_lines = self.line_ids.filtered(lambda l: l.qty > 0)
```

### Pattern 2: Batch Creates and Writes

```python
# ❌ BAD — one INSERT per record
for vals in data_list:
    self.env['my.model'].create(vals)

# ✅ GOOD — single multi-insert (use create_multi in 17.0)
self.env['my.model'].create(data_list)

# ❌ BAD — one UPDATE per record
for rec in records:
    rec.write({'state': 'done'})

# ✅ GOOD — one UPDATE for all records
records.write({'state': 'done'})
```

### Pattern 3: Indexes for Searchable / Sortable Fields

```python
class SaleOrder(models.Model):
    _name = 'sale.order'

    # ✅ Add index=True on fields used in domain searches or ORDER BY
    partner_id = fields.Many2one('res.partner', index=True)
    state = fields.Selection([...], index=True)
    date_order = fields.Datetime(index=True)

    # Many2one fields are indexed automatically in Odoo 17
    # Char fields used in search: add index manually
    reference = fields.Char(index=True)
```

---

## Decision Tree

```
Computed field searched in domain?     → store=True + index=True
Computed field rarely searched?        → store=False (real-time)
Computed field on header (not lines)?  → store=True (avoid recalculate in list)
Loop over self in compute?             → Rely on ORM prefetch; use mapped() for related
Creating many records?                 → Pass list to create([...])
Writing same value to many records?    → Call write() on whole recordset
Counting records?                      → search_count([domain]) not len(search())
Large data export/report?             → Use read_group() or raw SQL with cursor
Need to disable prefetch temporarily? → with_prefetch() context (advanced)
```

---

## Code Examples

### Example 1: `search_count` vs `search` for Counting

```python
# ❌ BAD — fetches all records then counts in Python
count = len(self.env['sale.order'].search([('state', '=', 'sale')]))

# ✅ GOOD — generates SELECT COUNT(*) — much faster
count = self.env['sale.order'].search_count([('state', '=', 'sale')])
```

### Example 2: `read_group` for Aggregation

```python
# ✅ Use read_group for grouped aggregations (avoids loading full records)
result = self.env['sale.order'].read_group(
    domain=[('state', '=', 'sale')],
    fields=['partner_id', 'amount_total:sum'],
    groupby=['partner_id'],
)
# result: [{'partner_id': (1, 'Azure'), 'amount_total': 5200.0, ...}, ...]
```

### Example 3: Prefetch Context Control

```python
# When iterating a large recordset and you only need one field,
# use with_prefetch=False to prevent loading unneeded fields
for rec in large_recordset.with_context(prefetch_fields=False):
    process(rec.name)

# Or selectively read only needed fields
records.read(['name', 'state'])  # returns list of dicts — no extra fields loaded
```

### Example 4: Raw SQL for Heavy Reports (use sparingly)

```python
def _get_sales_summary(self):
    """Aggregate sales data for reporting. ORM cannot do this efficiently."""
    self.env.cr.execute("""
        SELECT
            p.id AS partner_id,
            SUM(so.amount_total) AS total,
            COUNT(so.id) AS order_count
        FROM sale_order so
        JOIN res_partner p ON p.id = so.partner_id
        WHERE so.state = 'sale'
          AND so.company_id = %s
        GROUP BY p.id
        ORDER BY total DESC
        LIMIT 20
    """, [self.env.company.id])
    return self.env.cr.dictfetchall()
    # NEVER use string formatting for SQL params — always use %s placeholders!
```

### Example 5: `invalidate_recordset` After Manual SQL

```python
def _archive_old_records(self):
    self.env.cr.execute("""
        UPDATE my_module_model
        SET active = FALSE
        WHERE create_date < NOW() - INTERVAL '90 days'
    """)
    # Invalidate ORM cache so subsequent reads see updated data
    self.env['my.module.model'].invalidate_model(['active'])
```

---

## Commands

```bash
# Enable SQL query logging to find slow queries
# In odoo.conf:
# log_level = debug_sql

# Profile a specific HTTP request (developer mode)
# URL: ?debug=1 then use the performance tab in browser dev tools

# Check missing indexes in PostgreSQL
# psql -d mydb -c "SELECT relname, seq_scan, idx_scan FROM pg_stat_user_tables ORDER BY seq_scan DESC LIMIT 20;"

# Analyze slow queries
# psql -d mydb -c "SELECT query, calls, total_time, mean_time FROM pg_stat_statements ORDER BY mean_time DESC LIMIT 10;"
```

---

## Resources

- **Templates**: See [assets/](assets/) for a performance-optimized compute field template
- **Documentation**: See [references/](references/) for the official Odoo 17.0 Performance reference

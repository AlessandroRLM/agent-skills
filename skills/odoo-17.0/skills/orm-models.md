---
name: odoo-orm-models
description: >
  Defines how to correctly declare Odoo 17.0 model classes: choosing between
  models.Model, models.TransientModel, and models.AbstractModel; configuring
  class-level attributes (_name, _description, _inherit, _inherits, _order,
  _rec_name, _table, _log_access, _sql_constraints); and structuring the full
  anatomy of a model file in the correct order.
  Trigger: When creating a new model class, choosing how to extend an existing
  model, setting ordering/display name rules, or adding SQL-level constraints.
license: Apache-2.0
metadata:
  author: AlessandroRLM
  version: "1.0"
---

## When to Use

Use this skill when:

- Deciding which model base class to use (Model vs TransientModel vs AbstractModel)
- Creating a brand-new model with `_name`
- Extending an existing Odoo model with `_inherit` or `_inherits`
- Configuring `_order`, `_rec_name`, `_table`, or `_sql_constraints`
- Structuring the internal order of a model file correctly

---

## Critical Patterns

### Pattern 1: The Three Model Types — When to Use Each

```python
from odoo import models

# ── 1. models.Model ───────────────────────────────────────────────────────────
# Persistent records stored in the database.
# Use for: any business object that needs to be saved (orders, partners, tasks…)
class SaleOrder(models.Model):
    _name = 'sale.order'
    _description = 'Sale Order'


# ── 2. models.TransientModel ──────────────────────────────────────────────────
# Temporary records auto-deleted by a vacuum job (default: 1 hour).
# Use for: wizards, multi-step dialogs, import/export helpers.
# NEVER store important data here — records are deleted automatically.
class ImportWizard(models.TransientModel):
    _name = 'my.import.wizard'
    _description = 'CSV Import Wizard'


# ── 3. models.AbstractModel ───────────────────────────────────────────────────
# No database table — only exists as a reusable mixin.
# Use for: shared behaviour that multiple models inherit (like mail.thread).
# Cannot be instantiated directly — must be _inherit-ed.
class TimestampMixin(models.AbstractModel):
    _name = 'my.timestamp.mixin'
    _description = 'Adds processed_at timestamp to any model'
```

### Pattern 2: Key Class-Level Attributes

```python
class ProjectTask(models.Model):
    # REQUIRED — technical name, dot-notation, maps to DB table project_task
    _name = 'project.task'

    # REQUIRED — human-readable label shown in UI and error messages
    _description = 'Project Task'

    # Inherit fields + behaviour from another model (same _name = extension)
    # List form used when also inheriting from mixins
    _inherit = ['mail.thread', 'mail.activity.mixin']

    # Default ORDER BY for search() and list views
    # Always include 'id' as tiebreaker to ensure stable pagination
    _order = 'priority desc, sequence, id desc'

    # Field used as the display name (replaces default 'name' field)
    # Affects name_get(), display_name, and Many2one dropdowns
    _rec_name = 'full_reference'

    # Override the DB table name (default: _name with dots → underscores)
    # Only use this when migrating legacy tables
    _table = 'project_task'

    # False disables create_uid, create_date, write_uid, write_date columns
    # Only set False for very high-volume technical tables
    _log_access = True   # default True — almost always leave this

    # Database-level UNIQUE and CHECK constraints
    # Enforced at DB level — faster than @api.constrains for uniqueness
    _sql_constraints = [
        # (constraint_name, SQL_expression, error_message)
        ('name_project_uniq', 'unique(name, project_id)',
         'Task name must be unique within a project.'),
        ('positive_sequence', 'CHECK(sequence >= 0)',
         'Sequence must be a positive number.'),
    ]
```

### Pattern 3: `_inherit` — The Four Inheritance Modes

```python
# ── Mode 1: Extension (same _name, no new _name) ─────────────────────────────
# Adds fields/methods to an existing model WITHOUT creating a new table.
# This is the most common pattern for customizing standard Odoo models.
class SaleOrder(models.Model):
    _inherit = 'sale.order'   # string, not list — single model

    my_custom_field = fields.Char(string='Custom Reference')


# ── Mode 2: New model + copy behaviour (new _name + _inherit) ─────────────────
# Creates a NEW table but copies all fields/methods from the parent.
# Rarely used. Use when you want a separate model based on another.
class SpecialOrder(models.Model):
    _name = 'special.order'
    _inherit = 'sale.order'   # copy, not extend


# ── Mode 3: Mixin composition (list of AbstractModels) ────────────────────────
# Inherit behaviour from one or more abstract models/mixins.
# _inherit as a list means "compose from these".
class HelpDeskTicket(models.Model):
    _name = 'helpdesk.ticket'
    _description = 'Help Desk Ticket'
    _inherit = ['mail.thread', 'mail.activity.mixin', 'my.timestamp.mixin']


# ── Mode 4: _inherits (delegation / composition) ─────────────────────────────
# Links to a parent model via a Many2one — parent fields appear on this model.
# The child has its own table. Like "has-a" with field delegation.
# Rarely used in custom modules — mainly in Odoo core (res.users → res.partner).
class HrEmployee(models.Model):
    _name = 'hr.employee'
    _inherits = {'res.partner': 'partner_id'}   # delegates partner fields

    partner_id = fields.Many2one('res.partner', required=True, ondelete='cascade')
```

---

## Decision Tree

```
Stores data permanently?              → models.Model
Temporary form / wizard?              → models.TransientModel
Reusable behaviour for other models?  → models.AbstractModel

Modify existing Odoo model?           → _inherit = 'existing.model' (no _name)
Brand-new model?                      → _name = 'my.model' + _description (required)
New model based on existing one?      → _name = 'new.model' + _inherit = 'existing.model'
Add mixin behaviour?                  → _inherit = ['mail.thread', ...] (list)
Delegate fields from parent table?    → _inherits = {'parent.model': 'field_id'}

Custom sort in list views?            → _order = 'field1 desc, field2, id'
Display name ≠ 'name' field?          → _rec_name = 'other_field'
Uniqueness constraint needed?         → _sql_constraints (faster than @api.constrains)
```

---

## Code Examples

### Example 1: Correct Model File Structure (internal order)

```python
# models/project_task.py
from odoo import api, fields, models, _
from odoo.exceptions import UserError, ValidationError


class ProjectTask(models.Model):
    # ── 1. Class attributes ───────────────────────────────
    _name = 'project.task'
    _description = 'Project Task'
    _inherit = ['mail.thread', 'mail.activity.mixin']
    _order = 'sequence, priority desc, id desc'
    _rec_name = 'name'

    # ── 2. SQL constraints ────────────────────────────────
    _sql_constraints = [
        ('name_project_uniq', 'unique(name, project_id)',
         'Task name must be unique per project.'),
    ]

    # ── 3. Fields (see odoo-orm-fields skill for field rules) ─
    name = fields.Char(...)
    sequence = fields.Integer(...)
    project_id = fields.Many2one(...)

    # ── 4. Python constraints ─────────────────────────────
    @api.constrains('deadline')
    def _check_deadline(self): ...

    # ── 5. Compute methods ────────────────────────────────
    @api.depends('line_ids.price_total')
    def _compute_amount_total(self): ...

    # ── 6. Onchange methods ───────────────────────────────
    @api.onchange('project_id')
    def _onchange_project_id(self): ...

    # ── 7. CRUD overrides ─────────────────────────────────
    @api.model_create_multi
    def create(self, vals_list): ...

    def write(self, vals): ...

    def unlink(self): ...

    # ── 8. Action methods (called from buttons/menus) ─────
    def action_set_done(self): ...

    # ── 9. Private/helper methods ─────────────────────────
    def _compute_display_reference(self): ...
```

### Example 2: Extending `res.partner` (very common pattern)

```python
# models/res_partner.py
from odoo import fields, models


class ResPartner(models.Model):
    _inherit = 'res.partner'   # NO _name — extends existing table

    customer_code = fields.Char(
        string='Customer Code',
        copy=False,
        index=True,
    )
    credit_limit = fields.Monetary(
        string='Credit Limit',
        currency_field='currency_id',
    )
    is_vip = fields.Boolean(string='VIP Customer', default=False)
```

### Example 3: Abstract Mixin — Reusable Across Models

```python
# models/mixins.py
from odoo import api, fields, models


class AuditMixin(models.AbstractModel):
    """Add audit fields: processed_by and processed_at to any model."""
    _name = 'my.audit.mixin'
    _description = 'Audit Mixin'

    processed_by = fields.Many2one('res.users', readonly=True, copy=False)
    processed_at = fields.Datetime(readonly=True, copy=False)

    def _mark_as_processed(self):
        """Stamp the record with current user and timestamp."""
        self.write({
            'processed_by': self.env.uid,
            'processed_at': fields.Datetime.now(),
        })


# Usage in any other model:
class PurchaseOrder(models.Model):
    _name = 'my.purchase.order'
    _description = 'Purchase Order'
    _inherit = ['my.audit.mixin', 'mail.thread']
    # Now has processed_by, processed_at, and _mark_as_processed()
```

---

## Commands

```bash
# After adding/changing a model, always upgrade the module
./odoo-bin -d mydb -u my_module

# Check the generated DB table and columns
psql -d mydb -c "\d project_task"

# Verify model is registered (shell)
./odoo-bin shell -d mydb -e "print(env['project.task']._table)"

# Inspect SQL constraints on a table
psql -d mydb -c "\d+ project_task" | grep -i constraint
```

---

## Resources

- **Templates**: See [assets/](assets/) for Model, TransientModel, and AbstractModel file templates
- **Documentation**: See [references/](references/) for the official Odoo 17.0 ORM — Models reference

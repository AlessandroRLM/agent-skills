---
name: odoo-actions
description: >
  Guide for defining all Odoo 17.0 action types: window actions (ir.actions.act_window),
  server actions (ir.actions.server), URL actions (ir.actions.act_url), client actions
  (ir.actions.client), and scheduled actions (ir.cron). Covers action binding to menus,
  buttons, and server-side triggers.
  Trigger: When creating menu items, button actions, wizard launchers, cron jobs,
  redirects, or any system response to user interaction in Odoo 17.0.
license: Apache-2.0
metadata:
  author: AlessandroRLM
  version: "1.0"
---

## When to Use

Use this skill when:

- Adding a menu item that opens a model's list or form view
- Creating a button that triggers Python logic or opens a wizard
- Defining a scheduled action (cron job)
- Opening a URL or external page from a button
- Returning an action dict from a Python method
- Binding a server action to a model's action menu ("Action" dropdown)

---

## Critical Patterns

### Pattern 1: Window Action + Menu Item (most common)

```xml
<!-- views/menus.xml -->
<odoo>
    <!-- 1. Define the action -->
    <record id="action_my_model_list" model="ir.actions.act_window">
        <field name="name">My Models</field>
        <field name="res_model">my.module.model</field>
        <field name="view_mode">list,form</field>
        <field name="domain">[]</field>
        <field name="context">{}</field>
        <field name="help" type="html">
            <p class="o_view_nocontent_smiling_face">
                No records found. Create your first one!
            </p>
        </field>
    </record>

    <!-- 2. Top-level menu (no parent = app root) -->
    <menuitem
        id="menu_my_module_root"
        name="My Module"
        sequence="10"
        web_icon="my_module,static/description/icon.png"
    />

    <!-- 3. Sub-menu category -->
    <menuitem
        id="menu_my_module_config"
        name="Configuration"
        parent="menu_my_module_root"
        sequence="90"
    />

    <!-- 4. Leaf menu → binds to action -->
    <menuitem
        id="menu_my_model_list"
        name="My Models"
        parent="menu_my_module_root"
        action="action_my_model_list"
        sequence="10"
    />
</odoo>
```

### Pattern 2: Server Action (Python triggered from UI)

```xml
<!-- Bound to model's "Action" dropdown -->
<record id="action_send_reminder" model="ir.actions.server">
    <field name="name">Send Reminder</field>
    <field name="model_id" ref="model_my_module_model"/>
    <field name="binding_model_id" ref="model_my_module_model"/>
    <field name="binding_type">action</field>
    <field name="state">code</field>
    <field name="code">
        records.action_send_reminder()
    </field>
</record>
```

### Pattern 3: Return Action From Python (open wizard or view)

```python
def action_open_wizard(self):
    """Open the refund wizard for selected records."""
    return {
        'type': 'ir.actions.act_window',
        'name': _('Create Refund'),
        'res_model': 'my.module.wizard',
        'view_mode': 'form',
        'target': 'new',          # 'new' = dialog, 'current' = same page
        'context': {
            'default_origin_id': self.id,
            'default_partner_id': self.partner_id.id,
        },
    }

def action_view_related_orders(self):
    """Open list/form of related orders."""
    orders = self.mapped('order_ids')
    action = self.env['ir.actions.act_window']._for_xml_id(
        'sale.action_quotations_with_onboarding'
    )
    if len(orders) == 1:
        action['res_id'] = orders.id
        action['view_mode'] = 'form'
    else:
        action['domain'] = [('id', 'in', orders.ids)]
    return action
```

---

## Decision Tree

```
Open a model's records?          → ir.actions.act_window
Run Python code on records?      → ir.actions.server (state='code')
Open external URL?               → ir.actions.act_url
Open OWL client component?       → ir.actions.client
Run on a schedule?               → ir.cron
Open dialog/wizard?              → act_window with target='new'
Open form of specific record?    → act_window + res_id + view_mode='form'
Bind to model Action menu?       → ir.actions.server + binding_model_id
Button opens related records?    → Return action dict from Python method
```

---

## Code Examples

### Example 1: URL Action

```xml
<record id="action_open_website" model="ir.actions.act_url">
    <field name="name">Visit Website</field>
    <field name="url">https://www.mycompany.com</field>
    <field name="target">new</field>   <!-- 'new' = new tab, 'self' = same tab -->
</record>
```

### Example 2: Scheduled Action (Cron Job)

```xml
<!-- data/cron.xml (must be in __manifest__ 'data') -->
<record id="ir_cron_my_daily_task" model="ir.cron">
    <field name="name">My Module: Daily Cleanup</field>
    <field name="model_id" ref="model_my_module_model"/>
    <field name="state">code</field>
    <field name="code">model.action_daily_cleanup()</field>
    <field name="interval_number">1</field>
    <field name="interval_type">days</field>
    <field name="numbercall">-1</field>   <!-- -1 = runs forever -->
    <field name="active">True</field>
    <field name="user_id" ref="base.user_root"/>
</record>
```

```python
# models/my_model.py — method called by cron
@api.model
def action_daily_cleanup(self):
    """Remove records older than 30 days. Called by ir.cron."""
    cutoff = fields.Datetime.now() - timedelta(days=30)
    old_records = self.search([
        ('create_date', '<', cutoff),
        ('state', '=', 'cancelled'),
    ])
    old_records.unlink()
```

### Example 3: Client Action (OWL Component)

```xml
<record id="action_my_dashboard" model="ir.actions.client">
    <field name="name">My Dashboard</field>
    <field name="tag">my_module.MyDashboard</field>
</record>
```

```js
// static/src/components/MyDashboard.js
import { registry } from "@web/core/registry";
import { Component } from "@odoo/owl";

class MyDashboard extends Component {
  static template = "my_module.MyDashboard";
}

registry.category("actions").add("my_module.MyDashboard", MyDashboard);
```

### Example 4: Window Action With Filters and Context

```xml
<record id="action_overdue_tasks" model="ir.actions.act_window">
    <field name="name">Overdue Tasks</field>
    <field name="res_model">project.task</field>
    <field name="view_mode">list,form</field>
    <!-- Pre-filter: only show overdue -->
    <field name="domain">[('deadline', '&lt;', context_today().strftime('%Y-%m-%d')), ('state', '!=', 'done')]</field>
    <!-- Pre-fill fields in new records created from this view -->
    <field name="context">{'default_state': 'in_progress'}</field>
</record>
```

---

## Commands

```bash
# Upgrade to load new action/menu XML
./odoo-bin -d mydb -u my_module

# Check action exists in DB (shell)
./odoo-bin shell -d mydb -e "print(env.ref('my_module.action_my_model_list'))"

# Trigger a cron manually (shell)
./odoo-bin shell -d mydb -e "env.ref('my_module.ir_cron_my_daily_task').method_direct_trigger()"
```

---

## Resources

- **Templates**: See [assets/](assets/) for XML templates for each action type
- **Documentation**: See [references/](references/) for the official Odoo 17.0 Actions reference

---
name: odoo-mixins
description: >
  Guide to Odoo 17.0 built-in mixins and abstract classes: mail.thread,
  mail.activity.mixin, portal.mixin, rating.mixin, sequence.mixin (ir.sequence),
  website.published.mixin, utm.mixin, and avatar.mixin. Explains what each adds
  and how to correctly inherit and configure them.
  Trigger: When you need to add chatter/messaging, activities, sequence numbers,
  ratings, portal access, website publishing, or UTM tracking to a model.
license: Apache-2.0
metadata:
  author: AlessandroRLM
  version: "1.0"
---

## When to Use

Use this skill when:

- Adding a chatter (message thread) to a model
- Enabling activity scheduling on a model (reminders, calls, emails)
- Auto-generating sequence numbers (SO001, INV-2024-001, etc.)
- Allowing customers to rate records (helpdesk, services)
- Giving portal users read access to specific records
- Publishing records on the website
- Tracking marketing UTM source/medium/campaign on records

---

## Critical Patterns

### Pattern 1: Mail Thread + Activities (most common combo)

```python
from odoo import fields, models


class HelpDeskTicket(models.Model):
    _name = 'helpdesk.ticket'
    _description = 'Help Desk Ticket'
    _inherit = [
        'mail.thread',           # adds chatter: message log, followers, subtypes
        'mail.activity.mixin',   # adds Activities tab: schedule calls, tasks, etc.
    ]

    name = fields.Char(required=True, tracking=True)   # tracking=True logs changes in chatter
    state = fields.Selection([
        ('new', 'New'),
        ('in_progress', 'In Progress'),
        ('done', 'Done'),
    ], default='new', tracking=True)   # state changes logged automatically

    # mail.thread adds these automatically (no need to define):
    # message_ids, message_follower_ids, activity_ids, activity_state
```

```xml
<!-- Add chatter widget to the form view -->
<form>
    <sheet>
        <!-- your fields -->
    </sheet>
    <!-- Chatter: message thread + activities -->
    <div class="mail-thread" groups="base.group_user"/>
    <chatter/>
</form>
```

### Pattern 2: Sequence Mixin (auto-generated reference numbers)

```python
class PurchaseOrder(models.Model):
    _name = 'my.purchase.order'
    _description = 'Purchase Order'
    # No built-in sequence mixin in Odoo — use ir.sequence directly

    name = fields.Char(
        string='Reference',
        required=True,
        copy=False,
        readonly=True,
        default=lambda self: self.env['ir.sequence'].next_by_code('my.purchase.order'),
    )
```

```xml
<!-- data/sequence.xml -->
<odoo>
    <record id="sequence_my_purchase_order" model="ir.sequence">
        <field name="name">My Purchase Order</field>
        <field name="code">my.purchase.order</field>
        <field name="prefix">PO/%(year)s/</field>
        <field name="padding">5</field>          <!-- PO/2024/00001 -->
        <field name="company_dependent">True</field>
    </record>
</odoo>
```

---

## Decision Tree

```
Need message log / followers?       → _inherit = ['mail.thread']
Need activities (calls, tasks)?     → _inherit = ['mail.thread', 'mail.activity.mixin']
Need auto-incrementing reference?   → ir.sequence + next_by_code() in default
Need customer ratings (1-5 stars)?  → _inherit = ['rating.mixin']
Need portal customer access?        → _inherit = ['portal.mixin']
Need website published toggle?      → _inherit = ['website.published.mixin']
Need UTM tracking?                  → _inherit = ['utm.mixin']
Need avatar/image on model?         → _inherit = ['avatar.mixin']
```

---

## Code Examples

### Example 1: Rating Mixin (star ratings)

```python
class ServiceContract(models.Model):
    _name = 'service.contract'
    _description = 'Service Contract'
    _inherit = ['mail.thread', 'rating.mixin']

    name = fields.Char(required=True)
    partner_id = fields.Many2one('res.partner', required=True)
```

```python
# Send a rating request to the customer
def action_request_rating(self):
    """Send rating email to customer."""
    for rec in self:
        rec.rating_send_request(
            template=self.env.ref('my_module.rating_request_email_template'),
            lang=rec.partner_id.lang,
            force_send=True,
        )
```

### Example 2: Portal Mixin (share records with portal users)

```python
class ProjectTask(models.Model):
    _name = 'project.task'
    _inherit = ['portal.mixin', 'mail.thread']

    def _compute_access_url(self):
        """Define the URL where portal users can view this record."""
        for task in self:
            task.access_url = f'/my/task/{task.id}'
```

```xml
<!-- Give portal user read access in ir.model.access.csv -->
access_project_task_portal,project.task portal,model_project_task,base.group_portal,1,0,0,0
```

### Example 3: Website Published Mixin

```python
class BlogPost(models.Model):
    _name = 'blog.post'
    _inherit = ['website.published.mixin', 'mail.thread']

    name = fields.Char(required=True)
    content = fields.Html()

    # website.published.mixin adds:
    # is_published: Boolean  (toggle on/off)
    # website_published: Boolean (alias)
    # website_url: Char (must implement _compute_website_url)

    def _compute_website_url(self):
        for post in self:
            post.website_url = f'/blog/{post.id}'
```

### Example 4: tracking=True Field-Level Change Log

```python
class SaleOrder(models.Model):
    _name = 'sale.order'
    _inherit = ['mail.thread']

    # Each change to these fields creates a log message in the chatter
    state = fields.Selection([...], tracking=True)
    amount_total = fields.Monetary(tracking=True)
    user_id = fields.Many2one('res.users', tracking=True)

    # Custom tracking priority (higher = more prominent in chatter)
    partner_id = fields.Many2one('res.partner', tracking=10)
```

### Example 5: Sending Messages Programmatically

```python
def action_confirm(self):
    """Confirm the order and notify the salesperson."""
    self.write({'state': 'sale'})
    for order in self:
        # Post internal note (not visible to customer)
        order.message_post(
            body=f"Order confirmed by {self.env.user.name}.",
            message_type='comment',
            subtype_xmlid='mail.mt_note',
        )
        # Send email to followers
        order.message_post(
            body="Your order has been confirmed!",
            message_type='email',
            subtype_xmlid='mail.mt_comment',
            partner_ids=[order.partner_id.id],
        )
```

---

## Commands

```bash
# Upgrade module after adding _inherit
./odoo-bin -d mydb -u my_module

# Check chatter subtypes registered for a model (shell)
./odoo-bin shell -d mydb -e "print(env['mail.message.subtype'].search([('res_model','=','my.module.model')]).mapped('name'))"

# Test rating request manually (shell)
./odoo-bin shell -d mydb -e "env['my.module.model'].browse(1).rating_send_request(...)"
```

---

## Resources

- **Templates**: See [assets/](assets/) for model templates with each mixin pre-configured
- **Documentation**: See [references/](references/) for the official Odoo 17.0 Mixins and useful classes reference

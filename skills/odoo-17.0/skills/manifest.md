---
name: odoo-manifest
description: >
  Complete guide for writing correct Odoo 17.0 __manifest__.py files.
  Covers all required and optional keys, dependency resolution, asset bundles,
  and common mistakes that prevent module installation.
  Trigger: When creating a new Odoo module, adding a dependency, registering
  a new XML/CSV/JS file, or debugging "module cannot be installed" errors.
license: Apache-2.0
metadata:
  author: AlessandroRLM
  version: "1.0"
---

## When to Use

Use this skill when:

- Creating a new Odoo 17.0 custom module from scratch
- Adding new XML views, CSV data, or JS assets that need to be registered
- Adding or fixing module dependencies that prevent installation
- Upgrading a module from a previous Odoo version to 17.0

---

## Critical Patterns

### Pattern 1: Minimal Valid Manifest

```python
# __manifest__.py
{
    'name': 'My Module',
    'version': '17.0.1.0.0',
    'category': 'Uncategorized',
    'summary': 'One-line description of the module.',
    'description': """
        Extended description (RST format supported).
    """,
    'author': 'Your Company',
    'website': 'https://yourcompany.com',
    'license': 'LGPL-3',

    # Dependencies — MUST be correct or install fails
    'depends': ['base'],

    # All data files loaded in listed order
    'data': [
        'security/ir.model.access.csv',
        'security/security_groups.xml',
        'views/my_model_views.xml',
        'views/menus.xml',
        'data/my_module_data.xml',
    ],

    # Demo data only loaded in demo mode
    'demo': [
        'demo/demo_data.xml',
    ],

    'installable': True,
    'application': False,   # True only for top-level apps (e.g. Sales, Inventory)
    'auto_install': False,  # True only for glue modules
}
```

### Pattern 2: Module With JS Assets (OWL Components)

```python
{
    'name': 'My Module With Frontend',
    'version': '17.0.1.0.0',
    'depends': ['web', 'base'],
    'data': [
        'security/ir.model.access.csv',
        'views/my_model_views.xml',
    ],
    'assets': {
        # Main web client bundle — for most OWL components
        'web.assets_backend': [
            'my_module/static/src/components/MyComponent.js',
            'my_module/static/src/components/MyComponent.xml',
            'my_module/static/src/components/my_component.scss',
        ],
        # Public/portal pages bundle
        'web.assets_frontend': [
            'my_module/static/src/js/portal_script.js',
        ],
        # QUnit test bundle
        'web.qunit_suite_tests': [
            'my_module/static/tests/my_component_test.js',
        ],
    },
    'installable': True,
    'application': False,
    'auto_install': False,
}
```

---

## Decision Tree

```
New module?               → Start with Pattern 1 (minimal manifest)
Has OWL components?       → Add 'assets' key using Pattern 2
Has new models?           → Add security/ir.model.access.csv to 'data' FIRST
Extends another module?   → Add that module to 'depends'
Module won't install?     → Check: missing dependency / file not in 'data' / CSV syntax error
Top-level app?            → Set 'application': True
Auto-install glue?        → Set 'auto_install': True, list both parents in 'depends'
```

---

## Code Examples

### Example 1: Version Number Convention

```python
# Format: {odoo_version}.{major}.{minor}.{patch}
# Odoo 17, first release, no patches:
'version': '17.0.1.0.0',

# Odoo 17, second feature release:
'version': '17.0.2.0.0',

# Odoo 17, second feature, first patch:
'version': '17.0.2.1.0',
```

### Example 2: Common Dependency Chains

```python
# Module extending sale orders
'depends': ['sale_management'],

# Module extending accounting
'depends': ['account'],

# Module extending both HR and Payroll
'depends': ['hr', 'hr_payroll'],

# Module only needing base ORM (no UI)
'depends': ['base'],

# Module adding to the website/eCommerce
'depends': ['website', 'website_sale'],
```

### Example 3: External / Community Module Dependency

```python
# OCA modules follow same pattern — just use their technical name
'depends': ['base', 'partner_firstname'],
```

### Example 4: File Load Order Rules

```python
'data': [
    # 1. Security ALWAYS first — views may reference groups
    'security/ir.model.access.csv',
    'security/security_groups.xml',

    # 2. Base data / configuration
    'data/config_data.xml',

    # 3. Views (may reference menus or actions)
    'views/my_model_views.xml',

    # 4. Menus last (reference actions defined in views)
    'views/menus.xml',
],
```

---

## Commands

```bash
# Install a module
./odoo-bin -d mydb -i my_module

# Upgrade a module (picks up manifest/data changes)
./odoo-bin -d mydb -u my_module

# Check for manifest errors without starting server
python -c "import ast; ast.literal_eval(open('__manifest__.py').read())"

# List all installed modules
./odoo-bin shell -d mydb -e "print(env['ir.module.module'].search([('state','=','installed')]).mapped('name'))"
```

---

## Resources

- **Templates**: See [assets/](assets/) for a ready-to-copy `__manifest__.py` template
- **Documentation**: See [references/](references/) for the official Odoo 17.0 Module Manifest reference

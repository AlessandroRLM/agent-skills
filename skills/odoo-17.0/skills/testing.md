---
name: odoo-testing
description: >
  Complete testing guide for Odoo 17.0. Covers Python unit tests (TransactionCase,
  SavepointCase), integration/tour tests, JavaScript QUnit tests, test tags,
  common test helpers, and how to run tests.
  Trigger: When writing tests for any new model, method, wizard, controller,
  or OWL component in Odoo 17.0. Always triggered after generating new code.
license: Apache-2.0
metadata:
  author: AlessandroRLM
  version: "1.0"
---

## When to Use

Use this skill when:

- Writing unit tests for new models, methods, or constraints
- Testing wizards (TransientModel) and their results
- Writing integration tests that simulate full user workflows
- Testing OWL components or JS logic with QUnit
- Setting up test data with `setUpClass` for fast, shared fixtures
- Running tests in CI/CD or checking coverage

---

## Critical Patterns

### Pattern 1: Standard Python Unit Test (TransactionCase)

```python
# tests/test_project_task.py
from odoo.tests import TransactionCase, tagged


@tagged('post_install', '-at_install')   # run after install, not during
class TestProjectTask(TransactionCase):

    @classmethod
    def setUpClass(cls):
        """Create shared test data once for all tests in this class."""
        super().setUpClass()
        cls.project = cls.env['project.project'].create({
            'name': 'Test Project',
        })
        cls.user = cls.env['res.users'].create({
            'name': 'Test User',
            'login': 'test_user@example.com',
            'groups_id': [(4, cls.env.ref('base.group_user').id)],
        })

    def test_task_creation(self):
        """Task can be created with required fields."""
        task = self.env['project.task'].create({
            'name': 'My Task',
            'project_id': self.project.id,
        })
        self.assertEqual(task.state, 'draft')
        self.assertEqual(task.project_id, self.project)

    def test_action_set_done(self):
        """action_set_done transitions task to done state."""
        task = self.env['project.task'].create({
            'name': 'Task to Close',
            'project_id': self.project.id,
        })
        task.action_set_done()
        self.assertEqual(task.state, 'done')
        self.assertTrue(task.is_closed)

    def test_cannot_delete_done_task(self):
        """Deleting a done task raises UserError."""
        task = self.env['project.task'].create({
            'name': 'Done Task',
            'project_id': self.project.id,
            'state': 'done',
        })
        with self.assertRaises(Exception):   # UserError
            task.unlink()

    def test_deadline_constraint(self):
        """Deadline in the past raises ValidationError."""
        from odoo.exceptions import ValidationError
        from datetime import date, timedelta
        with self.assertRaises(ValidationError):
            self.env['project.task'].create({
                'name': 'Past Deadline',
                'project_id': self.project.id,
                'deadline': date.today() - timedelta(days=1),
            })

    def test_days_remaining_computed(self):
        """days_remaining is computed correctly from deadline."""
        from datetime import date, timedelta
        future = date.today() + timedelta(days=5)
        task = self.env['project.task'].create({
            'name': 'Future Task',
            'project_id': self.project.id,
            'deadline': future,
        })
        self.assertEqual(task.days_remaining, 5)
```

### Pattern 2: Wizard Test (TransientModel)

```python
def test_wizard_creates_refund(self):
    """Wizard creates a credit note for the selected invoice."""
    invoice = self._create_posted_invoice()
    wizard = self.env['my.refund.wizard'].with_context(
        active_ids=[invoice.id],
        active_model='account.move',
    ).create({'reason': 'Test refund'})
    result = wizard.action_create_refund()
    # Verify action returned navigates to credit note
    self.assertEqual(result['res_model'], 'account.move')
    credit_note = self.env['account.move'].browse(result['res_id'])
    self.assertEqual(credit_note.move_type, 'out_refund')
```

### Pattern 3: HTTP Controller Test

```python
from odoo.tests import HttpCase, tagged

@tagged('post_install', '-at_install')
class TestMyController(HttpCase):

    def test_route_returns_200(self):
        """Public route is accessible without login."""
        response = self.url_open('/my-module/public-page')
        self.assertEqual(response.status_code, 200)

    def test_authenticated_route(self):
        """Protected route requires login."""
        self.authenticate('admin', 'admin')
        response = self.url_open('/my-module/private-page')
        self.assertEqual(response.status_code, 200)
```

---

## Decision Tree

```
Testing Python model logic?      → TransactionCase
Testing wizards?                 → TransactionCase (create wizard + call method)
Testing HTTP routes?             → HttpCase
Testing full UI tour?            → HttpCase + self.start_tour()
Testing JS component logic?      → QUnit test in static/tests/
Shared expensive test data?      → setUpClass (class-level, faster)
Test needs full isolation?       → setUp (instance-level, recreated per test)
Test should run post-install?    → @tagged('post_install', '-at_install')
Test is slow integration test?   → @tagged('post_install', '-at_install')
```

---

## Code Examples

### Example 1: Test Tags and When to Use Them

```python
from odoo.tests import tagged

@tagged('post_install', '-at_install')   # MOST COMMON: after install, skip during install
class MyTest(TransactionCase): ...

@tagged('at_install')                    # Run during module install (rare)
class MyInstallTest(TransactionCase): ...

@tagged('-standard', 'my_custom_tag')   # Custom tag, excluded from standard suite
class MySlowTest(TransactionCase): ...
```

### Example 2: UI Tour (JavaScript integration test)

```python
# tests/test_ui.py
from odoo.tests import HttpCase, tagged

@tagged('post_install', '-at_install')
class TestMyModuleTour(HttpCase):

    def test_create_task_tour(self):
        """User can create a task through the UI."""
        self.start_tour(
            '/web',
            'my_module_create_task_tour',   # matches tour name in JS
            login='admin',
        )
```

```js
// static/tests/tours/my_module_create_task_tour.js
import { registry } from "@web/core/registry";

registry.category("web_tour.tours").add("my_module_create_task_tour", {
  test: true,
  steps: () => [
    {
      trigger: '.o_menu_brand:contains("My Module")',
      content: "Open My Module app",
      run: "click",
    },
    {
      trigger: ".o_list_button_add",
      content: "Click New",
      run: "click",
    },
    {
      trigger: '.o_field_widget[name="name"] input',
      content: "Fill task name",
      run: "fill Test Task",
    },
    {
      trigger: ".o_form_button_save",
      content: "Save",
      run: "click",
    },
    {
      trigger: '.o_field_widget[name="name"]:contains("Test Task")',
      content: "Verify task was saved",
    },
  ],
});
```

### Example 3: QUnit JS Unit Test

```js
// static/tests/my_component_test.js
import { mount } from "@odoo/owl";
import { MyComponent } from "@my_module/components/MyComponent";
import { makeTestEnv } from "@web/../tests/helpers/mock_env";
import { getFixture } from "@web/../tests/helpers/utils";

QUnit.module("my_module.MyComponent", () => {
  QUnit.test("renders with default props", async (assert) => {
    const env = await makeTestEnv();
    const target = getFixture();
    await mount(MyComponent, target, { env, props: { title: "Hello" } });
    assert.containsOnce(target, ".my-component");
    assert.strictEqual(
      target.querySelector(".my-component-title").textContent,
      "Hello",
    );
  });
});
```

---

## Commands

```bash
# Run all tests for a module
./odoo-bin test -d mydb -i my_module

# Run tests matching a tag
./odoo-bin test -d mydb --test-tags my_module,post_install

# Run a specific test class
./odoo-bin test -d mydb --test-tags /my_module.TestProjectTask

# Run a specific test method
./odoo-bin test -d mydb --test-tags /my_module.TestProjectTask.test_task_creation

# Run JS tests (requires test server running)
./odoo-bin --test-enable --stop-after-init -d mydb -u my_module

# Run tour test
./odoo-bin test -d mydb --test-tags /my_module.TestMyModuleTour.test_create_task_tour
```

---

## Resources

- **Templates**: See [assets/](assets/) for TransactionCase, HttpCase, and QUnit test file templates
- **Documentation**: See [references/](references/) for the official Odoo 17.0 Testing reference

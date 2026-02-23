# agent-skills

Custom agent skills for AI coding assistants. Install with the `npx skills` CLI.

## Install

```bash
# Install all skills
npx skills add AlessandroRLM/agent-skills #Just odoo-17 for now

# Install only the Odoo 17.0 skill
npx skills add AlessandroRLM/agent-skills --skill odoo-17.0

# Install to a specific agent
npx skills add AlessandroRLM/agent-skills --skill odoo-17.0 -a claude-code
```

## Available Skills

| Skill       | Description                                                                 |
| ----------- | --------------------------------------------------------------------------- |
| `odoo-17.0` | Full-stack Odoo 17.0 development — ORM, views, actions, testing, and mixins |

## Skills Structure

```
skills/
└── odoo-17.0/
    ├── SKILL.md              ← skill entry point (loaded by npx skills)
    ├── AGENTS.md             ← full compiled guide with all rules expanded
    ├── odoo-manifest.md      ← __manifest__.py patterns
    ├── odoo-orm-models.md    ← model types, _inherit, class anatomy
    ├── odoo-orm-fields.md    ← all field types and options
    ├── odoo-orm-decorators.md← @api decorators with gotchas
    ├── odoo-orm-recordsets.md← domains, search, write, env, context
    ├── odoo-actions.md       ← window, server, URL, client actions, cron
    ├── odoo-performance.md   ← N+1, batch ops, indexing, raw SQL
    ├── odoo-testing.md       ← unit tests, tours, QUnit
    └── odoo-mixins.md        ← mail.thread, sequences, portal, ratings
```

## License

Apache-2.0

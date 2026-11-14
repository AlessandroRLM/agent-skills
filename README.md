# agent-skills

Custom agent skills for AI coding assistants. Install with the `npx skills` CLI.

## Install

```bash
# Install all skills
npx skills add AlessandroRLM/agent-skills

# Install only the Odoo 17.0 skill
npx skills add AlessandroRLM/agent-skills --skill odoo-17.0

# Install to a specific agent
npx skills add AlessandroRLM/agent-skills --skill odoo-17.0 -a claude-code
```

## Available Skills

| Skill       | Description                                                                 |
| ----------- | --------------------------------------------------------------------------- |
| `odoo-17.0` | Full-stack Odoo 17.0 development — ORM, views, actions, testing, and mixins |
| `writing-data-metadata` | Data governance: write and review metadata (descriptions, glossary terms) a data consumer can use alone |

## Skills Structure

```
skills/
├── data-governance/          ← category folder; skills install flat by name
│   └── writing-data-metadata/
│       ├── SKILL.md
│       ├── references/       ← definition rules, fundamentals, examples
│       └── evals/evals.json  ← test prompts with and without the skill
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

The skills in this repository are licensed under the [Apache License 2.0](LICENSE).

Some skills reference third-party works (books, standards, vendor documentation). Their rules are written in our own words; short quotations keep their attribution. Quoted material belongs to its copyright holders and is not covered by this license. Each skill lists its sources in `references/sources.md`.

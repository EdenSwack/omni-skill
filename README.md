# Omni Analytics Skill for Claude Code

A Claude Code skill that makes Claude an expert in [Omni](https://omnianalytics.com) — the modern BI platform. When active, Claude can help you design, build, review, and improve Omni models, topics, dashboards, and queries using best practices from the official Omni documentation.

---

## What This Skill Does

When you invoke `/omni`, Claude gains deep knowledge of:

- **Topics** — how to design, structure, and curate topics; all topic parameters; security and access control
- **Views, Dimensions & Measures** — YAML syntax, key parameters, patterns like filtered measures, compound metrics, and symmetric aggregates
- **Joins** — relationship types, fan-out prevention, best practices
- **SQL in Omni** — SQL tabs vs model-based queries, templated filters, Omni SQL functions
- **Development workflows** — workbook modeling, promoting to shared model, branch mode, git integration
- **Performance** — caching policies, aggregate awareness
- **Access control** — row-level security, column masking, access grants
- **AI optimization** — how to structure topics and fields for better Omni AI query generation
- **MCP integration** — using Omni's MCP server with Claude Code
- **Dashboard best practices** — KPI design, drill paths, layout

---

## Installation

### Option 1: Install via Claude Code CLI

```bash
claude skill install https://github.com/<your-username>/omni-skill
```

### Option 2: Add as a marketplace in your Claude Code settings

Add to `~/.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "omni-skill": {
      "source": {
        "source": "github",
        "repo": "<your-username>/omni-skill"
      }
    }
  }
}
```

Then install the skill from the marketplace in Claude Code.

---

## Usage

In any Claude Code session, type:

```
/omni
```

Claude will activate the skill and be ready to help with Omni modeling tasks.

### Example prompts

```
/omni help me design a topic for enterprise usage analytics
/omni review this measure definition for fan-out risk
/omni what's the difference between always_where_sql and default_filters
/omni write a filtered measure that only counts enterprise customers
/omni how do I promote a workbook field to the shared model
```

---

## What's Covered

| Area | Details |
|---|---|
| Topics | Parameters, YAML syntax, security, AI optimization, best practices |
| Views | sql_table_name, derived tables, access filters |
| Dimensions | All parameters, date timeframes, access grants, masking |
| Measures | aggregate_type, filtered measures, compound metrics, symmetric aggregates |
| Joins | Relationship types, fan-out prevention |
| SQL | SQL tabs, templated filters, Omni functions |
| Workflows | Workbook → shared model promotion, branch mode, git |
| Performance | Caching, aggregate awareness |
| Access control | Row-level, column-level, topic-level security |
| AI | ai_context, sample_queries, ai_fields, synonyms |
| Dashboards | KPI tiles, drill paths, color schemes, scheduling |

---

## Requirements

- [Claude Code](https://claude.ai/code) installed
- An Omni account (for actually running queries — not required for modeling help)

---

## Contributing

PRs welcome. If Omni releases new features or changes the YAML spec, open an issue or submit a PR updating `skills/omni/SKILL.md`.

---

## License

MIT

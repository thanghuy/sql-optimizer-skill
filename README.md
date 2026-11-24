# sql-optimizer — an AI skill for SQL performance across 10 databases

[![Release](https://img.shields.io/github/v/release/thanghuy/sql-optimizer-skill)](https://github.com/thanghuy/sql-optimizer-skill/releases/latest)
[![License: MIT](https://img.shields.io/github/license/thanghuy/sql-optimizer-skill)](LICENSE)

An open **Agent Skill** (`SKILL.md` + `references/`) that plugs into Claude Code, Claude.ai / Claude Desktop, and any other agent that can read Markdown instructions (Cursor, Codex, Gemini CLI, Copilot, …).

Supported engines: PostgreSQL, MySQL, MariaDB, SQL Server, Oracle, SQLite, IBM Db2, Snowflake, BigQuery, ClickHouse.

## What the skill does

| Step | Action | Why |
|---|---|---|
| 0 | Ask for the engine and its **exact version** | Docs and generated syntax must match the version (e.g. `EXPLAIN (GENERIC_PLAN)` only exists from PostgreSQL 16) |
| 1 | Read the official docs for that version | Avoids advice based on outdated knowledge |
| 2 | Ask for table structure and existing indexes | Extending an existing index is often better than adding a new one |
| 3 | Ask for data volume, cardinality and skew | Selectivity decides whether an index is used at all |
| 4 | Ask for traffic: QPS, read/write ratio, SLO | Weighs read gains against the write cost of every index |
| 5 | **Review the query** → *Issues found* table (location, problem, severity, why it is slow, fix) + rewritten query | Many slow queries are not missing an index — they make the index unusable |
| 6 | Design indexes with the **ESR rule** (Equality → Sort → Range) | Seek on E, read rows already in S order, filter on R — no sort step |
| 7 | Ask for the real query plan, analyze, iterate | Proves the change with before/after numbers |

For Snowflake, BigQuery and ClickHouse (no B-tree indexes), ESR is translated into partition / clustering / sort-key order.

## Repository layout

```
skills/sql-optimizer/                # the skill itself (this is what you install)
├── SKILL.md                         # main workflow
├── assets/intake-template.md        # one-message questionnaire for the user
└── references/
    ├── query-antipatterns.md        # query anti-patterns, "invisible" causes (parameter sniffing, bind types,
    │                                #   collations, statistics, FKs) and things that look bad but are fine
    ├── esr-rule.md                  # ESR, exceptions, covering indexes, tie-breakers, checklist before DROP INDEX
    ├── query-plan-analysis.md       # reading plans, red flags, before/after comparison
    └── databases/<db>.md ×10        # version-specific doc links, commands for schema/index/stats/traffic/plan

sql-optimizer.skill                  # packaged skill (zip) for Claude.ai / Claude Desktop upload
evals/                               # 16 test cases (not part of the installed skill)
sql-optimizer-workspace/             # results of 4 benchmark rounds + static HTML review pages
```

## Installation

### With npx (Claude Code, Cursor, Codex and other agents)

Uses the open-source [`skills`](https://github.com/vercel-labs/skills) CLI (requires Node.js 22.20+):

```bash
# Install into the current project for Claude Code (.claude/skills/)
npx skills add thanghuy/sql-optimizer-skill -a claude-code

# Install globally for Claude Code (~/.claude/skills/), no prompts
npx skills add thanghuy/sql-optimizer-skill -g -a claude-code -y

# See what the repo contains without installing
npx skills add thanghuy/sql-optimizer-skill --list
```

Leave out `-a claude-code` to choose interactively among all supported agents. Update later with `npx skills update`, remove with `npx skills remove`. The CLI sends anonymous usage telemetry; set `DISABLE_TELEMETRY=1` to turn it off.

### Manually with git (Claude Code)

```bash
git clone https://github.com/thanghuy/sql-optimizer-skill.git
cp -r sql-optimizer-skill/skills/sql-optimizer ~/.claude/skills/
```
For a single project, copy it into `<project>/.claude/skills/` instead.

### Claude.ai / Claude Desktop

Settings → Capabilities → Skills → Upload, then choose `sql-optimizer.skill` (also attached to the [latest release](https://github.com/thanghuy/sql-optimizer-skill/releases/latest)).

### Other agents without npx

Cursor, Codex, Gemini CLI, Copilot, …: copy `skills/sql-optimizer` into your repo and add this to `AGENTS.md` or your rules file:
```
For any SQL performance, slow query, query review, index or EXPLAIN task, follow skills/sql-optimizer/SKILL.md
and read the referenced files it points to.
```

## Benchmark results (4 rounds, with skill vs. without skill)

| Round | Focus | With skill | Without skill |
|---|---|---|---|
| 1 | Index design, 4 databases | 32/33 | 32/33 |
| 2 | Stricter assertions + version handling (6 cases) | **52/52** | 46/52 |
| 3 | Planted query anti-patterns + a clean query | 42/42 | 42/42 |
| 4 | Hard cases: parameter sniffing, bind types, collations, FK-backing indexes, correlated statistics | **40/40** | 36/40 |

The recorded runs in `sql-optimizer-workspace/` are kept exactly as produced. Two MySQL cases were originally asked in Vietnamese (to test that the skill replies in the user's language), so those recorded prompts and responses are in Vietnamese; the prompts in `evals/` are now in English.

The base model already knows databases well. The measurable value of the skill is **process discipline**: it asks for traffic and data volume, tags features with their minimum version, keeps the index consistent with the final `ORDER BY` (including tie-breakers), reports issues in a table with explicit severity, and sanity-checks numbers. The trade-off is roughly 45–55 s and 20–30k extra tokens per run.

## Example

> This query takes 3 s on MySQL 8.0, and `orders` has 50 million rows:
> `SELECT * FROM orders WHERE customer_id = 42 AND created_at > '2026-01-01' ORDER BY total DESC LIMIT 20`

The agent reviews the query right away, asks for the table structure, indexes, data volume and traffic, proposes an ESR index (or E-R-S depending on selectivity), and then asks for `EXPLAIN ANALYZE` to verify.

## License

[MIT](LICENSE) — free to use, modify and redistribute, including commercially, as long as the copyright notice is kept.

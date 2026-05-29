# Contributing to Odoo Knowledge Base

## How to Add a New Entry

1. **Choose the right category folder:**
   - `setup/` — Installation, configuration, environment, dependencies
   - `orm/` — Models, fields, ORM methods, computed fields, constraints
   - `views/` — XML views, OWL components, QWeb, JS widgets
   - `security/` — Access rights, record rules, groups, multi-company
   - `performance/` — Query optimization, caching, indexing, profiling
   - `deployment/` — Server config, Nginx, SSL, Docker, production
   - `upgrade/` — Version migration, data migration, breaking changes
   - `misc/` — Anything that doesn't fit above

2. **Use the template:**
   - Copy `TEMPLATE.md` into the appropriate category folder
   - Rename it with a descriptive kebab-case name (e.g., `lxml-build-failure-python314.md`)

3. **Fill in ALL fields:**
   - Especially: Odoo Versions, Severity, Tags, Last Verified
   - The more specific, the more useful

4. **Update the index:**
   - Add your entry to the table in `README.md`
   - Keep entries sorted by category

## Naming Convention

- Use `kebab-case` for file names
- Be descriptive but concise
- Include the key technology/component in the name

**Good:** `lxml-build-failure-python314.md`, `computed-field-stored-performance.md`
**Bad:** `fix1.md`, `issue.md`, `temp-solution.md`

## When to Update an Existing Entry

- You found an **additional pitfall** or edge case
- The solution **changed** in a newer Odoo version
- You found a **better/simpler** solution
- The **verification steps** need updating

When updating, always update the `Last Verified` date.

## For AI Agents (Antigravity / Gemini)

When working on a task:
1. **FIRST** — Read `README.md` to scan for relevant entries
2. **SECOND** — Read any matching entry files before writing code
3. **THIRD** — After solving a problem:
   - If it matches an existing entry → Update it if needed
   - If it's a new problem → Create a new entry + update README.md index
4. **ALWAYS** — Follow the `TEMPLATE.md` format exactly

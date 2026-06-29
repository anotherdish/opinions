---
name: quintessential-opinion
description: Research a software framework's most widely-agreed best practices (official docs, popular guides, style guides, blog posts) and turn them into an opinionated Claude Code plugin of skills + rules. Use when asked to "build a quintessential <framework> plugin", "create opinionated skills for <framework>", "encode best practices for <framework>", or to add/update a framework plugin in this marketplace (e.g. Next.js, Payload, Django, Rails, SwiftUI, Flutter). Pairs with the generate-skills skill.
---

# Quintessential Opinion

Research how the community *actually* recommends building with a framework, distill
the consensus into a small set of design principles, and ship them as a Claude Code
plugin (skills + rules) in this marketplace. The output is one plugin folder under a
category dir, registered in `marketplace.json`.

**"Quintessential" = the dogma everyone converges on**, not an exhaustive manual.
Capture the conventions that shape *how you structure and build* an app. Skip
edge-cases, contested opinions, and anything the official docs already make obvious.

## Inputs

1. **Framework** (required) — the target (e.g. "Next.js 16", "Django 5", "SwiftUI").
   Ask for the major version if it matters; frameworks move fast.
2. **Category** — where the plugin lives (`web/`, `mobile/`, `backend/`, …). Infer it,
   confirm if ambiguous.
3. **Any user opinions** — conventions the user wants enforced. These override sources.

## Workflow

1. **Research the consensus.** Match the tool to the question; fetch in parallel:
   - **Official docs → Context7 MCP** (`resolve-library-id` → `query-docs`). The
     authoritative source for current APIs, recommended patterns, and version changes.
     Always pin the major version.
   - **Popular guides, style guides, "best practices" posts, framework starters/
     templates → `WebSearch` then `WebFetch`.** Search for "<framework> best practices",
     "<framework> project structure", "<framework> style guide", and the official
     starter template. These reveal what the community converges on beyond the docs.
   - Look for **agreement across sources**. A practice that the docs, the popular guides,
     *and* the canonical starter all recommend is dogma. One blog's hot take is not.

2. **Distill the dogma.** From the research, write down the non-negotiables: the stack,
   the project structure, the naming/coding conventions, the security/perf defaults, the
   anti-patterns. Cut anything that doesn't change how someone designs the app. Note
   version-specific gotchas (renamed APIs, removed flags) — those earn their place.

3. **Propose the skill breakdown.** Decide how skills split — **one cohesive topic per
   skill**, heavy workflows (scaffolding a component, deploy) get their own. Most plugins
   need one always-on "conventions/dogma" skill plus a handful of focused ones. Present
   the list (`name` + one-line purpose) and **get approval before writing files.**

4. **Scaffold (or locate) the plugin.** Create `<category>/<framework-kebab>/`:
   ```
   <category>/<plugin-name>/
   ├── .claude-plugin/plugin.json   # name, description, version "0.1.0", author, keywords
   ├── README.md                    # what it is, install, skills table, the overarching dogma
   └── skills/<skill-name>/SKILL.md # one folder per skill
   ```
   If updating an existing plugin, read every current skill first and edit in place —
   don't duplicate. `plugin.json.name` must equal the marketplace plugin name.

5. **Write each skill** (see conventions below). Ground every rule in the research; don't
   pad with generic advice the sources don't back.

6. **Register in `marketplace.json`.** Add a `plugins[]` entry: `name` (matches
   `plugin.json`), `source: "./<category>/<plugin-name>"`, a description that front-loads
   the framework and lists what's covered, `category`, and `keywords`.

7. **Report.** List the plugin path, each skill + the sources that back it, and any
   research that was thin or failed to fetch.

## SKILL.md conventions (match this repo — be concise and direct)

Good skills carry the **minimum content needed to convey the design principles**. Every
line should change a decision. If a sentence restates an obvious default or pads for
completeness, cut it.

- **Frontmatter is only `name` + `description`.** `name` is kebab-case (prefixed with the
  framework, e.g. `nextjs-`) and matches the folder. `description` is the routing signal:
  front-load the domain, then enumerate triggers ("Use when…", file names, task verbs).
- **Body: lead with one line stating what the skill governs**, then opinionated rules.
  Prefer an explicit **Dogma / non-negotiables** list near the top, an **Anti-patterns**
  list near the bottom, and **Cross-references** to sibling skills at the end.
- **State the rule, give the reason only when non-obvious, show the canonical snippet.**
  Imperative voice. Use tables for option matrices, fenced code for templates.
- **Pin the version** and tell Claude to verify fast-moving specifics against the live
  docs (Context7) rather than trusting the skill's snapshot.
- One always-on conventions skill should restate the stack and the overarching dogma so
  the plugin works even when only that skill loads.

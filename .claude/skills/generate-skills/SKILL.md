---
name: generate-skills
description: Generate a set of Claude Code skills from documented sources. Reads a metadata.md file (in a target skills directory) that lists sources — websites, PDFs, local files, libraries — fetches and consumes them, folds in any extra user-provided content, then writes one or more skills/<name>/SKILL.md folders following this repo's conventions. Use when asked to "generate skills", "build skills from these docs/sources", "turn this content into skills", or to scaffold a new skill set from a metadata.md.
---

# Generate Skills from Sources

Turn a `metadata.md` manifest of sources into a set of well-formed Claude Code skills. The output is **plain `SKILL.md` folders** (frontmatter + body) written into a user-specified target directory 

Your job is to create opinionated skills from the information gathered

# Example `metadata.md`

## Inputs

1. **Target directory** — where `metadata.md` is read from and where `skills/` are written. Ask the user for it if not given. Generated skills land in `<target>/skills/<skill-name>/SKILL.md`.
2. **`metadata.md`** — the source manifest in `<target>/metadata.md`. Its format is documented in the template (`meta/skills/generate-skills/metadata.md`).
3. **Additional user content** — anything the user pastes into the conversation or points at beyond what `metadata.md` lists. Treat it as a first-class source.

# Workflow

1. **Locate the manifest.** Read `<target>/metadata.md`. If it's missing, copy the template from this skill's folder into the target dir, tell the user, and stop so they can fill it in. If it exists but key sections are empty, ask before guessing.

2. **Parse the manifest.** Extract: the skill-set theme/domain, the naming prefix, each source (type + location + what to extract), any opinions/guidance, and a desired skill breakdown if the user supplied one.

3. **Fetch every source.** Match the tool to the source type:
   - **URL / website** → `WebFetch`. Follow obviously-relevant linked pages when a source says "and its subpages".
   - **Local PDF / file** → `Read` (PDFs via the `pages` param; chunk large ones).
   - **Library / framework / SDK / CLI** → **Context7 MCP**: `resolve-library-id` then `query-docs` with the source's focus as the query. Prefer this over web search for library docs.
   - **Named but no precise URL** → `WebSearch` first to locate the source, then `WebFetch` the result.

   Fetch sources in parallel where independent. If a fetch fails, note it and continue; report failures at the end rather than silently dropping a source.

4. **Fold in additional content.** Merge anything the user provided directly with the fetched material. User-provided opinions and corrections override what a source says.

5. **Propose the skill breakdown.** From the consumed content, decide how many skills and where the boundaries fall (one cohesive topic per skill; split heavy workflows into their own skill). Present the proposed list — `name` + one-line purpose each — and **get the user's approval before writing files.** Skip this only if `metadata.md` already pins down an explicit breakdown.

6. **Generate each skill.** For every approved skill, write `<target>/skills/<name>/SKILL.md`:
   - Frontmatter with `name` (kebab-case, prefixed per the manifest) and a `description` that says **what it covers and exactly when to load it** (trigger phrases, file types, tasks) — see the conventions below.
   - A body grounded in the consumed sources: opinionated rules, concrete steps, and code/templates where the content supports them. Don't pad with generic advice the sources don't back.
   - Cross-reference sibling skills by name when a task spans several ("Pairs with the `<other>` skill").

7. **Report.** List what was written, which sources backed each skill, and any sources that failed to fetch or were thin.

## SKILL.md conventions (match the rest of this repo)

- **Frontmatter is only `name` + `description`.** No other keys.
- `name` is kebab-case and matches the folder name exactly.
- `description` is the routing signal — it must front-load the domain, then enumerate triggers ("Use when…", file names, task verbs). Models load the skill based on this text, so be specific, not vague.
- Body is Markdown: lead with a one-line statement of what the skill governs, then opinionated rules / steps / templates. Use tables for option matrices, fenced code blocks for templates.
- One folder per skill: `skills/<name>/SKILL.md`. Bundle supporting files (scripts, reference docs) in the same folder only when a skill genuinely needs them.
- Write in the imperative, opinionated voice of the existing skills — state the rule, give the reason when it's non-obvious, show the canonical snippet.

# Skill Generation Manifest

> Template for the `generate-skills` skill. Copy this file into your **target directory**
> (the place where the generated `skills/` folder should be written), fill it in, then
> ask Claude to "generate skills from metadata.md in <target dir>".
> Delete the explanatory blockquotes once you've filled a section in.

## Skill set

> One short paragraph: what domain do these skills cover, and who/what are they for?

- **Theme:** <e.g. "Building accessible design-system components with Radix + Tailwind">
- **Name prefix:** <e.g. `designsys-` — prepended to every generated skill name. Leave blank for none.>
- **Output:** `skills/` in this directory (one folder per skill).

## Sources

> List every source to consume. One bullet per source. For each, give the **type**, the
> **location**, and a note on **what to extract** (which sections matter, what to ignore).
> Types: `url` (website), `pdf` (local file), `file` (local doc), `library` (fetched via Context7), `search` (name only — Claude will locate it).

- **type:** url
  **location:** https://example.com/docs/getting-started
  **extract:** Setup, configuration, and the API reference; ignore the marketing/pricing pages.

- **type:** library
  **location:** next.js
  **extract:** App Router data fetching and caching APIs (current version).

- **type:** pdf
  **location:** ./refs/spec.pdf
  **extract:** Sections 3–5 (the component contract); skip the appendices.

- **type:** search
  **location:** "WAI-ARIA Authoring Practices combobox pattern"
  **extract:** Keyboard interaction and ARIA role requirements.

## Additional content

> Anything not captured by a source above: pasted notes, internal conventions, opinions
> the generated skills must encode, things to always do or never do. Leave blank if none.

-

## Desired skill breakdown (optional)

> If you already know how you want the skills split, list them here as `name — purpose`.
> Leave this empty to let Claude propose a breakdown from the consumed content (it will
> ask for your approval before writing any files).

-

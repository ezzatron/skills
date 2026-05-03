# Google developer documentation style guide → skills

## General conventions

- Use **sentence case** for all headings in produced skill files — capitalize
  only the first word and proper nouns.
- All skill files go under a `skills/` directory (not `.agents/skills/`)
  relative to the current working directory.

## Goal

Exhaustively read the Google developer documentation style guide and condense
it into a set of dense, machine-optimized reference skills that an AI agent can
use when writing or reviewing technical documentation.

## Starting point

Start at <https://developers.google.com/style>. This is the landing page of the
style guide. It contains a navigation structure linking to every article in the
guide.

## Process

1. **Discover** — Fetch the landing page. Identify every article linked from
   the navigation/table of contents. Use `explore` agents to quickly gather the
   full list of article URLs. Stay within `developers.google.com/style/*` — do
   not follow links outside this path.

2. **Read and condense** — Dispatch `general-purpose` subagents in parallel to
   read each article (or group of closely related articles) IN FULL via
   `web_fetch`. Each subagent must read the complete content — do not summarize
   from headings or partial fetches. Then condense the content into a dense
   reference document. Use editorial judgment to group closely related
   micro-articles into a single document (e.g., colons + semicolons + commas →
   `punctuation.md`) when doing so produces better density. The goal is fewer,
   denser files — not a 1:1 mirror of the site structure.

3. **Format** — Each document must be:
   - Dense reference: rules, tables, examples. No fluff, no persuasion, no
     lengthy rationale. A single sentence of rationale is acceptable only when
     it disambiguates an edge case.
   - Concrete: include "do / don't" examples for every rule where the original
     article provides them.
   - Complete: every rule, exception, and nuance from the original article must
     be captured. If content was lost during condensation, the condensation
     failed.

4. **Word list special case** — The word list article
   (`developers.google.com/style/word-list`) is large reference data. Produce
   it as a JSON file (e.g., `word-list.json`), not Markdown. Structure it so
   agents can use `jq` or similar tooling to search entries. Include every
   entry from the original.

5. **Create skill files** — Write each document to disk as a skill:
   - Parent skill: `skills/google-developer-style-guide/SKILL.md`
   - Sub-skills: `skills/google-developer-style-guide-{topic}/SKILL.md`
     (sibling directories, e.g., `google-developer-style-guide-punctuation/`)
   - The word list JSON goes inside the relevant sub-skill directory alongside
     its `SKILL.md`.
   - Every `SKILL.md` must include YAML frontmatter:
     ```yaml
     ---
     name: google-developer-style-guide-punctuation
     description: >-
       Punctuation rules from the Google developer documentation style guide.
       Covers commas, colons, semicolons, dashes, parentheses, and more.
     ---
     ```
   - The parent `SKILL.md` must list every sub-skill by name and give a 1–2
     sentence description of what each covers. It acts as an index.

6. **Verify** — After all files are created, run a verification pass:
   - Confirm every `SKILL.md` has valid YAML frontmatter with `name` and
     `description`.
   - Confirm JSON files parse correctly (`cat file.json | python3 -c
     "import json,sys; json.load(sys.stdin); print('OK')"` or equivalent).
   - Flag any skill file under 500 bytes as suspiciously short — the subagent
     may have truncated content.
   - Report a summary of all created files with byte counts.

## Constraints

- Do not use any prior knowledge of the Google style guide. Read everything
  from the source.
- Follow links within `developers.google.com/style/*` as needed to fully
  understand each topic. Do not follow links outside this path.
- No depth limit on link following within the allowed path, but don't re-read
  pages you've already processed.
- If `web_fetch` truncates a page, use `start_index` pagination to read the
  remainder. Every page must be read in full.

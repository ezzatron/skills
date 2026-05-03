# Vale style authoring docs → skills

## General conventions

- Use **sentence case** for all headings in produced skill files — capitalize
  only the first word and proper nouns.
- All skill files go under a `skills/` directory (not `.agents/skills/`)
  relative to the current working directory.

## Goal

Exhaustively read the Vale documentation on style authoring and condense it
into a set of dense, machine-optimized reference skills that an AI agent can
use when creating or modifying Vale styles (collections of `.yml` rule files).

## Starting point

Start at <https://vale.sh/docs/styles>. This is the main page for
understanding Vale's style/extension system. From here, discover all related
pages needed to fully understand Vale style authoring.

## Process

1. **Discover** — Fetch the starting page. Identify all linked pages that are
   necessary to understand Vale style authoring: check types (existence,
   substitution, etc.), scopes, actions/fixers, regex, configuration,
   vocabularies, formatting/markup support, and anything else referenced. Use
   `explore` agents to gather the full list of relevant URLs. Stay within
   `vale.sh/docs/*`.

2. **Read and condense** — Dispatch `general-purpose` subagents in parallel to
   read each page (or group of closely related pages) IN FULL via `web_fetch`.
   Each subagent must read the complete content. Then condense into a dense
   reference document. Group related pages where it produces better density
   (e.g., all check types might be one document, or split into 2–3 if the
   content is substantial enough).

3. **Format** — Same rules as the Google style guide prompt:
   - Dense reference with rules, tables, and concrete YAML examples.
   - Every field, option, and behavior documented on the original page must be
     captured.
   - Include complete, working YAML examples for every check type.

4. **Create skill files** — Write each document to disk as a skill:
   - Parent skill: `skills/vale-style-authoring/SKILL.md`
   - Sub-skills: `skills/vale-style-authoring-{topic}/SKILL.md`
     (sibling directories, e.g., `vale-style-authoring-checks/`)
   - Every `SKILL.md` must include YAML frontmatter with `name` and
     `description`.
   - The parent `SKILL.md` must list every sub-skill by name and describe what
     each covers.

5. **Verify** — After all files are created, run a verification pass:
   - Confirm every `SKILL.md` has valid YAML frontmatter with `name` and
     `description`.
   - Flag any skill file under 500 bytes as suspiciously short — the subagent
     may have truncated content.
   - Report a summary of all created files with byte counts.

## Constraints

- Do not use any prior knowledge of Vale. Read everything from the source.
- Follow links within `vale.sh/docs/*` as needed. Do not follow links outside
  this path.
- If `web_fetch` truncates a page, paginate to read the remainder.
- Every page must be read in full.

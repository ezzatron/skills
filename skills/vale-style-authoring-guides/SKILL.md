---
name: vale-style-authoring-guides
description: "Reference guides for Vale: regex syntax, glob patterns, Hunspell dictionaries, and LSP integration"
---

# Vale guides

## Regex in Vale

Vale uses the [`regexp2`](https://github.com/dlclark/regexp2) library, which extends Go's standard [`regexp`](https://pkg.go.dev/regexp/syntax) package with lookaheads, lookbehinds, and lazy quantifiers.

**Exception:** `script`-based rules are limited to standard Go regex syntax and do not support the extended `regexp2` features.

### Assertion constructs

| Construct | Meaning |
|---|---|
| `(?=re)` | Positive lookahead |
| `(?!re)` | Negative lookahead |
| `(?<=re)` | Positive lookbehind |
| `(?<!re)` | Negative lookbehind |

For full base syntax see the [Go docs](https://pkg.go.dev/regexp/syntax). For extended syntax see the [`regexp2` README](https://github.com/dlclark/regexp2?tab=readme-ov-file#compare-regexp-and-regexp2).

### Regex in YAML

All regex must be wrapped in quotes to prevent YAML from interpreting special characters.

| Quote style | Behavior | When to use |
|---|---|---|
| Single (`'`) | YAML interprets nothing except `''` (escaped single quote). **Preferred for most cases.** | Default choice for regex. |
| Double (`"`) | YAML interprets escape sequences (`\n`, `\t`, etc.); backslashes must be doubled. | When you need YAML escape sequences. |

To include a literal single quote inside single-quoted regex, double it:

```yaml
tokens:
  - '([A-Z]\w+)([A-Z]\w+)''s'
```

### Word boundaries

`\b` matches the position between a word character and a non-word character. The `existence` and `substitution` rule extensions automatically add `\b` to the beginning and end of each token. Disable this by setting `nonword: true`:

```yaml
extends: existence
message: "Consider removing '%s'"
nonword: true
tokens:
  - some token
```

### Scoping and markup

For markup-based rules Vale converts documents to HTML and applies its scoping system before running rules. Regex matching therefore operates on the scoped/converted content, not raw markup. To match against the original, unprocessed document use `scope: raw`:

```yaml
extends: existence
message: "Consider removing '%s'"
scope: raw
tokens:
  - some token
```

### Debugging regex

[Vale Studio](https://studio.vale.sh/) provides a rule editor integrated with [regex101](https://regex101.com) for inspecting compiled patterns and testing against sample text.

---

## Glob patterns in Vale

Globs match file paths and are used both in `.vale.ini` section headers and the `--glob` CLI flag.

### Syntax

| Pattern | Meaning |
|---|---|
| `/` | Path segment separator |
| `*` | Zero or more characters within a single path segment |
| `?` | Exactly one character within a single path segment |
| `**` | Zero or more directories (recursive) |
| `[]` | Character range (e.g., `[a-z]`) |
| `[!...]` | Negated character range |
| `{}` | Set of alternative patterns (e.g., `{md,txt}`) |

### Negation with `--glob`

Prefix the entire pattern with `!` to exclude matching files:

```bash
vale --glob='!**/*.{md,py}' path/to/files
```

### Precedence

The `--glob` flag is evaluated **first**, then `.vale.ini` section patterns are applied to the remaining files.

Example `.vale.ini`:

```ini
StylesPath = styles

[*.md]
BasedOnStyles = Vale
```

Directory:

```
cases/test/
├── a.md
├── b/
│   └── b.md
└── c.md
```

Running `vale --glob='!**/b/*' .` excludes `b/b.md` because `--glob` takes precedence over `.vale.ini`.

---

## Hunspell dictionaries in Vale

Vale does **not** use Hunspell directly and does not require it to be installed. It uses a pure-Go package that parses Hunspell-compatible dictionaries, supporting a growing subset of Hunspell features.

### Dictionary file pair

| File | Extension | Purpose |
|---|---|---|
| Affix file | `.aff` | Morphological rules: prefixes, suffixes, language-specific grammar governing word formation. |
| Dictionary file | `.dic` | Root words with associated affix codes specifying valid transformations. |

Both files must share the same base name (e.g., `en_US.aff` and `en_US.dic`).

### Minimal example

**Dictionary file (`custom.dic`):**

```
1
software/M
```

- `1` — word count.
- `software/M` — root word `software` with affix code `M`.

**Affix file (`custom.aff`):**

```
SET UTF-8

SFX M Y 1
SFX M   0     's         .
```

| Line | Meaning |
|---|---|
| `SET UTF-8` | Character encoding. |
| `SFX M Y 1` | Suffix rule for code `M`; `Y` = cross-productible; `1` = one rule follows. |
| `SFX M 0 's .` | Append `'s` to the base word; `0` = remove nothing from base; `.` = no conditions. |

Result: accepts `software` and `software's`; rejects `softwares`, `softwaring`, etc.

### Where to find Hunspell dictionaries

- [`wooorm/dictionaries`](https://github.com/wooorm/dictionaries)
- [`LibreOffice/dictionaries`](https://github.com/LibreOffice/dictionaries)
- [Firefox language packs](https://addons.mozilla.org/en-US/firefox/language-tools)
- [OpenOffice extensions](https://extensions.openoffice.org/en/search?f%5B0%5D=field_project_tags%3A157)

For thorough Hunspell documentation see the [official repository](https://github.com/hunspell/hunspell). A well-documented Python port is available at [spylls](https://github.com/zverok/spylls).

---

## Vale language server (LSP)

The Vale Language Server (`vale-ls`) wraps a local Vale installation via the [Language Server Protocol](https://microsoft.github.io/language-server-protocol/), providing autocomplete, diagnostics, hover popups, and more.

Download releases from [GitHub](https://github.com/errata-ai/vale-ls/releases).

### Configuration (`initializationParams`)

| Parameter | Default | Description |
|---|---|---|
| `installVale` | `true` | Automatically install/update Vale into a `vale_bin` folder alongside `vale-ls`. If `false`, `vale` must be on the user's `$PATH`. |
| `filter` | `None` | An [output filter](https://vale.sh/manual/filter/) applied when calling Vale. |
| `configPath` | `None` | Absolute path to a `.vale.ini` file used as default configuration. |
| `syncOnStartup` | `true` | Runs `vale sync` when the server starts. |

### Available editor/integration support

| Integration | Link |
|---|---|
| VS Code | [vale-vscode](https://github.com/chrischinchilla/vale-vscode) (LSP) |
| Sublime Text | [LSP-vale-ls](https://packagecontrol.io/packages/LSP-vale-ls) (LSP) |
| Neovim | [ALE](https://github.com/dense-analysis/ale) (LSP) |
| Zed | [zed-vale](https://github.com/koozz/zed-vale) (LSP) |
| Emacs | [flymake-vale](https://github.com/tpeacock19/flymake-vale) |
| JetBrains | [Vale CLI plugin](https://plugins.jetbrains.com/plugin/19613-vale-cli/docs) |
| Obsidian | [obsidian-vale](https://github.com/ChrisChinchilla/obsidian-vale) |
| Oxygen XML | [Vale linter add-on](https://www.oxygenxml.com/doc/versions/23.1/ug-editor/topics/vale-linter-addon.html) |
| Qt Creator | [Setting up Vale](https://wiki.qt.io/Setting_Up_Vale) |
| GitHub Actions | [vale-action](https://github.com/errata-ai/vale-action) |
| CircleCI | [vale orb](https://circleci.com/developer/orbs/orb/circleci/vale) |
| Git hooks | [pre-commit integration](https://vale.sh/docs/integrations/pre-commit) |
| Laravel | [laravel-prose-linter](https://github.com/beyondcode/laravel-prose-linter) |

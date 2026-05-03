---
name: vale-style-authoring-config
description:
  "Complete reference for Vale configuration: .vale.ini structure, all
  configuration keys, vocabularies, and packages"
---

# Vale configuration reference

## .vale.ini file structure

Vale reads configuration from a `.vale.ini` (or `_vale.ini`) file. The file is
INI-formatted with three section types: **core settings** (global/top-level),
**format associations** (`[formats]`), and **format-specific settings** (under
glob patterns like `[*]` or `[*.md]`).

```ini
# Core settings (global section)
StylesPath = styles
MinAlertLevel = suggestion
Packages = Google, write-good
Vocab = Base

[formats]
# Fall back to Markdown when mdx2vast is not installed
mdx = md

[*]
# Applies to all files
BasedOnStyles = Vale

[*.{md,txt}]
# Applies only to .md and .txt files — overrides [*]
BasedOnStyles = Vale, write-good
```

## Search process

Vale searches for `.vale.ini` or `_vale.ini` starting from the directory where
`vale` was invoked, walking up the file tree. If none is found, it falls back to
the global configuration file.

### Global configuration locations

| OS      | Location                                           |
| ------- | -------------------------------------------------- |
| Windows | `%LOCALAPPDATA%\vale\.vale.ini`                    |
| macOS   | `$HOME/Library/Application Support/vale/.vale.ini` |
| Unix    | `$XDG_CONFIG_HOME/vale/.vale.ini`                  |

The global config is **always loaded in addition to** (not instead of) any
project config. It is read **after** other sources. Multi-valued settings (e.g.
`BasedOnStyles`) are **merged**; single-valued settings (e.g. `MinAlertLevel`)
are **overridden** by the global config.

Use `vale ls-dirs` to see exact locations on your system.

## Core settings

These appear at the top of the file (the global section) and apply to Vale
itself, not to a specific file format.

| Key              | Type       | Default                  | Description                                                                                                        |
| ---------------- | ---------- | ------------------------ | ------------------------------------------------------------------------------------------------------------------ |
| `StylesPath`     | `string`   | OS-dependent (see below) | Path to all Vale-related resources (styles, config). May be absolute or relative to the `.vale.ini` location.      |
| `Packages`       | `string[]` | _(none)_                 | Comma-separated list of packages to download and install via `vale sync`.                                          |
| `Vocab`          | `string[]` | _(none)_                 | Comma-separated list of vocabulary folder names to load from `<StylesPath>/config/vocabularies/`.                  |
| `MinAlertLevel`  | `enum`     | `suggestion`             | Minimum alert level to report. Values: `suggestion`, `warning`, `error`. Only `error` causes a non-zero exit code. |
| `IgnoredScopes`  | `string[]` | `code, tt`               | Comma-separated list of inline-level HTML tags whose content will not raise alerts, even within active scopes.     |
| `SkippedScopes`  | `string[]` | `script, style, pre`     | Comma-separated list of block-level HTML tags to skip entirely. All content inside is ignored.                     |
| `IgnoredClasses` | `string[]` | _(none)_                 | Comma-separated list of HTML class names to ignore. Applies to both inline- and block-level elements.              |

### StylesPath

Specifies where Vale looks for styles, vocabularies, dictionaries, scripts, and
other resources. If omitted, Vale uses its OS-dependent default:

| OS      | Default StylesPath                              |
| ------- | ----------------------------------------------- |
| Windows | `%LOCALAPPDATA%\vale\styles`                    |
| macOS   | `$HOME/Library/Application Support/vale/styles` |
| Unix    | `$XDG_DATA_HOME/vale/styles`                    |

#### StylesPath internal structure

A `StylesPath` contains style directories and a special `config` directory:

```
<StylesPath>/
├── config/              # Special directory
│   ├── actions/         # Tengo scripts for suggest actions
│   ├── dictionaries/    # Hunspell-compatible spelling dictionaries
│   ├── filters/         # Configuration filters
│   ├── ignore/          # Spelling ignore word lists
│   ├── scripts/         # Tengo scripts for script checks
│   ├── templates/       # Output format templates
│   ├── views/           # View YAML files for structured linting
│   └── vocabularies/    # Project-specific terminology lists
├── write-good/          # A style
└── MyStyle/             # Another style
```

### MinAlertLevel

```ini
MinAlertLevel = suggestion
```

Allowed values: `suggestion` (default), `warning`, `error`. Only `error`-level
alerts produce a non-zero exit code, making this useful for CI.

**CLI override:** `vale --minAlertLevel=warning README.md`

**Per-rule severity override:** You can change a rule's severity in `.vale.ini`:

```ini
[*.md]
BasedOnStyles = Vale
Vale.Spelling = warning
```

### IgnoredScopes

```ini
IgnoredScopes = code, tt
```

Inline-level HTML tags to ignore. Content inside these tags will not raise
alerts, but the surrounding scope remains active. Default: `code, tt`. For
example, inline `` `code` `` in Markdown will not be linted.

### SkippedScopes

```ini
SkippedScopes = script, style, pre
```

Block-level HTML tags to skip entirely. Default: `script, style, pre`. For
example, fenced code blocks in Markdown will not be linted.

### IgnoredClasses

```ini
IgnoredClasses = my-class, another-class
```

HTML class names to ignore. Applies to both inline- and block-level elements.
This is a core (global) setting, not format-specific.

## Format associations

The `[formats]` section maps unknown file extensions to supported ones
(extension-level substitution only — does not add support for new file types):

```ini
[formats]
mdx = md    # Fall back to Markdown when mdx2vast is not installed
mdtext = md # Treat .mdtext files as Markdown
```

## Format-specific settings

Format-specific sections use glob patterns and apply only to matching files.
More specific patterns override less specific ones. `[*]` matches all files.

| Key                 | Type        | Supported formats                              | Description                                                                 |
| ------------------- | ----------- | ---------------------------------------------- | --------------------------------------------------------------------------- |
| `BasedOnStyles`     | `string[]`  | All                                            | Comma-separated list of styles to enable (all rules in each style).         |
| `BlockIgnores`      | `string[]`  | Markdown, reStructuredText, AsciiDoc, Org Mode | Comma-separated regexes for block-level content to ignore.                  |
| `TokenIgnores`      | `string[]`  | Markdown, reStructuredText, AsciiDoc, Org Mode | Comma-separated regexes for inline-level content to ignore.                 |
| `CommentDelimiters` | `string[2]` | All                                            | Custom open/close comment delimiters (replaces `<!-- -->`).                 |
| `Transform`         | `string`    | XML                                            | Path to a version 1.0 XSL Transformation (XSLT) for converting XML to HTML. |

### BasedOnStyles

Enables all rules from the listed styles:

```ini
[*.md]
BasedOnStyles = Vale, MyStyle
```

**Selectively enable a single rule** (without BasedOnStyles):

```ini
[*.md]
Style1.Rule = YES
```

**Selectively disable a rule** within an enabled style:

```ini
[*.md]
BasedOnStyles = Vale, MyStyle
Vale.Spelling = NO
```

### BlockIgnores

Exclude block-level sections by regex. The first capture group must match the
entire block. Only supported in Markdown, reStructuredText, AsciiDoc, and Org
Mode.

```ini
[*.md]
BasedOnStyles = Vale
BlockIgnores = (?s) *({< file [^>]* >}.*?{</ ?file >})
```

### TokenIgnores

Exclude inline-level sections by regex. The first capture group must match the
entire token. Only supported in Markdown, reStructuredText, AsciiDoc, and Org
Mode.

```ini
[*.md]
BasedOnStyles = Vale
TokenIgnores = (\$+[^\n$]+\$+), (:math:`.*`)
```

### CommentDelimiters

Override the standard HTML comment delimiters (`<!-- -->`). Useful for formats
like MDX that don't support HTML comments.

```ini
[formats]
mdx = md  # Fall back to Markdown when mdx2vast is not installed

[*.mdx]
BasedOnStyles = Vale
CommentDelimiters = {/*, */}
```

With custom delimiters set, you can use markup-based configuration to toggle
rules inline:

```mdx
{/* vale off */} This text is ignored by Vale. {/* vale on */}

{/* vale Vale.Redundancy = NO */} This text skips the Redundancy rule.
{/* vale Vale.Redundancy = YES */}
```

### Transform

Specifies a version 1.0 XSLT file for converting XML to HTML before linting.
Path is relative to `StylesPath`.

```ini
[*.xml]
BasedOnStyles = Vale
Transform = docbook-xsl-snapshot/html/docbook.xsl
```

## Vocabularies

Vocabularies maintain custom terminology lists independent of styles. They live
at `<StylesPath>/config/vocabularies/<name>/` and contain two plain-text files:
`accept.txt` and `reject.txt`.

### Vocabulary effects

| File         | Effect                                                                                                                                                                            |
| ------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `accept.txt` | Entries are added to every exception list in all styles listed in `BasedOnStyles`. Entries are also added to the `Vale.Terms` substitution rule, enforcing exact casing/spelling. |
| `reject.txt` | Entries are added to `Vale.Avoid` existence rule, flagging all occurrences as errors.                                                                                             |

Overlap between the two files is rarely needed. Adding `JavaScript` to
`accept.txt` will automatically enforce correct casing — you do not also need
`[Jj]avascript` in `reject.txt`.

### Folder structure

```
<StylesPath>/
└── config/
    └── vocabularies/
        ├── Blog/
        │   ├── accept.txt
        │   └── reject.txt
        └── Marketing/
            ├── accept.txt
            └── reject.txt
```

Reference by folder name:

```ini
StylesPath = styles
Vocab = Blog

[*]
BasedOnStyles = Vale, MyStyle
```

The built-in `Vale` style is required for `Vale.Terms`, `Vale.Avoid`, and
`Vale.Spelling` to work.

### File format

Both `accept.txt` and `reject.txt` are plain-text, one entry per line. Entries
are evaluated as **case-sensitive regular expressions**. Lines starting with `#`
are comments and ignored.

```text
first
[pP]y.*\b
third
```

### Case sensitivity

Vocabulary files are case-aware by default. An entry of `MongoDB` enforces that
exact casing — `mongoDB`, `MongoDb`, etc. will all produce errors.

To make an entry case-insensitive, use a regex:

```text
(?i)MongoDB
[Oo]bservability
```

Alternatively, disable `Vale.Terms` and rely on `Vale.Spelling` for traditional
spell-checking:

```ini
[*.md]
BasedOnStyles = Vale
Vale.Terms = NO
```

### Vocabularies vs. ignore files

| Feature        | Vocabularies               | Ignore files       |
| -------------- | -------------------------- | ------------------ |
| Audience       | Style **users**            | Style **creators** |
| Applies to     | Multiple extension points  | `spelling` only    |
| Regex support  | Yes                        | No                 |
| Built-in rules | `Vale.Terms`, `Vale.Avoid` | None               |

### Rules targeting vocabulary entries

To write a rule that matches against a token that would otherwise be ignored by
a vocabulary, set `vocab: false` in the rule YAML:

```yaml
extends: existence
message: Did you mean '%s'?
vocab: false
tokens:
  - MongoDB
```

## Packages

Packages share, extend, sync, and update Vale configurations. They are `.zip`
files containing a `.vale.ini`, a `StylesPath` folder, or both. Install via
`vale sync`.

### Package sources

The `Packages` key accepts four types of values:

| Source type                    | Example                       |
| ------------------------------ | ----------------------------- |
| Official Package Explorer name | `Google`                      |
| URL to hosted `.zip`           | `https://example.com/pkg.zip` |
| Local path to `.zip` file      | `./packages/MyPkg.zip`        |
| Local path to directory        | `./packages/MyPkg/`           |

```ini
Packages = Microsoft,
  https://github.com/errata-ai/errata.ai/releases/download/v1.0.0/Test.zip
```

### Package types

**Style-only** — a `.zip` of a single style folder (e.g. `write-good/`). After
`vale sync`, the style is added to the active `StylesPath`.

**Config-only** — a `.zip` containing a single `.vale.ini`. After `vale sync`,
the config is added to `StylesPath/.vale-config/` in load order.

**Complete** — contains both `.vale.ini` and a `styles/` folder:

```
MyPackage/
├── .vale.ini
└── styles/
    ├── MyStyle/
    │   └── MyRule.yml
    └── config/
        ├── dictionaries/
        │   └── MyDic.dic
        ├── scripts/
        │   └── MyScript.tengo
        └── vocabularies/
            └── MyVocab/
                ├── accept.txt
                └── reject.txt
```

The packaged `StylesPath` must be named `styles` inside the archive. It is
merged with the local `StylesPath`.

### Package ordering

Later packages override earlier ones. Local configuration overrides all
packages:

```ini
Packages = pkg1, pkg2
# pkg2 overrides pkg1; local config overrides both
```

### VCS / .gitignore

Add packaged components to `.gitignore`, preserving local assets:

```gitignore
# Ignore StylesPath except local vocabularies
.github/styles/*
!.github/styles/config/
.github/styles/config/*
!.github/styles/config/vocabularies/
.github/styles/config/vocabularies/*
!.github/styles/config/vocabularies/Base
```

## Complete example

```ini
StylesPath = .github/styles
MinAlertLevel = suggestion
IgnoredScopes = code, tt
SkippedScopes = script, style, pre
IgnoredClasses = no-vale

Packages = Microsoft, write-good
Vocab = Base, Marketing

[formats]
mdx = md

[*]
BasedOnStyles = Vale, Microsoft, write-good

[*.md]
BasedOnStyles = Vale, Microsoft, write-good
BlockIgnores = (?s) *({< file [^>]* >}.*?{</ ?file >})
TokenIgnores = (\$+[^\n$]+\$+)
Vale.Spelling = warning

[*.mdx]
BasedOnStyles = Vale, Microsoft
CommentDelimiters = {/*, */}

[*.xml]
BasedOnStyles = Vale
Transform = docbook-xsl-snapshot/html/docbook.xsl
```

## Quick setup

```sh
cd my-project
# Create .vale.ini (or use https://vale.sh/generator)
cat > .vale.ini << 'EOF'
StylesPath = styles
MinAlertLevel = suggestion
Packages = Google

[*]
BasedOnStyles = Vale, Google
EOF

vale sync   # Downloads packages into StylesPath
vale .      # Lints all supported files
```

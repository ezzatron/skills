---
name: vale-style-authoring
description: "Core concepts for Vale style authoring: style structure, scopes, filters, templates, and views"
---

# Vale style authoring reference

## Related skills

| Skill | Covers |
|---|---|
| vale-style-authoring-checks | All 11 check types (extensions): existence, substitution, occurrence, repetition, consistency, conditional, capitalization, metric, spelling, sequence, and script |
| vale-style-authoring-actions | Actions and fixers: edit, remove, replace, and suggest |
| vale-style-authoring-formats | Format handling: Markdown, MDX, HTML, AsciiDoc, reStructuredText, Org, DITA, XML, code comments, and front matter |
| vale-style-authoring-config | Configuration: .vale.ini structure, all configuration keys, vocabularies, and packages |
| vale-style-authoring-guides | Reference guides: regex syntax, glob patterns, Hunspell dictionaries, and LSP integration |

## Styles and directory structure

A **style** is a folder of YAML rule files (`.yml`) stored under a user-specified `StylesPath`. Each subfolder is a distinct style.

```
styles/
├── base/
│   ├── ComplexWords.yml
│   └── SentenceLength.yml
├── blog/
│   └── TechTerms.yml
└── docs/
    └── Branding.yml
```

`base`, `blog`, and `docs` are styles; each `.yml` file inside is a **rule**.

### Built-in `Vale` style

| Rule | Description |
|---|---|
| `Vale.Spelling` | Spell-check using Hunspell-compatible dictionaries in `<StylesPath>/config/dictionaries`. |
| `Vale.Terms` | Enforces the project's accepted vocabulary terms. |
| `Vale.Avoid` | Enforces the project's rejected vocabulary terms. |
| `Vale.Repetition` | Flags repeated words (e.g., "the the"). |

## Rules

Every rule is a YAML file that **extends** a check. The file has a header (common fields) followed by check-specific arguments.

### Header fields

| Field | Required | Default | Description |
|---|---|---|---|
| `extends` | Yes | N/A | Name of the check to extend (e.g., `existence`, `substitution`). |
| `message` | Yes | N/A | Message shown when triggered. Supports `%s` formatting per the extension point. |
| `level` | No | `suggestion` | Severity: `suggestion`, `warning`, or `error`. |
| `scope` | No | `text` | Scope selector(s). See **Scopes** below. |
| `link` | No | N/A | URL providing more info about the rule. |
| `limit` | No | N/A | Max times the rule can trigger per file. |
| `vocab` | No | `true` | Set `false` to disable active vocabularies for this rule. |

### Example rule

```yaml
extends: existence
message: "Don't use end punctuation in headings."
link: https://docs.microsoft.com/en-us/style-guide/punctuation/periods
nonword: true
level: warning
scope: heading
action:
  name: edit
  params:
    - remove
    - '.?!'
tokens:
  - '[a-z0-9][.?!](?:\s|$)'
```

## Checks (extension points)

| Check | Description |
|---|---|
| `existence` | Check for the presence of a specific regex pattern. |
| `substitution` | Replace a regex pattern with a specific string. |
| `occurrence` | Ensure a regex pattern appears a specific number of times. |
| `repetition` | Avoid repeating a regex pattern. |
| `consistency` | Ensure a regex pattern is used consistently. |
| `conditional` | Check for a regex pattern based on a condition. |
| `capitalization` | Ensure a regex pattern is capitalized in a specific way. |
| `metric` | Check readability or other metrics using custom formulas. |
| `spelling` | Spell-check using Hunspell-compatible dictionaries. |
| `sequence` | Ensure a regex pattern is used in a specific order; supports POS tagging. |
| `script` | Run a custom Tengo script. |

## Regex

Vale uses a [superset](https://github.com/dlclark/regexp2) of Go's `regexp/syntax`. In addition to standard syntax, it supports:

- Positive lookahead: `(?=re)`
- Negative lookahead: `(?!re)`
- Positive lookbehind: `(?<=re)`
- Negative lookbehind: `(?<!re)`

## Scopes

Vale is markup-aware. A **scope** is a selector like `paragraph.rst` that targets specific sections of text. Vale classifies files into three types — `markup`, `code`, or `text` — which determines available scopes. Within each type, multiple formats share the same scope names, so rules are cross-format compatible.

### Markup scopes

| Scope | Description |
|---|---|
| `heading` | All `h{1,…}` tags. Append level: `heading.h1`, `heading.h2`, etc. |
| `table.header` | All `th` tags. |
| `table.cell` | All `td` tags. |
| `table.caption` | All `caption` tags. |
| `figure.caption` | All `figcaption` tags. |
| `list` | All `li` tags. |
| `paragraph` | Segments of text separated by two newlines. |
| `sentence` | All sentences. |
| `blockquote` | All `blockquote` tags. |
| `alt` | All alt attributes. |
| `summary` | Body text excluding headings, code spans, code blocks, and table cells. Useful for readability rules. |
| `raw` | Raw unprocessed markup source. Useful for regex rules matching original source. |

Default behavior for markup: rules apply to all non-ignored sections (no scope needed for most rules).

#### Supported markup formats

| Format | Built-in? |
|---|---|
| Markdown | Yes |
| HTML | Yes |
| Org | Yes |
| AsciiDoc | No (third-party) |
| reStructuredText | No (third-party) |
| XML | No (third-party) |
| DITA | No (third-party) |
| MDX | No (third-party) |

### Code scopes

Two scopes available: `comment.line` and `comment.block`.

### Selectors

**Multiple scopes (OR):** Use a YAML array:

```yaml
scope:
  - heading.h1
  - heading.h2
```

**Negation:** Prefix with `~`:

```yaml
scope:
  - ~heading.h2
```

**Chaining (AND with negation):** Use `&`:

```yaml
scope:
  - ~blockquote & ~heading
```

**Format qualifier:** Append format to any scope: `heading.md`, `paragraph.rst`, `comment.line`, `text.html`.

## Filters

The `--filter` CLI option reports an arbitrary subset of your `.vale.ini` configuration. A filter is an [expr-lang](https://expr-lang.org/docs/language-definition) expression targeting rule-definition keys.

### Available filter keys

| Key | Description |
|---|---|
| `.Name` | Full rule name (e.g., `demo.Cap`). |
| `.Level` | Severity string (`error`, `warning`, `suggestion`). |
| `.Scope` | Scope value(s). |
| `.Message` | Rule message. |
| `.Description` | Rule description. |
| `.Extends` | Check name the rule extends. |
| `.Link` | URL associated with the rule. |

### Saving filters

Store reusable filters in `<StylesPath>/config/filters` as `.expr` files:

```
$ vale --filter=headings.expr docs/
```

Where `headings.expr` contains:

```
"heading" in .Scope
```

### Example filter expressions

```
# Filter by level and name
.Level in ["error", "suggestion"] and .Name != "demo.Cap"

# Filter by check type
.Extends == "existence"

# Run only a specific rule
.Name == "demo.Cap"
```

All [expr-lang operators](https://expr-lang.org/docs/language-definition#operators) are supported.

## Templates

Vale supports three built-in output styles (`line`, `JSON`, `CLI`) plus custom templates using Go's `text/template` package. Specify via `--output`:

```
$ vale --output=line README.md
$ vale --output='template.tmpl' somefile.md
```

Custom template files are stored in `<StylesPath>/config/templates`.

### Data structures available to templates

```go
type ProcessedFile struct {
    Alerts []core.Alert
    Path   string
}

type Data struct {
    Files       []ProcessedFile
    LintedTotal int
}
```

`core.Alert` has the same fields as Vale's `--output=JSON` object (including `.Severity`, `.Line`, `.Span`, `.Message`, `.Check`).

### Template functions

| Function | Argument(s) | Description |
|---|---|---|
| `red` | `string` | ANSI red foreground. |
| `blue` | `string` | ANSI blue foreground. |
| `yellow` | `string` | ANSI yellow foreground. |
| `underline` | `string` | ANSI underline. |
| `newTable` | `bool` | Create a tablewriter struct. Bool controls `SetAutoWrapText`. |
| `addRow` | `[]string` | Append a row to a table. |
| `renderTable` | `Table` | Print table to stdout. |
| `jsonEscape` | `string` | Ensure valid JSON string. |

All [Sprig functions](http://masterminds.github.io/sprig/) are also available (e.g., `add1`, `plural`, `toStrings`, `list`, `indent`, `printf`).

### Example: reimplementing default output

```gotemplate
{{- $e := 0 -}}{{- $w := 0 -}}{{- $s := 0 -}}{{- $f := 0 -}}
{{- range .Files}}
{{$table := newTable true}}
{{- $f = add1 $f -}}
{{- .Path | underline | indent 1 -}}
{{- range .Alerts -}}
{{- $error := "" -}}
{{- if eq .Severity "error" -}}
    {{- $error = .Severity | red -}}
    {{- $e = add1 $e  -}}
{{- else if eq .Severity "warning" -}}
    {{- $error = .Severity | yellow -}}
    {{- $w = add1 $w -}}
{{- else -}}
    {{- $error = .Severity | blue -}}
    {{- $s = add1 $s -}}
{{- end}}
{{- $loc := printf "%d:%d" .Line (index .Span 0) -}}
{{- $row := list $loc $error .Message .Check | toStrings -}}
{{- $table = addRow $table $row -}}
{{end -}}
{{- $table = renderTable $table -}}
{{end}}
{{- $e}} {{"errors" | red}}, {{$w}} {{"warnings" | yellow}} and {{$s}} {{"suggestions" | blue}} in {{$f}} {{$f | int | plural "file" "files"}}.
```

### Example: RDJSONL for Reviewdog

```gotemplate
{{- range .Files}}
{{- $path := .Path -}}
{{- range .Alerts -}}
{{- $error := "" -}}
{{- if eq .Severity "error" -}}{{- $error = "ERROR" -}}
{{- else if eq .Severity "warning" -}}{{- $error = "WARNING" -}}
{{- else -}}{{- $error = "INFO" -}}
{{- end}}
{{- $line := printf "%d" .Line -}}
{{- $col := printf "%d" (index .Span 0) -}}
{{- $check := printf "%s" .Check -}}
{{- $message := printf "%s" .Message -}}
{"message": "[{{ $check }}] {{ $message | jsonEscape }}", "location": {"path": "{{ $path }}", "range": {"start": {"line": {{ $line }}, "column": {{ $col }}}}}, "severity": "{{ $error }}"}
{{end -}}
{{end -}}
```

## Views

A **view** is a virtual, filtered perspective of a structured file. It defines transformation steps that extract named scopes, controlling exactly what content is linted. Views are stored in `<StylesPath>/config/views` as YAML files.

### Referencing views in `.vale.ini`

```ini
[*.json]
BasedOnStyles = Vale
View = MyView
```

### View step fields

| Field | Required | Description |
|---|---|---|
| `name` | No | Step name. Without `type`, used as the only scope. With `type`, used as a **metascope** appended to the active scope (e.g., `heading.<name>.md`). |
| `expr` | Yes | Expression selecting data to lint, evaluated by the active engine. |
| `type` | No | Data type of extracted content: `md`, `adoc`, `html`, `rst`, or `org`. |

Top-level field: `engine` — specifies which query engine to use.

### Engines

#### Dasel (JSON, YAML, TOML)

[Dasel](https://github.com/TomWright/dasel) queries structured data using selectors. Works with JSON, YAML, TOML, XML.

```yaml
engine: dasel
scopes:
  - name: title
    expr: info.title
    type: md

  - expr: info.description
    type: md

  - expr: servers.all().description
    type: md
```

Use `all()` to iterate arrays. Example extracting from features array:

```yaml
engine: dasel
scopes:
  - name: feature
    expr: features.all().title
    type: md

  - expr: features.all().description
    type: md
```

The `name: feature` creates a metascope, enabling rules that specifically target the `feature` scope.

#### Tree-sitter (source code)

[Tree-sitter](https://tree-sitter.github.io/tree-sitter/) parses source code in any language. Queries use tree-sitter's pattern-matching syntax.

```yaml
engine: tree-sitter
scopes:
  - name: comment
    expr: (comment)+ @comment

  - expr: |
      ((function_definition
        body: (block . (expression_statement (string) @docstring)))
      (#offset! @docstring 0 3 0 -3))
```

The `#offset!` predicate trims characters from matched nodes (useful for stripping quote delimiters from docstrings).

#### TextFSM (text)

Coming soon.

## Config directory summary

| Path | Purpose |
|---|---|
| `<StylesPath>/config/dictionaries/` | Hunspell-compatible dictionaries for `Vale.Spelling`. |
| `<StylesPath>/config/filters/` | Saved `.expr` filter files. |
| `<StylesPath>/config/templates/` | Custom `.tmpl` output template files. |
| `<StylesPath>/config/views/` | View YAML files for structured file linting. |

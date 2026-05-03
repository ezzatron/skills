---
name: vale-style-authoring-actions
description:
  "Complete reference for Vale actions and fixers: edit, remove, replace, and
  suggest"
---

# Vale actions and fixers

Actions define dynamic fixes for custom Vale rules. They surface in the CLI (as
computed suggestions in output messages) and in LSP-based integrations (as
"Quick Fix" menu items via the Vale Language Server).

An `action` block is placed inside a rule YAML file alongside the rule's other
keys (`extends`, `message`, `tokens`, etc.).

## Actions overview

| Name      | Signature                             | Description                                                                                                                 |
| --------- | ------------------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| `suggest` | `func suggest(match string) []string` | Returns an array of dynamically-computed suggestions (via script or spelling dictionaries).                                 |
| `replace` | `func replace(match string) []string` | Returns an array of static, user-provided replacement strings. Auto-populated by `substitution` and `capitalization` rules. |
| `remove`  | `func remove(match string)`           | Removes the matched text entirely. Takes no parameters.                                                                     |
| `edit`    | `func edit(match string) string`      | Performs an in-place edit on the matched text using a named operation and parameters.                                       |

Actions fall into two categories:

- **Static** — the suggestion is known ahead of time (e.g., `replace` with a
  `substitution` swap map). Vale can pre-generate the output message.
- **Dynamic** — the suggestion depends on a runtime string transformation (e.g.,
  `edit` with a regex, or `suggest` with a Tengo script). The CLI computes the
  suggestion from the matched text at runtime.

Both static and dynamic actions work with the Vale Language Server, which
exposes them as Quick Fixes.

---

## Fixer: `edit`

Performs an in-place edit of the matched text. Requires `params` — an array
whose first element is the operation name and remaining elements are operation
arguments.

### Operations

| Operation    | Params                   | Behavior                                                                                                               | Go equivalent                           |
| ------------ | ------------------------ | ---------------------------------------------------------------------------------------------------------------------- | --------------------------------------- |
| `regex`      | `[regex, pattern, repl]` | Replace all occurrences of `pattern` in the match with `repl`. Supports capture-group back-references (`$1`, `$2`, …). | `pattern.ReplaceAllString(match, repl)` |
| `trim_right` | `[trim_right, string]`   | Trim `string` from the **end** of the matched text.                                                                    | —                                       |
| `trim_left`  | `[trim_left, string]`    | Trim `string` from the **start** of the matched text.                                                                  | —                                       |

### Fields

| Key      | Type   | Required | Description                                                                                                                  |
| -------- | ------ | -------- | ---------------------------------------------------------------------------------------------------------------------------- |
| `name`   | string | yes      | Must be `edit`.                                                                                                              |
| `params` | array  | yes      | First element is the operation (`regex`, `trim_right`, or `trim_left`). Remaining elements are operation-specific arguments. |

### Examples

#### Regex: snake_case to kebab-case

```yaml
extends: existence
message: "'%s' should be '%s'."
level: error
action:
  name: edit
  params:
    - regex
    - '_'
    - '-'
nonword: true
tokens:
  - '\w+_\w+'
```

#### Regex: split PascalCase with hyphen

```yaml
extends: existence
message: "'%s' should be '%s'."
level: warning
action:
  name: edit
  params:
    - regex
    - '([a-z])([A-Z])'
    - '$1-$2'
nonword: true
tokens:
  - '[A-Z][a-zA-Z]*[A-Z][a-zA-Z]*'
```

#### trim_right: remove trailing exclamation mark

```yaml
extends: existence
message: "Don't use exclamation points in text."
nonword: true
action:
  name: edit
  params:
    - trim_right
    - "!"
tokens:
  - '\w+!(?:\s|$)'
```

#### trim_left: remove leading extra space

```yaml
extends: existence
message: "'%s' too many spaces."
level: warning
nonword: true
action:
  name: edit
  params:
    - trim_left
    - " "
tokens:
  - "(?<=[a-z][.!?] ) [A-Z]"
```

---

## Fixer: `remove`

Removes the entire matched text. Takes no parameters.

### Fields

| Key    | Type   | Required | Description       |
| ------ | ------ | -------- | ----------------- |
| `name` | string | yes      | Must be `remove`. |

### Example

```yaml
extends: existence
message: "Don't use an ellipsis in documentation."
nonword: true
action:
  name: remove
tokens:
  - "..."
```

---

## Fixer: `replace`

Returns an array of static replacement suggestions provided via `params`. Rules
that extend `substitution` or `capitalization` auto-populate `params` from their
swap maps, so only `name` is needed.

### Fields

| Key      | Type   | Required | Description                                                                                                           |
| -------- | ------ | -------- | --------------------------------------------------------------------------------------------------------------------- |
| `name`   | string | yes      | Must be `replace`.                                                                                                    |
| `params` | array  | no       | Explicit list of replacement strings. Omit when the rule extends `substitution` or `capitalization` (auto-populated). |

### Examples

#### Explicit params

```yaml
action:
  name: replace
  params:
    - option1
    - option2
```

#### Auto-populated from substitution swap map

```yaml
extends: substitution
message: "Use '%s' instead of '%s'."
level: error
action:
  name: replace
swap:
  Javascript: JavaScript
```

#### Auto-populated from capitalization (name only)

```yaml
action:
  name: replace
```

---

## Fixer: `suggest`

Returns an array of dynamically-computed suggestions. Supports two modes
selected by the first element of `params`: a **Tengo script** or the built-in
**spellings** engine.

### Modes

| Mode      | Params value         | Description                                                                                                                                                                                                 |
| --------- | -------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| script    | `[scriptName.tengo]` | Executes a custom Tengo script that must set a `suggestions` array variable. Scripts live in `<StylesPath>/config/actions/`. The variable `match` is injected by Vale and contains the rule's matched text. |
| spellings | `[spellings]`        | Returns the top 5 spelling suggestions from all active dictionaries, ordered by Levenshtein distance to the matched text.                                                                                   |

### Fields

| Key      | Type   | Required | Description                                                                                |
| -------- | ------ | -------- | ------------------------------------------------------------------------------------------ |
| `name`   | string | yes      | Must be `suggest`.                                                                         |
| `params` | array  | yes      | Single-element array: either a `.tengo` script filename or the literal string `spellings`. |

### Script details

- Language: [Tengo](https://github.com/d5/tengo).
- Location: `<StylesPath>/config/actions/`.
- Input variable: `match` (string) — the rule's matched text, injected by Vale.
- Output variable: `suggestions` (array of strings) — required; Vale reads this
  for the fix list.

### Examples

#### Tengo script: PascalCase to snake_case

Script (`<StylesPath>/config/actions/CamelToSnake.tengo`):

```tengo
text := import("text")

// `match` is provided by Vale and represents the rule's matched text.
// Insert underscore before each uppercase letter that follows a lowercase letter.
made := text.re_replace(`([a-z])([A-Z])`, match, `${1}_${2}`)
// Insert underscore between consecutive uppercase and an uppercase+lowercase pair.
made = text.re_replace(`([A-Z]+)([A-Z][a-z])`, made, `${1}_${2}`)
made = text.to_lower(made)

// `suggestions` is required by Vale and represents the script's output.
suggestions := [made]
```

Rule referencing the script:

```yaml
extends: existence
message: "'%s' should be in snake_case."
nonword: true
level: error
action:
  name: suggest
  params:
    - CamelToSnake.tengo
tokens:
  - '[A-Z]\w+[A-Z]\w+'
```

#### Spelling suggestions

```yaml
action:
  name: suggest
  params:
    - spellings
```

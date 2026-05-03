---
name: vale-style-authoring-checks
description:
  "Complete reference for all Vale check types (extensions): existence,
  substitution, occurrence, repetition, consistency, conditional,
  capitalization, metric, spelling, sequence, and script"
---

# Vale check types reference

Every Vale rule YAML file must declare `extends: <check_type>`. This document
covers all 11 check types, their keys, and complete examples.

## Common keys

These keys are available on every check type (not repeated per-type below):

| Key       | Type     | Description                                                                 |
| --------- | -------- | --------------------------------------------------------------------------- |
| `extends` | `string` | **Required.** The check type name.                                          |
| `message` | `string` | Alert message. Use `%s` for match interpolation.                            |
| `level`   | `string` | `suggestion`, `warning`, or `error`.                                        |
| `scope`   | `string` | The scope to lint (e.g. `heading`, `sentence`, `text`, `raw`, `paragraph`). |
| `link`    | `string` | URL for additional context shown with the alert.                            |

---

## Existence

Flags the **presence** of particular tokens (words, phrases, or regex patterns).

### Keys

| Key          | Type    | Default | Description                                                               |
| ------------ | ------- | ------- | ------------------------------------------------------------------------- |
| `tokens`     | `array` | —       | Strings/regexes compiled into a word-bounded non-capturing group.         |
| `raw`        | `array` | —       | Raw regex fragments concatenated as-is (no word boundaries).              |
| `append`     | `bool`  | `false` | When `true`, appends `raw` to the end of `tokens` (both must be defined). |
| `ignorecase` | `bool`  | `false` | Case-insensitive matching.                                                |
| `nonword`    | `bool`  | `false` | Removes default `\b` word boundaries.                                     |
| `exceptions` | `array` | `[]`    | Strings to ignore even if matched.                                        |
| `vocab`      | `bool`  | `true`  | Set `false` to disable active vocabularies for this rule.                 |
| `action`     | `map`   | —       | Options for correcting matches (see Vale actions docs).                   |

`tokens` entries are compiled into `(?i)(?m)\b(?:token1|token2)\b`. Use `raw`
when you need full regex control; entries are concatenated.

`message` supports one `%s` placeholder replaced with the matched text.

### Example

```yaml
extends: existence
message: "Consider removing '%s'"
level: warning
ignorecase: true
tokens:
  - appears to be
  - arguably
```

### Example with raw

```yaml
extends: existence
message: "Incorrect use of symbols in '%s'."
ignorecase: true
raw:
  - \$\d* ?(?:dollars|usd|us dollars)
```

---

## Substitution

Associates an **observed** string with a **preferred** replacement.

### Keys

| Key          | Type    | Default | Description                                                                                |
| ------------ | ------- | ------- | ------------------------------------------------------------------------------------------ |
| `swap`       | `map`   | —       | `observed: expected` pairs. Keys may be regex; values may reference capture groups (`$1`). |
| `ignorecase` | `bool`  | `false` | Case-insensitive matching.                                                                 |
| `nonword`    | `bool`  | `false` | Removes default `\b` word boundaries.                                                      |
| `capitalize` | `bool`  | `false` | Matches the capitalization of the source token.                                            |
| `exceptions` | `array` | `[]`    | Strings to ignore.                                                                         |
| `vocab`      | `bool`  | `true`  | Set `false` to disable active vocabularies.                                                |
| `action`     | `map`   | —       | Required when using multiple suggestions (pipe-separated values).                          |

`swap` keys can be regexes (e.g. `'(?:give|gave) rise to': lead to`). Capture
groups in keys can be referenced in values (`$1`, `$2`, …).

Multiple suggestions are separated by `|` in the value (e.g.
`masterful: skilled|authoritative|commanding`). When using multiple suggestions,
an `action` with `name: replace` is required.

`message` supports one or two `%s` specifiers: the first is the suggested
replacement, the second is the matched text.

### Example

```yaml
extends: substitution
message: "Consider using '%s' instead of '%s'"
level: warning
ignorecase: false
swap:
  abundance: plenty
  accelerate: speed up
```

### Example with regex keys and capture groups

```yaml
extends: substitution
message: "Consider using '%s' instead of '%s'"
level: warning
swap:
  "(?:give|gave) rise to": lead to
  "within the (.*)?directory": in the $1 directory
```

### Example with multiple suggestions

```yaml
extends: substitution
message: "Consider using %s instead of '%s.'"
level: warning
action:
  name: replace
swap:
  masterful: skilled|authoritative|commanding
```

---

## Occurrence

Enforces a **minimum or maximum count** of a token within a scope.

### Keys

| Key     | Type     | Default | Description                        |
| ------- | -------- | ------- | ---------------------------------- |
| `token` | `string` | —       | The token (string/regex) to count. |
| `max`   | `int`    | —       | Maximum allowed occurrences.       |
| `min`   | `int`    | —       | Minimum required occurrences.      |

`message` supports one `%s` placeholder replaced with the occurrence count.

### Example

```yaml
extends: occurrence
message: "More than 3 commas!"
level: error
scope: sentence
max: 3
token: ","
```

### Example with message interpolation

```yaml
extends: occurrence
message: "Titles should use fewer than 70 characters (found: %s)."
level: warning
scope: heading
max: 70
token: "."
```

---

## Repetition

Flags **repeated consecutive occurrences** of its tokens (e.g. "the the").

### Keys

| Key          | Type    | Default | Description                                             |
| ------------ | ------- | ------- | ------------------------------------------------------- |
| `tokens`     | `array` | —       | Strings/regexes transformed into a non-capturing group. |
| `ignorecase` | `bool`  | `false` | Case-insensitive matching.                              |
| `alpha`      | `bool`  | `false` | Limits matches to alphanumeric tokens.                  |
| `exceptions` | `array` | `[]`    | Strings to ignore.                                      |
| `vocab`      | `bool`  | `true`  | Set `false` to disable active vocabularies.             |

`message` supports one `%s` placeholder replaced with the repeated token.

Vale includes a built-in `Vale.Repetition` rule that catches repeated words even
across markup boundaries.

### Example

```yaml
extends: repetition
message: "'%s' is repeated!"
level: error
alpha: true
tokens:
  - "[^s.!?]+"
```

---

## Consistency

Ensures that only **one** of two variant spellings/forms appears in the document
(e.g. "advisor" vs "adviser").

### Keys

| Key          | Type   | Default | Description                                                   |
| ------------ | ------ | ------- | ------------------------------------------------------------- |
| `either`     | `map`  | —       | `option_a: option_b` pairs; only one of each pair may appear. |
| `ignorecase` | `bool` | `false` | Case-insensitive matching.                                    |
| `nonword`    | `bool` | `false` | Removes default `\b` word boundaries.                         |

`message` supports one `%s` placeholder replaced with the inconsistent term.

### Example

```yaml
extends: consistency
message: "Inconsistent spelling of '%s'."
level: error
ignorecase: true
either:
  advisor: adviser
  centre: center
```

---

## Conditional

Asserts that **if** a pattern (`first`) exists, a corresponding pattern
(`second`) must also exist — i.e. "the existence of `first` implies the
existence of `second`."

### Keys

| Key          | Type     | Default | Description                                                     |
| ------------ | -------- | ------- | --------------------------------------------------------------- |
| `first`      | `string` | —       | Regex for the antecedent (the thing that triggers the check).   |
| `second`     | `string` | —       | Regex for the consequent (the thing that must also be present). |
| `ignorecase` | `bool`   | `false` | Case-insensitive matching.                                      |
| `exceptions` | `array`  | `[]`    | Tokens that won't be flagged.                                   |
| `vocab`      | `bool`   | `true`  | Set `false` to disable active vocabularies.                     |

Capture groups in `first` and `second` are compared. Regex lookarounds can
restrict what is captured for more complex conditional logic.

`message` supports one `%s` placeholder replaced with the unmatched antecedent.

### Example (unexpanded acronyms)

```yaml
extends: conditional
message: "'%s' has no definition"
level: error
scope: text
ignorecase: false
first: '\b([A-Z]{3,5})\b'
second: '(?:\b[A-Z][a-z]+ )+\(([A-Z]{3,5})\)'
exceptions:
  - ABC
  - ADD
```

### Example with lookarounds (unused imports)

```yaml
extends: conditional
message: "'%s' has been imported but not used."
level: error
scope: raw
first: '(?<=import )(\w+)(?= from)'
second: '(?<=<)(\w+)'
```

---

## Capitalization

Checks that text in the specified scope matches a **case style**.

### Keys

| Key          | Type     | Default | Description                                                                    |
| ------------ | -------- | ------- | ------------------------------------------------------------------------------ |
| `match`      | `string` | —       | One of `$title`, `$sentence`, `$lower`, `$upper`, or a custom regex pattern.   |
| `style`      | `string` | `AP`    | `AP` or `Chicago`. Only applies when `match` is `$title`.                      |
| `exceptions` | `array`  | `[]`    | Strings to ignore during checking.                                             |
| `indicators` | `array`  | `[]`    | Suffixes that indicate the next token should be ignored.                       |
| `threshold`  | `float`  | `0.8`   | Minimum proportion of words that must match the case for the sentence to pass. |
| `prefix`     | `string` | —       | Regex for a constant prefix to ignore during case conversion.                  |
| `vocab`      | `bool`   | `true`  | Set `false` to disable active vocabularies.                                    |

**`$title` styles:**

- **AP** (Associated Press): capitalize first and last words; capitalize "to" in
  infinitives; do not capitalize articles, conjunctions, and prepositions of
  three letters or fewer.
- **Chicago** (Chicago Manual of Style): capitalize first and last words; do not
  capitalize articles (a, an, the), coordinating conjunctions (and, but, or,
  for, nor), and prepositions regardless of length.

`message` supports one or two `%s` specifiers: with two, the first is the found
text and the second is the expected form.

### Example

```yaml
extends: capitalization
message: "'%s' should be in title case"
level: warning
scope: heading
match: $title
style: AP
exceptions:
  - ABC
  - add
```

### Example with prefix

```yaml
extends: capitalization
message: "'%s' should be sentence-cased."
scope: heading
match: $sentence
prefix: '^[a-z]\.\s'
```

---

## Metric

Evaluates an **arithmetic formula** over document-level variables and triggers
when a **condition** is met. All metric rules are automatically summary-scoped
(whole document).

### Keys

| Key         | Type     | Default | Description                                                        |
| ----------- | -------- | ------- | ------------------------------------------------------------------ |
| `formula`   | `string` | —       | Arithmetic expression using pre-defined variables and operators.   |
| `condition` | `string` | —       | Binary comparison (e.g. `> 8.0`, `< 1`, `>= 100`, `== 0`, `<= 5`). |

`message` supports one `%s` placeholder replaced with the formula result.

### Variables

| Variable             | Description                                                        |
| -------------------- | ------------------------------------------------------------------ |
| `blockquote`         | Number of `blockquote` tags.                                       |
| `characters`         | Number of characters.                                              |
| `complex_words`      | Polysyllabic words without common suffixes (`es`, `ed`, `ing`, …). |
| `heading.h{n}`       | Number of headings at level _n_ (e.g. `heading.h1`).               |
| `list`               | Number of `ol` and `ul` tags.                                      |
| `long_words`         | Words with more than 6 characters.                                 |
| `paragraphs`         | Number of paragraphs.                                              |
| `polysyllabic_words` | Words with more than 2 syllables.                                  |
| `pre`                | Number of `pre` tags.                                              |
| `sentences`          | Number of sentences.                                               |
| `syllables`          | Number of syllables.                                               |
| `words`              | Number of words.                                                   |

### Operators

| Operator       | Description           |
| -------------- | --------------------- |
| `+`            | Addition              |
| `-`            | Subtraction           |
| `*`            | Multiplication        |
| `/`            | Division              |
| `math.sqrt(x)` | Square root of _x_    |
| `math.abs(x)`  | Absolute value of _x_ |

Condition operators: `>`, `<`, `==`, `>=`, `<=`.

### Example (Flesch–Kincaid)

```yaml
extends: metric
message: "Try to keep the Flesch-Kincaid grade level (%s) below 8."
link: https://en.wikipedia.org/wiki/Flesch%E2%80%93Kincaid_readability_tests
formula: |
  (0.39 * (words / sentences)) + (11.8 * (syllables / words)) - 15.59
condition: "> 8.0"
```

---

## Spelling

Spell checking based on **Hunspell-compatible dictionaries**.

### Keys

| Key            | Type     | Default      | Description                                                                                                      |
| -------------- | -------- | ------------ | ---------------------------------------------------------------------------------------------------------------- |
| `dictionaries` | `array`  | `["en_US"]`  | Dictionary names to load. Vale looks for `<name>.dic` and `<name>.aff` in `<StylesPath>/config/dictionaries`.    |
| `append`       | `bool`   | `false`      | When `true`, adds custom dictionaries after the default Vale dictionary instead of replacing it.                 |
| `dicpath`      | `string` | —            | Directory for `.dic`/`.aff` files. Absolute or relative to `StylesPath`. Can also be set via `DICPATH` env var.  |
| `custom`       | `bool`   | `false`      | When `true`, disables the built-in filters (see below) so only user-defined `filters` apply.                     |
| `filters`      | `array`  | _(built-in)_ | Regex patterns for words to ignore during spell checking.                                                        |
| `ignore`       | `string \| array` | —            | Path(s) (relative to `<StylesPath>/config/ignore`) to file(s) of words to ignore (one per line, case-insensitive). Accepts a single string or an array of strings. |
| `vocab`        | `bool`   | `true`       | Set `false` to disable active vocabularies.                                                                      |

### Built-in filters (active when `custom` is `false`)

| Filter                    | Description                                       |
| ------------------------- | ------------------------------------------------- |
| `[A-Z]{1}[a-z]+[A-Z]+\w+` | Mixed-case words (e.g. "MongoDB").                |
| `[^a-zA-Z_']`             | Words containing non-word tokens (numbers, etc.). |
| `[A-Z]+$`                 | Fully upper-cased words.                          |

Default dictionary: open-source American English (`en_US`).

### Example (defaults)

```yaml
extends: spelling
message: "Did you really mean '%s'?"
level: error
```

### Example with custom dictionaries

```yaml
extends: spelling
message: "'%s' is a typo!"
dictionaries:
  - en_US
  - en_medical
```

### Example with custom filters

```yaml
extends: spelling
message: "Did you really mean '%s'?"
level: error
custom: true
filters:
  - '[pP]y.*\b'
```

### Example with ignore files

```yaml
extends: spelling
message: "Did you really mean '%s'?"
level: error
ignore:
  - ignore1.txt
  - ignore2.txt
```

---

## Sequence

Matches **ordered sequences of NLP tokens** (part-of-speech tags and/or regex
patterns). Designed for grammar-focused rules. All sequence rules are
sentence-scoped.

### Keys

| Key          | Type         | Default | Description                                             |
| ------------ | ------------ | ------- | ------------------------------------------------------- |
| `tokens`     | `[]NLPToken` | —       | Ordered list of token descriptors (see NLPToken below). |
| `ignorecase` | `bool`       | `false` | Case-insensitive matching.                              |

At least one entry must specify a `pattern` — this is the anchor. Vale finds all
instances of the first pattern, then checks that the surrounding tokens match
the rest of the sequence.

### NLPToken fields

| Field     | Type     | Required        | Description                                                           |
| --------- | -------- | --------------- | --------------------------------------------------------------------- |
| `pattern` | `string` | If no `tag`     | Regex to match the token text.                                        |
| `tag`     | `string` | If no `pattern` | Part-of-speech tag. Use `\|` to accept alternatives (e.g. `VB\|VBN`). |
| `negate`  | `bool`   | No              | When `true`, this position must **not** match.                        |
| `skip`    | `int`    | No              | Allow up to _n_ intervening tokens between this and the next entry.   |

`message` supports positional format specifiers: `%[N]s` refers to the _N_-th
token in the sequence.

POS tags follow the Penn Treebank tagset as implemented by `prose/tagging`.

### Example

```yaml
extends: sequence
message: |
  The infinitive '%[4]s' after 'be' requires 'to'.
  Did you mean '%[2]s %[3]s *to* %[4]s'?
tokens:
  - tag: MD
  - pattern: be
  - tag: JJ
  - tag: VB|VBN
```

---

## Script

Executes arbitrary logic via a **Tengo** script (a Go-like scripting language).

### Keys

| Key      | Type     | Default | Description                                                            |
| -------- | -------- | ------- | ---------------------------------------------------------------------- |
| `script` | `string` | —       | Filename of the Tengo script, stored at `<StylesPath>/config/scripts`. |

### Script contract

Inside the script:

1. Import Tengo's `text` module for string/regex utilities.
2. Read the `scope` variable — contains text per the rule's `scope` setting.
3. Populate the `matches` array. Each entry is a map with:
   - `begin` (`int`): start offset in `scope`.
   - `end` (`int`): end offset in `scope`.

### Example rule

```yaml
extends: script
message: "Consider inserting a new section heading at this point."
link: https://tengolang.com
scope: raw
script: MyScript.tengo
```

### Example script (`MyScript.tengo`)

````tengo
text := import("text")

matches := []
p_limit := 3

document := text.re_replace("(?s) *(\n```.*?```\n)", scope, "")

count := 0
for line in text.split(document, "\n") {
    if text.has_prefix(line, "#") {
        count = 0
    } else if count > p_limit {
        start := text.index(scope, line)
        matches = append(matches, {begin: start, end: start + len(line)})
        count = 0
    } else if text.trim_space(line) == "" {
        count += 1
    }
}
````

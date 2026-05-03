---
name: vale-style-authoring-formats
description:
  "Complete reference for Vale format handling: Markdown, MDX, HTML, AsciiDoc,
  reStructuredText, Org, DITA, XML, code comments, and front matter"
---

# Vale format support

## Format overview

| Format           | Support type | External dependency            | Extensions                              |
| ---------------- | ------------ | ------------------------------ | --------------------------------------- |
| Markdown (GFM)   | Built-in     | None                           | `.md`, `.mdown`, `.markdown`, `.markdn` |
| MDX              | External     | `mdx2vast` (npm)               | `.mdx`                                  |
| HTML5            | Built-in     | None                           | `.html`, `.htm`, `.shtml`, `.xhtml`     |
| AsciiDoc         | External     | `asciidoctor`                  | `.adoc`, `.asciidoc`, `.asc`            |
| reStructuredText | External     | `rst2html` (Python `docutils`) | `.rst`, `.rest`                         |
| Org              | Built-in     | None                           | `.org`                                  |
| DITA             | External     | DITA Open Toolkit              | `.dita`                                 |
| XML              | External     | `xsltproc`                     | `.xml`                                  |

## What each format ignores by default

| Format           | Ignored elements                                                                 |
| ---------------- | -------------------------------------------------------------------------------- |
| Markdown         | Indented blocks (4+ spaces), fenced blocks (``` ), code spans (backticks), URLs  |
| MDX              | Fenced blocks, code spans, URLs, JSX expressions/components, ESM imports/exports |
| HTML             | `script`, `style`, `pre`, `code`, `tt` tags; URLs                                |
| AsciiDoc         | Literals and source code, URLs                                                   |
| reStructuredText | Literal blocks, inline literals, URLs                                            |
| Org              | Code blocks, literal examples, code/verbatim strings, URLs                       |
| DITA             | `<codeblock>`, `<tt>`, `<codeph>` elements                                       |

## Comment-based configuration

Most markup formats support inline comment directives to control Vale behavior.
The directives are identical; only the comment syntax differs per format.

> **Note:** Not all formats support comment-based configuration. DITA and XML
> have no documented comment-based directives (XML content is processed after
> conversion to HTML, where HTML comment directives apply).

### Comment syntax per format

| Format           | Comment wrapper           | Example                    |
| ---------------- | ------------------------- | -------------------------- |
| Markdown         | `<!-- ... -->`            | `<!-- vale off -->`        |
| MDX              | `{/* ... */}`             | `{/* vale off */}`         |
| HTML             | `<!-- ... -->`            | `<!-- vale off -->`        |
| AsciiDoc         | `pass:[<!-- ... -->]`     | `pass:[<!-- vale off -->]` |
| reStructuredText | `.. ` (directive comment) | `.. vale off`              |
| Org              | `# ` (comment line)       | `# vale off`               |

### Available directives

All directives below use Markdown comment syntax as example. Substitute the
appropriate comment wrapper for other formats.

**Turn Vale off/on entirely:**

```
<!-- vale off -->
This text will be ignored.
<!-- vale on -->
```

**Turn off a specific rule:**

```
<!-- vale Style.Redundancy = NO -->
This is some text ACT test
<!-- vale Style.Redundancy = YES -->
```

**Turn off specific match(es) within a rule:**

```
<!-- vale Style.Redundancy["ACT test","OTHER"] = NO -->
This is some text ACT test
<!-- vale Style.Redundancy["ACT test","OTHER"] = YES -->
```

**Turn on or off specific styles:**

```
<!-- vale StyleName1 = YES -->
<!-- vale StyleName2 = NO -->
```

**Set styles (enables them and switches off any other styles):**

```
<!-- vale style = StyleName1 -->
<!-- vale styles = StyleName1, StyleName2 -->
```

## Format-specific details

### Markdown (GFM)

- Built-in support for GitHub-Flavored Markdown.
- No external dependencies required.
- Extensions: `.md`, `.mdown`, `.markdown`, `.markdn`.
- Ignores: indented blocks (4+ spaces), fenced blocks, code spans, URLs.

### MDX

- Requires external program `mdx2vast`.
- Install: `npm install -g mdx2vast`. Must be available on `$PATH`.
- Extension: `.mdx`.
- Ignores: fenced blocks, code spans, URLs, JSX expressions and components, ESM
  imports and exports.
- Uses JSX comment syntax `{/* ... */}` for inline directives.

### HTML

- Built-in HTML5 support.
- Extensions: `.html`, `.htm`, `.shtml`, `.xhtml`.
- Ignores: `script`, `style`, `pre`, `code`, `tt` tags; URLs.

### AsciiDoc

- Requires external program `asciidoctor`. Must be on `$PATH`.
- Extensions: `.adoc`, `.asciidoc`, `.asc`.
- Ignores: literals and source code, URLs.
- Comment directives must be wrapped in `pass:[<!-- ... -->]`.

**Attributes configuration:** Customize how `asciidoctor` is called by passing
document attributes in `.vale.ini`:

```ini
StylesPath = styles

[asciidoctor]
# attribute = value
# YES enables, NO disables

# enable an attribute
experimental = YES

# assign a specific value
attribute-missing = drop

[*.adoc]
BasedOnStyles = Vale
```

### reStructuredText

- Requires external program `rst2html` from the Python `docutils` package.
- Install: `pip install docutils`. Must be on `$PATH`.
- Extensions: `.rst`, `.rest`.
- Ignores: literal blocks, inline literals, URLs.
- Comment directives use the reStructuredText comment syntax: `.. vale off`.

### Org

- Built-in support.
- Extension: `.org`.
- Ignores: code blocks, literal examples, code and verbatim strings, URLs.
- Comment directives use Org comment syntax: `# vale off`.

### DITA

- Requires DITA Open Toolkit. Follow installation instructions including adding
  the `bin` directory to `PATH`.
- Extension: `.dita`.
- Ignores: `<codeblock>`, `<tt>`, `<codeph>` elements.
- **Performance note:** Due to the dependency on the third-party `dita` command,
  DITA files will likely have worse performance compared to other formats.
- No comment-based configuration directives documented.

### XML

- Requires external program `xsltproc`. Must be on `$PATH`.
- Install:
  - Windows (Chocolatey): `choco install xsltproc`
  - macOS (Homebrew): `brew install libxslt`
  - Debian/Ubuntu: `apt-get install xsltproc`
- Extension: `.xml`.
- Requires a version 1.0 XSL Transformation (XSLT) for converting to HTML via
  the `Transform` option:

```ini
[*.xml]
Transform = docbook-xsl-snapshot/html/docbook.xsl
```

- Once converted, Vale follows the same rules as HTML (ignoring `script`,
  `style`, `pre`, `code`, `tt` tags).
- No comment-based configuration directives documented for XML specifically;
  after conversion, HTML comment directives apply.

## Code comments

Vale supports linting source code comments across many languages. Each comment
style is assigned a scope.

### Supported languages

In the scope columns below, replace `.ext` with the file's actual extension
(e.g., `text.comment.line.c`, `text.comment.block.py`).

| Language   | Extensions                            | Line comment scopes                    | Block comment scopes                   |
| ---------- | ------------------------------------- | -------------------------------------- | -------------------------------------- |
| C          | `.c`, `.h`                            | `//` → `text.comment.line.ext`         | `/*` → `text.comment.block.ext`        |
| C#         | `.cs`, `.csx`                         | `//` → `text.comment.line.ext`         | `/*` → `text.comment.block.ext`        |
| C++        | `.cpp`, `.cc`, `.cxx`, `.hpp`         | `//` → `text.comment.line.ext`         | `/*` → `text.comment.block.ext`        |
| CSS        | `.css`                                | —                                      | `/*` → `text.comment.block.ext`        |
| Go         | `.go`                                 | `//` → `text.comment.line.ext`         | `/*` → `text.comment.block.ext`        |
| Haskell    | `.hs`                                 | `--` → `text.comment.line.ext`         | `{-` → `text.comment.block.ext`        |
| Java       | `.java`, `.bsh`                       | `//` → `text.comment.line.ext`         | `/*` → `text.comment.block.ext`        |
| JavaScript | `.js`                                 | `//` → `text.comment.line.ext`         | `/*` → `text.comment.block.ext`        |
| Julia      | `.jl`                                 | `#`, `"..."` → `text.comment.line.ext` | `#=`, `"""` → `text.comment.block.ext` |
| LESS       | `.less`                               | `//` → `text.comment.line.ext`         | `/*` → `text.comment.block.ext`        |
| Lua        | `.lua`                                | `--` → `text.comment.line.ext`         | `--[[` → `text.comment.block.ext`      |
| Perl       | `.pl`, `.pm`, `.pod`                  | `#` → `text.comment.line.ext`          | —                                      |
| PHP        | `.php`                                | `//`, `#` → `text.comment.line.ext`    | `/*` → `text.comment.block.ext`        |
| PowerShell | `.ps1`                                | `#` → `text.comment.line.ext`          | `<#` → `text.comment.block.ext`        |
| Protobuf   | `.proto`                              | `//` → `text.comment.line.ext`         | `/*` → `text.comment.block.ext`        |
| Python     | `.py`, `.py3`, `.pyw`, `.pyi`, `.rpy` | `#` → `text.comment.line.ext`          | `"""` → `text.comment.block.ext`       |
| R          | `.r`, `.R`                            | `#` → `text.comment.line.ext`          | —                                      |
| Ruby       | `.rb`                                 | `#` → `text.comment.line.ext`          | `^=begin` → `text.comment.block.ext`   |
| Rust       | `.rs`                                 | `//` → `text.comment.line.ext`         | —                                      |
| Sass       | `.sass`                               | `//` → `text.comment.line.ext`         | `/*` → `text.comment.block.ext`        |
| Scala      | `.scala`, `.sbt`                      | `//` → `text.comment.line.ext`         | —                                      |
| Swift      | `.swift`                              | `//` → `text.comment.line.ext`         | `/*` → `text.comment.block.ext`        |
| TypeScript | `.ts`, `.tsx`                         | `//` → `text.comment.line.ext`         | `/*` → `text.comment.block.ext`        |

### Comment-to-markup associations

When source code comments contain embedded markup (e.g., Markdown in Rust doc
comments), you can associate the comment scope with a markup format using the
`[formats]` section in `.vale.ini`. This allows Vale to lint the embedded markup
with full format support (ignore patterns, comment-based configuration, etc.).

```ini
StylesPath = styles
MinAlertLevel = suggestion

[formats]
# Associate Rust files with Markdown
rs = md

[*.{rs,md}]
BasedOnStyles = Vale
```

Example Rust doc comment with embedded Markdown:

````rust
impl Person {
    /// Creates a person with the given name.
    ///
    /// # Examples
    ///
    /// ```
    /// use doc::Person;
    /// let person = Person::new("name");
    /// ```
    pub fn new(name: &str) -> Person {
        Person {
            name: name.to_string(),
        }
    }
}
````

## Front matter

Vale supports linting front matter fields in Markdown, AsciiDoc,
reStructuredText, MDX, and Org files.

### Supported front matter types

- YAML
- TOML
- JSON

### Scope generation

Each front matter field is dynamically assigned its own scope in the form
`text.frontmatter.<field_name>`.

Given this YAML front matter:

```yaml
---
title: "My document"
description: "A short summary of the document's purpose."
author: "John Doe"
---
```

The generated scopes are:

| Field         | Scope                          |
| ------------- | ------------------------------ |
| `title`       | `text.frontmatter.title`       |
| `description` | `text.frontmatter.description` |
| `author`      | `text.frontmatter.author`      |

### Targeting front matter fields in rules

Use the dynamically generated scope in a rule's `scope:` field:

```yaml
extends: capitalization
message: "'%s' should be in title case"
level: warning
scope: text.frontmatter.title
```

This rule applies only to the `title` field in the front matter.

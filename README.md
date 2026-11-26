# tree-sitter-markdown-text

Markdown grammar for [tree-sitter](https://github.com/tree-sitter/tree-sitter), shaped so that its AST lines up with the [textlint `TxtNode`](https://github.com/textlint/textlint/blob/master/docs/txtnode.md) model. Maintained at **[`gratex/tree-sitter-markdown-text`](https://github.com/gratex/tree-sitter-markdown-text)** — current grammar version **1.3.0** (see `tree-sitter.json`; tags are the release contract).

Parses `.md` (and `.markdown`, `.mdown`, `.mkd`, `.mkdn`) files into a concrete syntax tree covering the full CommonMark block structure plus common extensions (GFM pipe tables, task lists, GFM alerts, YAML/TOML front matter, Pandoc math and directive blocks, footnotes, MDX JSX). Inline content is surfaced as structured children of the `inline` wrapper: classified tokens (`word_token`, `numeric_token`, `identifier_like_token`, `path_like_token`) and punctuation-class nodes (`terminator`, `separator`, `bracket`, `operator_like`), plus inline structural nodes (`emphasis`, `strong`, `strikethrough`, `link`, `image`, `autolink`, `inline_code`, `html_inline`, `math_inline`, `mdx_jsx_inline`, `footnote_reference`).

## What and why

### The problem with syntax-highlighting grammars

Existing tree-sitter Markdown grammars are optimized for syntax highlighting and editor tooling: the `inline` wrapper is opaque, every word is one big text run, and prose-level structure is something the consumer reconstructs with regex. That works for colorizing a file, but it falls apart the moment you want to *reason about prose*.

This fork exists because **Rea Skills** needs to reason about prose, not colorize it: tell `path_like_token` from `identifier_like_token` from `word_token`, classify every link into its four reference forms, and see tables / math / front-matter / MDX as first-class nodes — all **without re-tokenizing**.

The original driver was [mehen](https://github.com/ophidiarium/mehen) — a CLI that computes documentation-aware metrics on Markdown files in software repositories: Halstead-style metrics over Markdown constructs, [repository grounding](https://github.com/ophidiarium/mehen/blob/main/docs/mehen_markdown_metrics_research_foundation.md#15-repository-grounding-score), [link debt](https://github.com/ophidiarium/mehen/blob/main/docs/mehen_markdown_metrics_research_foundation.md#11-link-and-reference-metrics), [artifact debt](https://github.com/ophidiarium/mehen/blob/main/docs/mehen_markdown_metrics_research_foundation.md#19-artifact-debt-score), and a language-aware prose layer. A grammar that surfaces those distinctions natively is faster and more accurate than bolting them onto a highlighting grammar with post-processing.

Other use cases that benefit from the same shape:

- doc-quality linters (extending `textlint` / `vale` / `proselint` with deeper AST queries),
- prose stylometry / readability tooling that needs sentence-level segmentation outside of code fences,
- AI-assisted documentation review — structural signals (filler density, evidence coverage, repository grounding),
- any tool that needs to walk Markdown the way a compiler walks source code, not the way an editor highlights it.

### Non-negotiable principles

That single goal drives every departure from upstream in this repo:

- **Structured `inline` children** (`text_span`, `word_token`, `path_like_token`, `terminator`, `separator`, …) are the whole point. If an edit ever collapses the `inline` wrapper back into an opaque text run, the fork has lost its reason to exist.
- **Explicit link kinds** (`inline_link`, `full_reference_link`, `collapsed_reference_link`, `shortcut_link`) exist so consumers can classify links in a single pass.
- **Everything else** (pipe tables, math blocks, front matter, directive blocks, `html_comment_block`, `blank_line`, `level` fields) serves the same principle: *the AST must tell you what a token is without re-parsing the source*.

The fork's value is the **shape of the AST**, not merely that it parses Markdown.

### Lineage (fork chain)

```text
tree-sitter-grammars/tree-sitter-markdown   (two-grammar upstream: block + inline)
  └─ ophi-dev/tree-sitter-markdown-text      (single-tree unification, textlint TxtNode shape;
                                              formerly published as ophidiarium/…, redirects here)
       └─ gratex/tree-sitter-markdown-text   (THIS repository — release line v1.x)
```

The block-level parsing, the `inline` wrapper, and the textlint `TxtNode` alignment all come from upstream. On top of it, this fork adds:

- **Four explicit link kinds.** The single `link` node is split into `inline_link`, `full_reference_link`, `collapsed_reference_link`, and `shortcut_link`, so a consumer can tell link syntax apart without re-tokenizing the source.
- **Structural content inside link labels.** `link_label` children are parsed into structured inline nodes (code spans, emphasis/strong, autolinks, etc.) instead of opaque text.
- **All image reference forms.** Images support the same four forms as links — `![alt](dest)` inline, `![alt][label]` full reference, `![alt][]` collapsed reference, `![alt]` shortcut reference. A standalone image paragraph parses as `paragraph > inline > image` (there is no block-level image node; the former `image_block` kind was removed in 1.3.0).
- **A reference-style link resolver layer** in [`tools/link-resolver`](tools/link-resolver/) that consumes the parse tree (via ast-grep custom-language scanning) and resolves reference links against `link_reference_definition`s per CommonMark normalization rules. It is a post-processing tool, not part of the grammar.

### Runaway-emphasis fix

Upstream has a known bug where an unclosed paired delimiter (`*`, `_`, `**`, `__`, `~~`, `$$`, `:::`, `<...>`, a backtick run, or `[^...`) commits the parser into a state it cannot close, and the resulting error recovery swallows all following block-level content (headings, code blocks, lists, blockquotes, tables) into a single paragraph/ERROR region that never appears in the tree. Filed upstream as [ophi-dev/tree-sitter-markdown-text#23](https://github.com/ophi-dev/tree-sitter-markdown-text/issues/23); still open as of 2026-08.

This fork fixes it with a shared lookahead mechanism (`has_closing_delimiter` / `looks_like_block_start` in `src/scanner.c`): before committing to open any paired construct, the scanner verifies a matching closing delimiter exists before the current block boundary. If none is found, the delimiter degrades to plain text instead of triggering runaway recovery. The fix is applied uniformly across all affected constructs: emphasis, strong, strikethrough, math blocks, directive blocks, autolinks, footnote references, and inline code.

**Known limitations** — the safe-degradation cases below are **not exhaustive**: several real inputs do produce an `ERROR` node (some contained, one cross-block that swallows the rest of the document). For the full, current list see [KNOWN-LIMITATIONS.md](KNOWN-LIMITATIONS.md). The bullets here are the deliberate, spec-correct safe-degradation subset:

- `_text\n_` — a lone closing `_` on its own line is not recognized as a closer (indistinguishable from a `___` thematic-break start without extra lookahead).
- `***bold+italic***` — a run of three stars is not split into nested `strong`+`emphasis`; the whole run degrades to text.
- A strikethrough closer (`~~`) alone on a new line is not formed, because a `~~~` fence is the more likely reading of that line start.
- The block-boundary heuristic (`looks_like_block_start`) recognizes setext underlines (`===`), ordered-list items (`1. `), and CommonMark type-6 HTML block start tags (`<div>` / `</div>`). It deliberately does **not** treat indented code (4-space lines) or type-7 inline HTML (`<span>`) as block starts — per CommonMark those cannot interrupt a paragraph; a closer following only such a line may not be found (still degrades to text, never an `ERROR`).
- An autolink at the very start of a line is not recognized (`<https://...>` at column 0 parses as an HTML block, not an autolink), because a block HTML tag takes precedence there.

These additions are surfaced as new kinds in `src/node-types.json`; see the [Node kind reference](#node-kind-reference).

## Features

### Block nodes

- **Document structure** — `document`, nested `section` wrappers around ATX headings, `paragraph`, `blank_line` (as a first-class node).
- **Headings** — ATX (`#`..`######`) and setext (`===`/`---`) with the heading level exposed as a `level` field on both `atx_heading` and `setext_heading`.
- **Code blocks** — indented code blocks and fenced code blocks (backtick and tilde), with `info_string`/`language` children for the GFM language tag.
- **Math blocks** — Pandoc/GitLab/KaTeX display math (`$$…$$`) as a dedicated `math_block` with `math_block_delimiter`/`math_block_content` children.
- **Lists** — unordered (`+`/`-`/`*`) and ordered (`1.`/`1)`) list markers. GFM task list items are promoted to `task_list_item` (distinct from `list_item`), with `task_list_marker_checked`/`task_list_marker_unchecked` markers.
- **Block quotes and callouts** — nested quotes and lazy continuations. A block quote whose first paragraph begins with `[!NOTE]` / `[!TIP]` / `[!IMPORTANT]` / `[!WARNING]` / `[!CAUTION]` (or any uppercase-only label) is surfaced as `callout` with a `callout_type` field.
- **Thematic breaks** — `---`, `***`, `___`.
- **HTML blocks** — all 7 CommonMark HTML block types; block-level HTML comments are aliased to `html_comment_block` for easy metric extraction.
- **MDX JSX blocks** — shallow `mdx_jsx_block` for lines that start with an MDX-style JSX element (`<Component ...>`, `<Component/>`, `</Component>`). Component-style mixed-case names disambiguate from all-caps HTML blocks such as `<DIV>`.
- **Pipe tables** — `pipe_table` with `pipe_table_header`, `pipe_table_delimiter_row`, `pipe_table_row`, `pipe_table_cell`, `pipe_table_align_left`/`pipe_table_align_right`.
- **Link reference definitions** — `link_reference_definition` with `link_label`/`link_destination`/`link_title` children.
- **Footnote definitions** — `footnote_definition` (`[^id]: …`) with a `footnote_label` child.
- **Directive blocks** — generic container directives (`:::name … :::`, per remark-directive / MyST / Pandoc fenced divs) as `directive_block` with `directive_block_delimiter`/`directive_name`/`directive_block_content` children.
- **Images in paragraphs** — a paragraph consisting of a single image (`![alt](dest)` on its own line) is parsed as a normal `paragraph` with an inline `image` child. There is no block-level image node, so standalone, EOF and titled images all share the same `paragraph > inline > image` shape.
- **Front matter** — YAML (`---` fenced) as `minus_metadata`, TOML (`+++` fenced) as `plus_metadata`.
- **Injections query** — ships `queries/injections.scm` that injects into fenced-code-block info strings, HTML blocks, and front matter (no `markdown_inline` injection: this grammar's `inline` wrapper is already structured).

### Inline nodes (children of the `inline` wrapper)

- **Classified text tokens** — `text_span` wraps runs of classified tokens: `word_token` (Unicode alphabetic), `numeric_token` (integers, decimals, versions), `identifier_like_token` (camelCase / PascalCase / snake_case), `path_like_token` (paths with `/` separators or dotted identifiers).
- **Punctuation classes** — every punctuation lexeme is classified: `terminator` (`.`, `?`, `!`, `。`, `…`), `separator` (`,`, `;`, `:`), `bracket` (`(`, `)`, `[`, `]`, `{`, `}`, `<`, `>`), `operator_like` (`::`, `->`, `=>`, `=`, `+`, `-`, `*`, `/`, `|`, `&`, and other punctuation).
- **Emphasis / strong / strikethrough** — `emphasis` (`*…*` or `_…_`), `strong` (`**…**` or `__…__`), `strikethrough` (`~~…~~`), each with a `_delimiter`/`_content`/`_delimiter` sub-tree.
- **Code spans** — `inline_code` with matched backtick-run delimiters (1 or 2 backticks).
- **Links and images** — `link` (inline, full-reference, collapsed-reference, shortcut-reference forms) and `image` (`![alt](dest)` or `![alt][ref]`). Both expose `link_label`/`link_destination`/`link_title` children. See the [node kind reference](#node-kind-reference) for the exact per-form kinds.
- **Autolinks** — `autolink` with `uri` or `email` children for `<https://…>` and `<user@example.com>`.
- **Raw HTML inline** — `html_inline` with `html_open_tag`/`html_close_tag`/`html_comment`/`html_cdata`/`html_declaration`/`html_processing_instruction` children.
- **MDX JSX inline** — shallow `mdx_jsx_inline` with `mdx_jsx_open_tag`/`mdx_jsx_close_tag`/`mdx_jsx_expression` children.
- **Inline math** — `math_inline` (`$…$`) with `math_inline_delimiter`/`math_inline_content` children. Disambiguated from `math_block` (`$$…$$`).
- **Footnote references** — `footnote_reference` (`[^id]` inside prose) with a `footnote_reference_label` child.

## Example

````markdown
# Heading

A paragraph with inline content.

- one
- two

```go
func main() {}
```
````

Parsed tree (abbreviated):

```text
(document
  (section
    (atx_heading level: (atx_h1_marker) heading_content: (inline))
    (blank_line)
    (paragraph (inline))
    (blank_line)
    (list
      (list_item (list_marker_minus) (paragraph (inline)))
      (list_item (list_marker_minus) (paragraph (inline))))
    (blank_line)
    (fenced_code_block
      (fenced_code_block_delimiter)
      (info_string (language))
      (code_fence_content)
      (fenced_code_block_delimiter))))
```

## Installation

### Installing as a git dependency (recommended for consumer projects)

The most common way to consume this grammar from another project is as an npm
git dependency. Point at this fork (`gratex/...`) and **pin a specific tag or
commit SHA**:

```sh
npm install git+https://github.com/gratex/tree-sitter-markdown-text.git#v1.3.0
```

or, equivalently, pin by full commit SHA:

```sh
npm install git+https://github.com/gratex/tree-sitter-markdown-text.git#106bfe90f127e84c06af28275ddf87134d534c68
```

List the available release tags with:

```sh
git ls-remote --tags https://github.com/gratex/tree-sitter-markdown-text.git
```

**Why pin a tag or SHA instead of `#main`?**

- **Reproducible builds.** A floating ref like `#main` resolves to whatever is
  on the branch the day the dependency is (re)installed. Two developers —
  or one CI run and a local machine — can silently end up with different
  parser versions locked in different `package-lock.json` snapshots, and a fresh
  `npm install` on a clean checkout can pull a newer, behaviorally different
  grammar than the one the code was written against.
- **Protection from future upstream changes.** This is a fork that will keep
  evolving. Pinning a SHA or a tag makes the grammar a *fixed contract*: the AST
  kinds and fields your rules depend on will not change under you. `#main` gives
  you none of that guarantee.
- **A tag is as immutable as a SHA but readable.** `v1.3.0` is easy to grep in
  lockfiles and to reason about in PRs, yet it still resolves to one exact
  commit. Treat tags published here as "do not move" contracts. (`git tag -f`
  can re-point a tag, so a tag is only as stable as its maintainer's discipline
  — a full commit SHA is the strongest possible guarantee.)

The `install` script of the package compiles the native Node binding via
`node-gyp-build`, and the npm tarball includes `src/` (the C parser source), so
a working C toolchain is all that is required beyond npm itself.

### npm

```sh
npm install tree-sitter-markdown-text
```

### Cargo

```sh
cargo add tree-sitter-markdown-text
```

### PyPI

```sh
pip install tree-sitter-markdown-text
```

### Go

```go
import tree_sitter_markdown_text "github.com/ophidiarium/tree-sitter-markdown-text/bindings/go"
```

The root package also exports the bundled queries via `go:embed`:

```go
import markdown "github.com/ophidiarium/tree-sitter-markdown-text"

lang := markdown.GetLanguage()
query, _ := markdown.GetHighlightsQuery()
```

> **Note:** the Go module path still declares `github.com/ophidiarium/...`
> (the upstream repository), because renaming a Go module path is a breaking
> change for existing consumers. The registry (npm/Cargo/PyPI) examples above
> are the *published bindings* of the upstream package. If you consume this
> fork directly, prefer the git-dependency form from the previous section and
> build the shared library yourself as described below.

## Migrating from upstream tree-sitter-markdown

If you already have a consumer that loads the *original*
[`tree-sitter-grammars/tree-sitter-markdown`](https://github.com/tree-sitter-grammars/tree-sitter-markdown)
grammar and you swap the dependency to this fork, **loading and compiling will
just work**, but **AST queries and rules will not**. This grammar keeps the same
grammar name (`markdown`), scope (`source.markdown`), exported symbol
(`tree_sitter_markdown`), and ABI 14, so any tree-sitter host accepts it
silently. What changes is the *shape of the tree*: this fork's value is a
structured, textlint-aligned AST, not a syntax-highlighting one. A consumer
that matches upstream node kinds will not error — it will silently match
nothing and produce empty results.

To migrate, do three things:

1. **Repoint the dependency and pin it.** Use the
   [git-dependency form](#installing-as-a-git-dependency-recommended-for-consumer-projects)
   and pin a tag or commit SHA. Do not float on `#main` — the fork evolves and
   the AST contract would change under you.
2. **If you load the grammar as a dynamic library** (ast-grep custom languages,
   dlopen, etc.), build the `.so` and register it as described in
   [Using with ast-grep](#using-with-ast-grep). Remember: **do not name the
   custom language `markdown`** on ast-grep ≥0.43 — the built-in Markdown
   shadows it and your rules fail with `Invalid Kind`.
3. **Rewrite queries and rules from upstream kinds to fork kinds** using the
   tables below. This is the only non-mechanical step.

### Upstream kind → fork kind

Every mapping below was produced by diffing `src/node-types.json` of this fork
against the original grammar's `node-types.json` (v0.7.1). "Same name" kinds
keep their name but may have a different shape; "no direct equivalent" kinds
require a structural rewrite.

**Block nodes**

| Upstream kind | Fork kind | Notes |
|---|---|---|
| `document`, `paragraph`, `block_quote`, `fenced_code_block`, `indented_code_block`, `info_string`, `code_fence_content`, `thematic_break`, `link_reference_definition`, `atx_heading`, `setext_heading` | same name | Fork adds a `level:` field to both `atx_heading` and `setext_heading`. |
| `tight_list` / `loose_list` | `list` | Fork has a single `list` node; the tight/loose distinction is not exposed as separate kinds. |
| `table` | `pipe_table` | |
| `table_header_row` | `pipe_table_header` | |
| `table_delimiter_row` | `pipe_table_delimiter_row` | |
| `table_data_row` | `pipe_table_row` | |
| `table_cell` | `pipe_table_cell` | |
| `table_column_alignment` | `pipe_table_align_left` / `pipe_table_align_right` | |
| `list_marker` | `list_marker_dot` / `list_marker_minus` / `list_marker_parenthesis` / `list_marker_plus` / `list_marker_star` | Upstream's generic marker is split by punctuation. |
| `task_list_item` | `task_list_item` | |
| `task_list_item_marker` | `task_list_marker_checked` / `task_list_marker_unchecked` | Checked/unchecked are distinct kinds. |
| `heading_content` | `inline` | Headings carry their content as the structured `inline` wrapper. |

**Inline nodes**

| Upstream kind | Fork kind | Notes |
|---|---|---|
| `link` | `inline_link` / `full_reference_link` / `collapsed_reference_link` / `shortcut_link` | The four reference forms are explicit kinds in this fork. |
| `link_text` / `image_description` | `link_label` | Label text is parsed structurally, not opaque. |
| `link_destination`, `link_title` | same name | |
| `image` | `image` | |
| `code_span` | `inline_code` | |
| `emphasis` | `emphasis` | |
| `strong_emphasis` | `strong` | |
| `strikethrough` | `strikethrough` | |
| `uri_autolink` / `email_autolink` / `www_autolink` | `autolink` | Single kind with `uri` / `email` children. |
| `character_reference` | `entity_reference` / `numeric_character_reference` | Named vs. numeric references are split. |
| `text` | `text_span` + classified tokens | `text_span` wraps `word_token`, `numeric_token`, `identifier_like_token`, `path_like_token`; punctuation is classified as `terminator`, `separator`, `bracket`, `operator_like`. |
| `line_break` / `soft_line_break` / `hard_line_break` | no direct equivalent | Line breaks are not surfaced as dedicated kinds; handle them at the `text_span` level. |
| `html_open_tag`, `html_close_tag` | same name | |
| `html_self_closing_tag` | no direct equivalent | Self-closing tags are not split into their own kind. |
| `html_comment` | `html_comment` (inline) / `html_comment_block` (block) | |
| `html_cdata_section` | `html_cdata` | |
| `html_declaration`, `html_processing_instruction` | same name | |
| `html_atrribute` (sic), `html_attribute_key`, `html_attribute_value`, `html_tag_name` | no direct equivalent | This fork's `html_open_tag` is shallow and does not expose attribute children. |
| `virtual_space` | no direct equivalent | Whitespace handling differs; see `blank_line` and the inline token classes. |

**Kinds new to this fork** (no upstream counterpart — add rules for them if you
need them): `blank_line`, `section`, `callout` (+ `callout_type`),
`directive_block`, `footnote_definition`, `footnote_reference`, `math_block`,
`math_inline`, `mdx_jsx_block`, `mdx_jsx_inline`, `minus_metadata`,
`plus_metadata`, `backslash_escape`.

The two changes that break the most highlight/analysis consumers are:
`inline_span` / `inline_spans` do **not** exist here — the `inline` wrapper is
structured, and the old `text` blob is replaced by `text_span` plus the
classified tokens. Anything that matched `text` or `inline_span` needs a
rewrite to the fork's kinds (see the [Node kind reference](#node-kind-reference)
for the full list).

*Migration section generated/modified by AI Kilo Code 7.4.17-gratex-009, used
model deepseek-v4-flash.*

<!-- Generated/modified by AI Kilo Code 7.4.17-gratex-009, used model deepseek-v4-flash -->

## Relationship to textlint

The grammar is structurally close to the textlint AST. Every block-level `TxtNode` type has a direct counterpart here; inline `TxtNode` types (`Str`, `Emphasis`, `Strong`, `Link`, `Image`, `Code`, `Html`, `Delete`, `FootnoteReference`) also have direct counterparts as children of the `inline` wrapper. Names stay snake_case per the tree-sitter convention; consumers map names themselves. See [docs/textlint-mapping.md](docs/textlint-mapping.md) for the full table.

## Building the shared library (`markdown-text.so`) from source

Any consumer that loads the grammar as a dynamically-linked library (notably
[ast-grep custom languages](#using-with-ast-grep)) needs a `.so`/`.dylib`/`.dll`
built from the parser sources. The npm git dependency ships `src/parser.c`,
`src/scanner.c`, and the `src/tree_sitter/` headers, so the only external
requirement is a C compiler.

Since the npm tarball ships only the sources, the tree-sitter CLI and the
universal `cc` command work directly in the installed package; the
`Makefile` route requires a clone of this repository — the `Makefile`
is not part of the npm tarball. Three options:

**Option 1 — the repository's own `Makefile`** (works on Linux/macOS, from a clone of this repository):

```sh
make
make clean
```

This produces `libtree-sitter-markdown-text.so` (and `.a`, `.pc`).

**Option 2 — the tree-sitter CLI** (also produces the grammar's language symbol):

```sh
tree-sitter build --output markdown-text.so
```

**Minimal universal fallback** (no Makefile, no tree-sitter CLI — just a
C compiler, e.g. `gcc`/`cc`/`clang`):

```sh
cc -shared -fPIC -O2 -Isrc -o markdown-text.so src/parser.c src/scanner.c
```

All three routes export the grammar's language entry point as
`tree_sitter_markdown` (the grammar name is `markdown`, per `tree-sitter.json`).

## Using with ast-grep

### Registering as a custom language in `sgconfig.yml`

Once the `.so` is built (see above), register it in the `customLanguages`
section of any ast-grep project's `sgconfig.yml`.

> **Don't name the custom language `markdown`.** ast-grep ≥0.43 ships a
> built-in Markdown language, and a custom language registered under the same
> key silently shadows it: rules that target this grammar's kinds then fail
> with `Invalid Kind` (`inline_link`, `shortcut_link`, `image`, …), and kinds
> that overlap with the built-in grammar (`paragraph`, `inline`, `atx_heading`,
> `section`) are matched against the *built-in* parse tree instead of this one.
> Register under a distinct key and point `languageSymbol` at the exported
> symbol. (Verified on ast-grep 0.45.)

Minimal working example (recommended):

```yaml
# sgconfig.yml
ruleDirs:
  - ./rules
customLanguages:
  markdownText:
    libraryPath: ./vendor/markdown-text.so   # relative to sgconfig.yml, or absolute
    extensions: [md, markdown, mdown, mkd, mkdn]
    languageSymbol: tree_sitter_markdown      # symbol exported by this grammar
    expandoChar: _                            # optional: replaces '$' in patterns
```

`languageSymbol` names the loader symbol this grammar exports
(`tree_sitter_markdown`, per `tree-sitter.json`). The custom-language key
(`markdownText` above) is arbitrary and only labels the language in rules, so
keep it distinct from built-in language names.

Then rules can target the grammar's kinds, e.g.:

```yaml
id: no-broken-shortcut-links
language: markdownText
severity: warning
rule:
  kind: shortcut_link
```

**Not recommended: naming the language `markdown`.** On ast-grep builds without
a built-in Markdown language, default symbol inference would load
`tree_sitter_markdown` from the `.so` and this would work:

```yaml
customLanguages:
  markdown:
    libraryPath: ./vendor/markdown-text.so
    extensions: [md, markdown, mdown, mkd, mkdn]
    expandoChar: _
```

But on ast-grep ≥0.43 this key conflicts with the built-in Markdown support
(see the warning above), so prefer the `markdownText` form unless you have
confirmed your ast-grep build has no built-in Markdown language.

See the official [ast-grep custom-language
guide](https://ast-grep.github.io/advanced/custom-language.html) for the full
option list (`outlineRules`, per-platform `libraryPath` maps, etc.).

### Node kind reference

Exact kinds this grammar produces, taken verbatim from `src/node-types.json`.
Use them as `kind:` selectors in ast-grep rules (or as node names in any
tree-sitter consumer). Anonymous punctuation/lexemes (`*`, **, `` ` ``, `~~`,
`[`, `]`, …) are omitted here; the full list lives in `src/node-types.json`.

**Links** — the four explicit reference kinds (this fork's addition), plus
autolinks and reference definitions:

| kind | meaning | children |
|---|---|---|
| `inline_link` | `[text](dest "title")` | `link_label`, `link_destination`, `link_title` |
| `full_reference_link` | `[text][label]` | `link_label` |
| `collapsed_reference_link` | `[text][]` | `link_label` |
| `shortcut_link` | `[label]` (bare) | `link_label` |
| `autolink` | `<https://…>` / `<user@…>` | `uri`, `email` |
| `link_reference_definition` | `[label]: dest "title"` | `link_label`, `link_destination`, `link_title` |

**Images** — all four reference forms collapse into one `image` node:

| kind | meaning | children |
|---|---|---|
| `image` | `![alt](dest)`, `![alt][label]`, `![alt][]`, `![alt]` | `link_label`, `link_destination`, `link_title` |

**Code blocks / code spans:**

| kind | meaning | children |
|---|---|---|
| `fenced_code_block` | ```` ```lang … ``` ```` / `~~~` | `fenced_code_block_delimiter`, `info_string`, `code_fence_content`, `block_continuation` |
| `indented_code_block` | 4-space-indented block | `blank_line`, `block_continuation` |
| `inline_code` | `` `code` `` / ` ``code`` ` | `inline_code_delimiter`, `inline_code_content` |
| `info_string` / `language` | fence language tag | — |

**Shared link part nodes:**

| kind | meaning |
|---|---|
| `link_label` | the `[...]` label text; **children are parsed structurally** (this fork's addition): `text_span`, `emphasis`, `strong`, `strikethrough`, `inline_code`, `autolink`, `html_inline`, `math_inline`, `mdx_jsx_inline`, `backslash_escape`, `entity_reference`, `numeric_character_reference` |
| `link_destination` | the destination (`(dest)` or `label` target) |
| `link_title` | the optional `"title"` |

**Other kinds** (block & inline) you may want as selectors: `document`,
`section`, `paragraph`, `blank_line`, `atx_heading`/`setext_heading` (with
`level:` field), `block_quote`, `callout` (+ `callout_type`), `list`,
`list_item`, `task_list_item`, `thematic_break`, `pipe_table` (+
`pipe_table_header`/`pipe_table_row`/`pipe_table_cell`), `html_block`,
`html_comment_block`, `mdx_jsx_block`, `directive_block`, `footnote_definition`,
`footnote_reference`, `minus_metadata`/`plus_metadata`, `math_block`/
`math_inline`, `inline` (the structured wrapper: `text_span`, `word_token`,
`numeric_token`, `identifier_like_token`, `path_like_token`, `terminator`,
`separator`, `bracket`, `operator_like`, `emphasis`, `strong`,
`strikethrough`, `inline_code`, `html_inline`, `mdx_jsx_inline`,
`footnote_reference`).

## Usage

### Node.js

```javascript
import Parser from "tree-sitter";
import Markdown from "tree-sitter-markdown-text";

const parser = new Parser();
parser.setLanguage(Markdown);

const tree = parser.parse("# hello\n");
console.log(tree.rootNode.toString());
```

### Rust

```rust
let mut parser = tree_sitter::Parser::new();
let language = tree_sitter_markdown_text::LANGUAGE;
parser.set_language(&language.into()).unwrap();

let tree = parser.parse("# hello\n", None).unwrap();
println!("{}", tree.root_node().to_sexp());
```

### Python

```python
from tree_sitter import Language, Parser
import tree_sitter_markdown_text

parser = Parser(Language(tree_sitter_markdown_text.language()))
tree = parser.parse(b"# hello\n")
print(tree.root_node.sexp())
```

## Development

Contributor documentation lives in [AGENTS.md](AGENTS.md) — parser generation
(`npm run generate`, ABI 14 pinning), scanner conventions, corpus rules, and
the full local validation sweep. Known parser limitations and their status are
tracked separately in [KNOWN-LIMITATIONS.md](KNOWN-LIMITATIONS.md).

Quick start for a fresh clone:

```sh
npm ci
npm run generate
npx tree-sitter test
```

## Credits and references

- [tree-sitter-grammars/tree-sitter-markdown](https://github.com/tree-sitter-grammars/tree-sitter-markdown) — upstream grammar, specifically the `split_parser` branch's block grammar, which this grammar is derived from.
- [ophi-dev/tree-sitter-markdown-text](https://github.com/ophi-dev/tree-sitter-markdown-text) — the single-tree, textlint-shaped upstream this fork is based on (formerly `ophidiarium/tree-sitter-markdown-text`).
- [textlint TxtNode](https://github.com/textlint/textlint/blob/master/docs/txtnode.md) — the AST shape this grammar targets for compatibility.
- [CommonMark Spec](https://spec.commonmark.org/) — the block structure this grammar implements.
- [Github Flavored Markdown](https://github.github.com/gfm/) — for the pipe-table and task-list extensions.

## License

[MIT](LICENSE)

---

*README rewritten and generated by AI Kilo Code 7.8.18-gratex-027, used model mimo-2.6-flash.*

<!-- Generated/modified by AI Kilo Code 7.8.18-gratex-027, used model mimo-2.6-flash -->

<!-- "Runaway-emphasis fix" section restored from README.md by AI Kilo Code 7.8.18-gratex-027, used model gti-litellm/deepseek-v4.1-flash -->

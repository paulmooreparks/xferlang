# Conformance suite

The suite is two text archives in txtar form, chosen so that hundreds of small cases live in two reviewable files and any language can split them with twenty lines of code.

## txtar form

A file is a sequence of entries. Each entry begins with a line `-- name --` and runs to the next such line or the end of the file. Everything between is the entry's content, byte for byte, including its final newline. Lines before the first entry are a comment. A name may contain any character except newline.

## valid.txtar

Each case is three entries sharing a stem:

| Entry | Contents |
|---|---|
| `<stem>.xfer` | The input document. |
| `<stem>.canonical.xfer` | The canonical form the parser must produce, byte for byte. |
| `<stem>.value.json` | The value model in tagged JSON. |

Tagged JSON follows the convention toml-test uses, so that every value's type is visible and no JSON number parsing is involved:

```json
{"type": "integer", "value": "42"}
{"type": "fractional", "value": "1.50"}
{"type": "string", "value": "text"}
{"type": "symbol", "value": "dark"}
{"type": "bool", "value": "true"}
{"type": "null"}
{"type": "date", "value": "2025-08-01"}
{"type": "time", "value": "09:30:00"}
{"type": "local-datetime", "value": "2025-08-01T09:30:00"}
{"type": "datetime", "value": "2025-08-01T09:30:00Z"}
```

An object is a JSON object whose members are tagged values in source order; a sequence is a JSON array of tagged values. Integer values are written in decimal regardless of source radix; fractional values are written as in the source with `_` removed. A tagged value or object member that carries annotations has an `"annotations"` array of `{"name": ..., "value": <tagged>}` in source order; the document's `xfer` head, when present, appears as `"xfer"` at the top level beside `"root"`:

```json
{"xfer": 1, "root": {"type": "integer", "value": "42"}}
```

A runner feeds `<stem>.xfer` to the implementation, requires zero error diagnostics, compares the canonical output to `<stem>.canonical.xfer` byte for byte, and compares the value model to `<stem>.value.json` structurally.

## invalid.txtar

Each case is two entries:

| Entry | Contents |
|---|---|
| `<stem>.xfer` | The input document. |
| `<stem>.diag` | One line per required diagnostic: `CODE line:column`, 1-based, column in scalar values. |

A runner requires that every listed diagnostic is reported with that code at that position. It permits further diagnostics, since recovery may report consequences of the first fault, but the listed ones must be present. A case with the single line `CODE *` requires the code at any position (used for limits and encoding).

## bytes/

Cases whose bytes a text editor or version-control system would alter live as individual files under `bytes/`, protected by `.gitattributes`: byte order marks, CR and CRLF line endings, NUL and control characters inside strings, invalid UTF-8, and the limit cases. They use the same stem conventions as the archives (`<stem>.xfer` with either `.canonical.xfer` and `.value.json`, or `.diag`).

A `.diag` stem ending in `-at-minimum` is required only of a parser configured at the specification's minimum limits (spec 3.4). A parser configured with a higher limit MAY accept that document and is still conformant; a runner testing such a parser skips those cases or lowers the limit first.

## Property tests

Implementations MUST also pass two generated tests, described here rather than enumerated:

**Embedding.** For strings drawn at random from the alphabet `a`, `"`, `#`, space, LF, of lengths 0 to 64, with at least ten thousand samples: writing the string with the canonical choice rule, parsing the result as a document, and reading the value yields the original string.

**Whitespace.** For every document in `valid.txtar`: replacing every maximal run of whitespace between tokens (not inside strings or comments) with a single space, parsing, and comparing value models to the original's gives equality; and the canonical form of the collapsed document equals the canonical form of the original.

## Adding a case

Every rule in the specification has at least one valid and one invalid case. When a rule changes, its cases change in the same commit. Name cases by the rule they exercise, in kebab case, prefixed with a three-digit group number that matches the specification section (`4xx` lexical, `5xx` syntax, `6xx` canonical, `7xx` annotations, `8xx` JSON mapping is tested by the CLI, not here).

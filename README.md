# XferLang

XferLang is a data format for people and programs that write data by hand or generate it, with the data model of JSON and three properties JSON does not have: no escape sequences anywhere in the grammar, no significance to line breaks or indentation, and a type for every kind of value a configuration or content file actually contains.

Any text can be placed inside an XferLang string without modification, at any depth of nesting, because a string's delimiter is lengthened to clear its content instead of the content being rewritten to clear the delimiter. A document that holds a document that holds a program holds the program byte for byte.

```xfer
<! xfer 1 !>
{
    name "greeter"
    language javascript
    updated 2025-08-01T09:30:00Z
    script """function greet(name) { console.log("Hello, " + name + "!"); }"""
    tags [ demo "web apps" ]
    </ The entry below is switched off. />
    </ { name "farewell" script "..." } />
}
```

This repository is the specification and the conformance suite. It contains no implementation. Implementations live in their own repositories and are conformant when they pass every case under `tests/`.

| Path | Contents |
|---|---|
| `spec/xferlang.md` | The specification: lexical structure, grammar, semantics, canonical form, JSON mapping, diagnostics, limits. |
| `spec/xferlang.abnf` | The grammar in RFC 5234 ABNF, with the constraints ABNF cannot express stated beside it. |
| `tests/README.md` | The conformance suite's file format and how to run an implementation against it. |
| `tests/valid.txtar` | Documents that must parse, each with its canonical form and its value model. |
| `tests/invalid.txtar` | Documents that must not parse, each with the diagnostic code required. |
| `history/` | The assessment, proposal and reviews that led to this design. Read these for the reasoning behind any rule. |

## Status

Draft 1 of specification 1.0. Nothing is frozen. The specification freezes at 1.0.0 when a reference implementation passes the suite and the first production document has migrated to it. Changes before then are made here and in the suite together; a rule without a conformance case does not exist.

## Versioning

The specification carries a semantic version. A document may state the major version it was written against with `<! xfer 1 !>` as its first token; a parser that does not know that major version rejects the document. Within a major version, every valid document stays valid and keeps its value.

## Licence

MIT, as the original XferLang project.

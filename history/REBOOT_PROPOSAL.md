# XferLang reboot: implementation strategy and clean-sheet design

Written 2026-10-18 as the follow-on to `REBOOT_ASSESSMENT.md`. The assessment concluded that XferLang's first thesis (a human-writable, typed, comment-friendly, escape-free data format) is sound and validated by its one consumer, and that the second thesis (a programmable document preprocessor) grew on top of it and should be separated out. This document proposes how to build the first thesis properly. Recommendations are marked as such; the decisions are Paul's and should be recorded in this file as they are made.

## 1. The product is the specification and its conformance suite

Every format that has lasted (JSON, TOML, and more recently KDL) is defined by a short specification and a language-neutral test suite, and has several independent implementations that pass it. No single parser, however good, makes a format trustworthy; the suite does, because it is what lets a second implementation exist and be checked.

The reboot should therefore produce:

1. A specification: an ABNF grammar, prose semantics for each construct, a canonical serialization, a JSON mapping in both directions, resource-limit requirements, and a diagnostic code list.
2. A conformance suite: a directory of valid documents each paired with its expected canonical form and its expected value model (in the tagged-JSON form toml-test uses), and a directory of invalid documents each paired with the diagnostic code the parser must report. The suite is the test oracle for every implementation.
3. A pure managed .NET implementation, passing the suite, which is what the site and the rest of the portfolio consume.
4. A reference implementation in C, written against the frozen suite and differentially tested against the managed one.
5. Bindings over the C ABI for other languages, generated from the header.

Items 1 to 3 are developed together: a suite with no parser to run against is unvalidated, and the first implementation of any spec finds the spec's faults. The spec freezes when the suite is green on the managed implementation and the site has migrated; the C core is then written against a frozen target. (Review round 1, finding 20.) The current project did these in reverse, and the assessment documents the result.

## 2. C as the core language

The original plan was a Rust core with a C ABI exported from it. The plan today is a C core. This section weighs that.

### 2.1 What C buys

A C library is the lowest common denominator: every language has a well-trodden path to a C header (ctypes and cffi, N-API, cgo, JNI and Panama, P/Invoke), and a Rust core would in practice export the same C ABI through cbindgen anyway, so the binding story is identical. C compiles everywhere, including freestanding targets and WebAssembly through clang, with no runtime and a binary measured in tens of kilobytes. It is also what the parsers that get embedded are written in (libyaml, expat, cJSON, yyjson, Lua), and SQLite shows what the language can reach when the discipline in 2.2 is applied. For a format that wants to be embedded in other people's software, a dependency-free C library is the form they expect.

### 2.2 What C costs

A hand-written C parser of untrusted input is the canonical source of memory-safety CVEs. Rust removes that class of defect at compile time; C requires discipline to remove it. The discipline is well understood, and the cost is paid once in the design:

- **No recursion.** Nesting is handled with an explicit stack in an arena, so depth is bounded by memory rather than by the machine stack, and a deeply nested document cannot crash the process.
- **No raw pointer arithmetic on input.** The lexer works on a bounds-checked slice type, and every access goes through a function that cannot read past the end.
- **Arena allocation.** The parser allocates from one arena the caller provides or that the library creates, and frees it in one call. There are no per-node frees and therefore no use-after-free or double-free paths.
- **Limits are mandatory, not optional.** Maximum document size, nesting depth, string length, delimiter run length and element count are fields of the parser options, with safe defaults, and exceeding one is a diagnostic rather than an allocation.
- **No undefined behavior.** The code is built with `-Wall -Wextra -Werror`, `-fsanitize=address,undefined` in every test run, and MemorySanitizer or Valgrind in CI. Signed overflow, strict-aliasing violations and uninitialized reads are build failures. This is the same rule as "we never rely on undocumented behavior", applied to the C standard.
- **Fuzzing from the first commit.** A libFuzzer or AFL++ harness runs in CI against the parser entry point on every change, with the conformance suite as its seed corpus, and a crash found by the fuzzer is a release blocker.
- **A frozen, small ABI.** Opaque handles, no struct layouts exposed, every function versioned, and a documented stability policy.

SQLite demonstrates that C written this way can be among the most reliable software in existence. It does not come for free, and the fallback is defined rather than left to mood: the question is evaluated at the 1.0 freeze and at each minor release; any memory-safety finding after 1.0 triggers a written review; a second within twelve months triggers a rewrite of the core in Rust behind the same C ABI. The ABI and the conformance suite make that swap possible without touching any consumer. Fuzzing runs as a time-boxed job on every pull request, continuously on a schedule, and through OSS-Fuzz once the project is public.

### 2.3 The .NET implementation should not call the C core

The portfolio's consumers are .NET, and .NET deployments are hostile to native dependencies: NativeAOT, trimming, Blazor WebAssembly, container images, and plain "copy the DLL" simplicity all suffer when a package carries native binaries per platform. A pure managed implementation that passes the same suite is the right shape for the main consumer, and the current `XferConvert` object mapper is a reasonable surface to carry forward onto it. A P/Invoke binding to the C core can exist as a separate package, mainly for differential testing between the two implementations.

**Recommendation:** C core as the reference and embeddable implementation, built under the rules in 2.2; pure managed .NET as the portfolio's implementation; the conformance suite as the thing that makes both legitimate.

## 3. The language, reconsidered from first principles

The goals, restated from the assessment: writable by people and by language models, typed without heuristics, no escape sequences, no commas, no indentation significance, comments, losslessly round-trippable, strictly specified, with a defined JSON mapping.

The following is a proposed Level 0 design. Each item gives the recommendation and the reasoning; the decision column is for Paul.

### 3.1 Lexical foundation

A document is UTF-8; a leading BOM is tolerated and ignored, invalid UTF-8 is an error, and no other encoding is accepted. Before lexing, every CRLF and every lone CR is normalized to LF, so a document means the same thing whatever a version-control system or editor has done to its line endings; a lone CR therefore cannot appear in a value and must be transported encoded.

Whitespace is space, tab, LF and FF (CR has been normalized away). No other code point is whitespace; Unicode spaces are invisible and the language uses whitespace as its separator, so admitting them would be a hazard.

Tokens divide into **self-delimiting** tokens, which are the brackets `{ } [ ]`, the `<` that opens a comment or annotation, and the `!>` that closes an annotation, and **scalar** tokens, which are everything else (numbers, strings, symbols, keywords, temporals, identifiers). A scalar token must be followed by whitespace, a self-delimiting token, or end of input. So `1-2` is an error (two scalars with nothing between them), `"a""b"` is an error, and `{a 1 b [1 2]}` is legal because every omitted space is beside a bracket.

Comments are `</ ... />`, nest by lengthening the run (`<// ... //>`), and may appear wherever whitespace may; a comment counts as whitespace for the separation rule. Comments are preserved in the lossless tree and dropped in the value model.

Line endings have no meaning anywhere outside a string literal. This is a founding principle of the language and the spec states it as an invariant:

> **The whitespace invariant.** Outside string literals, whitespace between a scalar token and a following scalar token is mandatory and may be any non-empty run; whitespace adjacent to a self-delimiting token is optional. Replacing any whitespace run under these rules with any other does not change the document's value. A document can be collapsed to one line or expanded to any layout and means the same thing.

The conformance suite tests it by collapsing every inter-token whitespace run in each valid document to a single space, re-parsing, and comparing value models. The invariant is why there are no line-end comments: a comment that runs to end of line gives a newline meaning, and a formatter could no longer collapse or re-flow a document without rewriting its comments. A one-line note is written `</ like this />`, which is what the live nav file already does.

### 3.2 Scalars without sigils

The current sigils (`#`, `&`, `*`, `^`, `~`, `?`, `@`, `=`, `:`, `_`, `|`) expose CLR storage types and are unmnemonic; two of them appear in zero sample files and one (`&`) is assigned to two concepts. "Explicit types" does not require sigils. It requires that every literal form be lexically distinct and never coerced, which is achievable with ordinary literals, as TOML shows.

- **Integers:** `42`, `-7`, `1_000_000`, `0xFF`, `0b1010`, `0o755`. A decimal integer is `0` or a non-zero digit followed by digits, with an optional leading `-` and no `+`; a leading zero on a non-zero number is an error, so `0755` never means 755 or 493. The radix prefixes are lowercase `0x`, `0o`, `0b`; hex digits may be either case and the canonical form lowercases them. A single `_` may appear between any two digits and nowhere else, in any integer or fractional form. A leading `-` is permitted before any integer form, radix forms included (`-0xFF`), which the current parser rejects. The written value has no width. Implementations map to the consumer's type and report overflow as a diagnostic rather than wrapping or promoting silently; the value model keeps the lexeme so no precision is lost before the consumer asks.
- **Fractional numbers:** `1.5`, `-0.25`, `6.02e23`, `1e5`. A fractional literal is a decimal integer part, then either a `.` followed by at least one digit, an exponent (`e` or `E`, optional sign, digits), or both. `1.`, `.5` and `1.e5` are errors. Any exponent makes the literal fractional, so `1e5` is fractional and `100000` is an integer. The lexeme is exact as written. Whether the consumer receives a decimal or a binary float is the consumer's choice at mapping time, and the mapping must say when it is lossy. This removes the decimal/double split from the syntax while keeping exactness available. The keywords `inf`, `-inf` and `nan` are reserved as fractional literals, as TOML reserves them, so a generator serializing a binary float always has a representation; they have no JSON mapping and `to-json` reports an error for them.
- **Booleans and null:** the keywords `true`, `false`, `null`. Keywords are case-sensitive: `True` and `NULL` are symbols, never coerced. A linter should warn on a symbol that differs from a keyword only in case, since that is the likeliest typo; the parser does not, because a symbol is a legal value.
- **Strings:** `"..."`, with the run-length rule (`""..."".."` when the text contains a quote), multi-line by default, content verbatim. There are no escape sequences anywhere in the language.
- **Symbols:** a bare word that is not one of the six keywords (`true`, `false`, `null`, `inf`, `-inf`, `nan`) is a symbol: `theme dark`, `level info`, `when window`. Symbols are a distinct type that maps to an enum or a string at the consumer's choice and is never coerced to a number or a boolean. This gives the config-file idiom the current language forbids (`{ x hello }` is an error today) without reintroducing YAML's guessing, because the only special words are the keywords. The cost, accepted and recorded in section 7, is that a missing value in an object no longer fails at the next token: `{ a b c d e f g }` with one value missing shifts every later pair and the error lands at `g`. The spec requires the odd-length diagnostic to say that a value is missing and to list the pairings the parser inferred, so the author can see where the shift began, and a schema is the real defense.
- **Temporal values:** bare literals using RFC 3339's `full-date`, `partial-time` and `date-time` productions and nothing else: `2025-08-01`, `09:30:00`, `09:30:00.250`, `2025-08-01T09:30:00Z`, `2025-08-01T09:30:00+08:00`, and a local date-time `2025-08-01T09:30:00` with no offset. The separator between date and time is `T` only; RFC 3339 also permits a lowercase `t` or a space, and both are excluded, the space because it would split the literal into two tokens under the separation rule. Reduced forms such as `09:30` or `2025-08` are errors. They are lexically distinct from numbers and symbols. Durations are deferred from Level 0; they have no JSON mapping and no consumer yet.

| Decision | Recommendation | Status |
|---|---|---|
| Drop all type sigils | Yes. Explicitness is kept by distinct literal forms. The language loses a visual signature and gains the readability its one consumer kept working around (`win "0.9,0.9"` becomes `win [ 0.9 0.9 ]`). | agreed 2026-10-18 |
| One fractional type in syntax | Yes; exactness is preserved in the lexeme and the consumer chooses storage. | agreed 2026-10-18 |
| Symbols as a type | Yes. The alternative, bare words as strings, would make a mistyped `treu` silently a string; a symbol type lets a schema catch it. | agreed 2026-10-18 |
| Bare ISO 8601 temporals | Yes. Proven in TOML; no sigil needed. | agreed 2026-10-18 |

### 3.3 Strings and the run-length rule

The rule is the language's founding idea and stays. Stated precisely, it is this:

> **The embedding invariant.** For any sequence of Unicode scalar values *s*, there is a string literal whose content is exactly *s*, and the bytes of *s* appear in the literal unchanged. Because the same rule applies at every level, the invariant composes: a document that contains a document that contains a string holds the innermost content byte for byte, at a cost of one additional delimiter character per level of nesting.

This is what distinguishes the language from every escape-based format. Escaping rewrites content, and the rewrite stacks at each layer (JSON inside JSON inside JSON doubles its backslashes per layer). Run-length delimiters never touch the content. The invariant is a normative statement in the spec, and the conformance suite tests it with property-based round trips of arbitrary strings through the canonical writer and the parser. The specification settles the following cases, which the current parser gets wrong or leaves undefined.

- **The empty string.** `""` is read today as an opening run of two, and fails. The rule is that a run of exactly two quotes followed by a non-quote is the empty string; otherwise the opening run is maximal.
- **Closing (investigated for review finding 8).** Two rules were considered. Under *at-least* matching (Markdown's rule) the closer is the first run of n or more quotes and any surplus belongs to the content, so `"a""` is the string `a"`; a stray quote after a string silently changes its value. Under *exact* matching (the rule in Rust, Lua and C++) the closer is the first maximal run of exactly n quotes; a longer or shorter run is content, so `"a""` is an unterminated string and the fault is loud. Exact matching is recommended: it fails loudly, it matches the languages authors already know, and it lets the canonical writer pick a run visibly longer than anything inside. Under exact matching the content may contain quote runs of any length other than n, and may not begin or end with a quote (a leading quote is absorbed into the opener; a trailing one merges with the closer into a run that is not n).
- **The repetition exception, stated in full.** Content that begins with a quote, ends with a quote, or consists only of quotes cannot be written with repetition. (The all-quotes case is the begins-with case; it is listed so nobody wonders.) This is not a corner case: the live `sitenav.xfer` has a title that begins and ends with a quote, and it is the entry the author had to work around with an interpolated string. The hash-run form `#"..."#` covers it: the opener is m hashes and a quote, the closer is a quote followed by a maximal run of exactly m hashes, and content may contain anything whose quote-then-hashes runs are not exactly m. Hash-run has no exception at all: every string is representable with m one greater than the longest run of hashes that follows a quote in the content.
- **The canonical writer's choice rule.** If the content contains no quote, write `"c"`. Otherwise, if the content neither begins nor ends with a quote, write repetition with n one greater than the longest quote run inside. Otherwise write hash-run with m one greater than the longest hash run following a quote inside. If "one greater than the longest" would exceed the run-length limit, use instead the smallest length not present among the content's runs, which always exists and is always within the limit, since exact matching only requires that the delimiter run differ from every run inside. This is deterministic, always picks a delimiter visibly longer than anything in the content where the limit allows, and uses hash-run only where repetition cannot. The suite checks it with property-based round trips over strings drawn from the alphabet `a " # space newline`.

Multi-line strings are verbatim, including the newline after the opener and the indentation of continuation lines. The idiom is to open the string on the same line as its key and continue flush left, as the live nav file's long excerpt does; an author who indents continuation lines gets the indentation in the value. Raw control characters, including NUL, are legal content, since the invariant admits every scalar value, so every string in every API is a pointer and a length and never a NUL-terminated C string; invalid UTF-8 is an error. Line endings are normalized to LF before lexing (3.1), so a string's value is the same on every platform. A later minor version may add an indentation-stripping form; it is not needed to ship.

| Decision | Recommendation | Status |
|---|---|---|
| Hash-run strings | Add `#"..."#` alongside repetition. Repetition stays the common idiom; hash-run covers the case repetition cannot, and the live file needs it once. Without it the embedding invariant above has an exception, so this is a requirement of the invariant rather than a preference. The alternative, hash-run only, is simpler to specify but gives up the `""..""` idiom. | agreed 2026-10-18 |
| Exact closer matching | Yes, for strings and comments alike (finding 8 investigation above). | agreed 2026-10-18 |

### 3.4 One escape mechanism

The explicit `<"...">` wrapper exists to give elements boundaries inside interpolated text. With interpolation moved out of Level 0, the wrapper has no purpose for values, and the live nav file shows what two mechanisms do to an author (two different quote workarounds in one file). `<` and `>` remain in use only for comments and annotations.

| Decision | Recommendation | Status |
|---|---|---|
| Remove explicit `<...>` value forms | Yes. One rule for embedding delimiter characters, applied everywhere. | agreed 2026-10-18 |

### 3.5 Keys and objects

A key is a bare identifier or a string. Identifiers start with a letter or `_` and continue with letters, digits, `_`, `-` and `.`; this matches what the live file already uses (`menu-id`). `.` has no path meaning: `a.b` is one flat key, unlike TOML's dotted keys. Which letters count is review finding 7, discussed in 3.5a. A key that needs anything else is a string: `"first name" "Alice"`. This removes the `=` specifier. A keyword is not an identifier, so `{ true 1 }` is an error and the key is written `"true"`; a bare word is lexed the same way in every position. An identifier may not end with `.`, so prose-like punctuation does not get absorbed into a key. Duplicate keys are an error. Key order is preserved in the tree and the canonical form. The recursive key/value form (`key1 key2 "value"`) is removed; it cannot coexist with bare-word values, and its one use case (`script javascript "body"`) is a nested object or a two-element array.

### 3.5a The identifier alphabet (review finding 7)

The review recommended ASCII-only identifiers and Paul asked why not UTF-8. The question is only about bare identifiers and symbols; a quoted key already admits any string, so nothing is unrepresentable either way.

The usual way to admit Unicode identifiers is UAX #31 (`XID_Start` and `XID_Continue`), which is what Rust, Python and JavaScript do. It has three costs for a data format. It ties the spec to a Unicode version, since the classes grow with each release, so two conformant parsers built against different Unicode versions can disagree on whether a document is valid; it brings classification tables into the C core, which is the largest table the core would otherwise carry; and it raises the normalization question, because `é` as U+00E9 and as `e` plus U+0301 are different scalar sequences and the spec must say whether they are the same key.

There is a table-free rule that gets the benefit without those costs, and it is the direction TOML 1.1 took for bare keys. An identifier may contain any code point at or above U+0080 except a fixed, small list that is stable across Unicode versions: the Unicode whitespace characters (U+00A0, U+1680, U+2000 to U+200A, U+2028, U+2029, U+202F, U+205F, U+3000, U+FEFF), and the format and bidirectional-control characters (U+200B to U+200F, U+202A to U+202E, U+2060 to U+2064, U+2066 to U+2069). ASCII keeps the rule in 3.5, and any admitted non-ASCII code point may also start an identifier. Keys compare by scalar-value sequence with no normalization; the canonical writer never normalizes; a formatter or editor may offer NFC as a tool, outside the spec. Homoglyph confusion (Cyrillic `а` for Latin `a`) is not the format's problem to solve, any more than it is JSON's for quoted keys, though a linter may warn.

**Recommendation:** adopt the table-free rule. It costs the C core a dozen range comparisons, it admits `naïve`, `日本語` and `größe` as bare keys and symbols, it excludes the invisible characters that make Unicode identifiers dangerous, and it depends on no Unicode version.

| Decision | Recommendation | Status |
|---|---|---|
| Identifier alphabet | Table-free Unicode rule above; byte identity; no normalization. | agreed 2026-10-18 |

### 3.6 Sequences

One sequence type, `[ ... ]`, heterogeneous. The homogeneity rule and the separate tuple type exist only to enforce each other; homogeneity is a schema's job. `( )` becomes free for Level 2.

### 3.7 Document

A document is optional annotations followed by exactly one element of any type, as RFC 8259 allows for JSON. The current single-root-collection rule is tighter than it needs to be and complicates the JSON mapping. An empty document (nothing but whitespace and comments) is an error, and so is anything other than whitespace and comments after the root element; a stream of concatenated documents is not a 1.0 feature. A document may begin with a version annotation, `<! xfer 1 !>`, and a parser rejects a major version it does not know.

| Decision | Recommendation | Status |
|---|---|---|
| Any element at root | Yes, for JSON parity. Paul previously settled on collection-only; this proposes reopening it. | agreed 2026-10-18 |

### 3.8 Annotations (Level 1)

`<! name value !>` attaches to the next element: to the key/value pair when it appears inside an object, and to the element otherwise. An annotation with no following element (the last thing in a collection, or at the end of a document) is an error. An annotation's value is a plain Level 0 element and may not itself carry annotations. Several annotations may precede one element and all attach to it. Level 1 defines a closed set: `xfer` (version; optional, and absence means version 1; if present it must be the first token of the document, before any comment or other annotation), `id`, `tag`, and `lang` (what an embedded string contains, for editors). A `schema` annotation is deferred to the schema specification so that 1.0 does not ship a reference to a language that does not exist. There is one parse mode, and an unknown annotation is an error in it; tooling that needs to carry unknown annotations reads the lossless tree, which keeps everything. Annotations are grammar and metadata; nothing in Level 0 or 1 evaluates, reads the environment, or touches the file system.

### 3.9 Level 2 is a separate language

`let`, `if`, dynamic sources, interpolation and anything that computes become a preprocessor language that reads its own documents and writes Level 0. It is a separate specification, a separate library and a separate CLI verb. A Level 0 parser never sees it. Whether it is worth building at all is a question for after 1.0 ships; CUE, Pkl and Jsonnet exist, and the better move may be a converter from one of them.

### 3.10 Lossless tree and canonical form

The parser produces a concrete syntax tree that preserves every byte of the source as trivia (whitespace, comments, delimiter run lengths, number spellings), so a formatter or a structured editor can rewrite one entry and leave the rest untouched. A value model is derived from it for consumers who want data. The canonical serialization is defined to the byte (indentation, spacing, number spelling, minimal run lengths, key order as written) so that two implementations produce identical output, which is what the conformance suite checks.

### 3.11 Diagnostics and limits

Parsing returns a result, never throws. The result is a possibly partial tree containing error nodes, plus diagnostics. A diagnostic has a stable code (`XFER0101` and so on, listed in the spec), a severity, a UTF-8 byte range, a line and column counted in Unicode scalar values (so the C core, the .NET implementation and a UTF-16 language server all agree, and the suite can compare them), and a message. The parser recovers at the next element boundary it can isolate, so a document with several faults reports each one the parser can separate; an unterminated string has no boundary and swallows the rest of the document, and the spec says so. The partial tree is what lets a consumer such as the site's `NavService` keep the entries that parsed. Limits (document size, nesting depth, scalar token length, delimiter run length for quotes, hashes and slashes alike, element count) are normative: the spec states minimums every conformant parser must accept (for example depth 256, run length 64) and defaults; an implementation may raise them and never lower them, so two conformant parsers never disagree on a document within the minimums.

### 3.12 JSON mapping

Defined in both directions. Xfer to JSON: symbols and temporals become strings with an informational diagnostic; integers are emitted as JSON numbers at full precision (RFC 8259 permits it; only JavaScript loses it), with a warning above 2^53 and a `--stringify-big-ints` option for JavaScript consumers; comments and annotations are dropped. JSON to Xfer: lossless, except that JSON permits duplicate keys and XferLang does not, so from-json keeps the last value and reports a diagnostic, which is what every JSON parser does silently. The CLI ships `to-json` and `from-json` from day one, because the quickest way for anyone to trust a new format is to see it round-trip through the one they already use.

### 3.13 Examples of the proposed syntax

These examples use the recommendations above as if every open decision had gone the recommended way. They are illustrations for the design conversation, not the spec.

**A configuration file.** This covers every Level 0 scalar, both collection types, comments, keys with hyphens and keys that need quoting.

```xfer
<! xfer 1 !>
</ Production API configuration. A comment is an element and needs no line ending. />
{
    server {
        host "api.example.com"
        port 8443
        tls true
        timeout 30
        max-connections 100            </ hyphens are legal in bare keys />
        "listen address" "0.0.0.0"     </ any other character: quote the key />
    }
    database {
        primary { host "db1.example.com" port 5432 pool 20 }
        replica { host "db2.example.com" port 5432 read-only true }
        password null                  </ supplied by the deployment, not the file />
    }
    logging {
        level info                     </ a symbol, not a string: maps to an enum />
        destinations [ console file syslog ]
    }
    limits {
        ratio 0.8125                   </ fractional: exact as written />
        avogadro 6.02214076e23
        flags 0b1111_0000              </ integer, binary, with digit separators />
        colour 0xFF5733
        perms 0o755
        big 9_223_372_036_854_775_807  </ no width in the syntax; overflow is the consumer's diagnostic />
    }
    schedule {
        launched 2025-08-01T09:30:00Z  </ bare ISO 8601, lexically distinct from numbers and symbols />
        daily-at 09:30:00
        expires 2026-01-01
        local 2025-08-01T09:30:00+08:00
    }
}
```

**Strings.** Run-length delimiters replace escaping; multi-line content is verbatim.

```xfer
{
    plain       "Hello, world"
    empty       ""
    with-quote  ""He said "hello" and left.""
    many        """Content with "" inside"""
    odd-runs    "Content with "" inside"     </ legal under exact matching: a run of 2 is not a run of 1; the canonical writer would use """ />
    ends-quote  #"say "hi""#                 </ ends with a quote, so repetition cannot write it; hash-run can />
    multiline   "First line.
Second line, indentation kept as written.
    Third line, indented four spaces in the value."
    html        ""<a href="/x" class="y">link</a>""
    regex       "^[a-z]+\d*$"              </ backslash is an ordinary character; there are no escapes />
}
```

The one thing pure repetition cannot write is a string whose content begins or ends with a quote character; the hash-run form covers it, and composes the same way:

```xfer
{
    quoted   #""To be, or not to be""#            </ content begins and ends with a quote />
    deeper   ##"a "# inside, and ends with #"##   </ content contains "# so the run is two hashes />
}
```

**Embedding at depth.** This is the founding use case. A JavaScript source fragment, an XferLang document that holds it, and an XferLang document that holds that one. The JavaScript is byte-identical at every level, and each level adds one quote to its delimiter run.

```javascript
function greet(name) { console.log("Hello, " + name + "!"); }
```

```xfer
</ Level 1: the script in a document />
{
    name "greeter"
    script ""function greet(name) { console.log("Hello, " + name + "!"); }""
}
```

```xfer
</ Level 2: that document in another document />
{
    package "widgets"
    modules [
        """{
    name "greeter"
    script ""function greet(name) { console.log("Hello, " + name + "!"); }""
}"""
    ]
}
```

```xfer
</ Level 3: and that one in a third />
{
    bundle """"{
    package "widgets"
    modules [
        """{
    name "greeter"
    script ""function greet(name) { console.log("Hello, " + name + "!"); }""
}"""
    ]
}""""
}
```

The same three levels in JSON would read `\"Hello, \"`, then `\\\"Hello, \\\"`, then `\\\\\\\"Hello, \\\\\\\"`. The canonical writer computes the minimal run automatically: it scans the content for its longest quote run and uses one more. Comments compose the same way with `</ />`, `<// //>`, `</// ///>`, so a document with comments can be commented out whole.

An optional Level 1 annotation, `<! lang javascript !>` on a string, tells editors what the content is so they can highlight it, and tells a structured editor that the string may be opened as a document of that language. It carries no meaning for the parser.

**Symbols versus strings versus keywords.**

```xfer
{
    enabled true          </ boolean keyword />
    missing null          </ null keyword />
    mode dark             </ symbol: a bare word that is not a keyword />
    label "dark"          </ string />
    answer 42             </ integer />
    code "42"             </ string />
    treu                  </ a symbol, legal here, which a schema can reject;
                             never silently a boolean or a string />
}
```

The only words with meaning are `true`, `false` and `null`. Nothing else is ever coerced, so `NO`, `yes`, `on` and `1.10` mean exactly what they say.

**Sequences.** One type, heterogeneous, nested freely.

```xfer
{
    ports    [ 80 443 8080 ]
    point    [ 42.3601 -71.0589 ]
    win      [ 0.9 0.9 ]                  </ the live nav file writes this as the string "0.9,0.9" today />
    row      [ "Alice" 30 true null 2025-08-01 ]
    matrix   [ [ 1 2 ] [ 3 4 ] ]
    records  [
        { name "Alice" age 30 }
        { name "Bob"   age 25 }
    ]
    empty    [ ]
}
```

**Any element at the document root.** All of these are complete documents.

```xfer
{ name "Alice" }
```

```xfer
[ 1 2 3 ]
```

```xfer
"just a string"
```

```xfer
42
```

**Block comments and disabling entries.** Block comments nest by lengthening the run, and may wrap whole elements.

```xfer
{
    </ A block comment. It may span
       several lines. />
    nav [
        { slug "home"  title "Home" }
        </ { slug "draft" title "Not yet" } />          </ the entry to the left is switched off />
        <// A comment that contains </ a comment marker /> inside it. //>
        { slug "about" title "About" }
    ]
}
```

Comments survive a round trip through the lossless tree and are absent from the value model.

**Annotations (Level 1).** They attach to the next element and never evaluate.

```xfer
<! xfer 1 !>
<! schema "sitenav.xfs" !>
{
    <! id "main-menu" !>
    <! tag "navigation" !>
    menu [
        <! tag "experimental" !>
        { title "Beta features" url "/beta" }
    ]
}
```

**The site navigation, migrated.** This is the live file's shape under the proposal. The head annotation changes, `<"...">` strings lose their angle brackets, `~true` becomes `true`, and the one title written `<'"..."'>` becomes a hash-run string. Window shapes could stay as strings or become sequences, as the engine prefers.

```xfer
<! xfer 1 !>
{
    slug "home"
    title "Home"
    url "/home"
    description "Paul's Thoughts on Computing and Other Stuff"
    </ The menu bar logo menu (Architecture/site-menu-design.md). Each entry
       is a title, the first the logo's own. />
    menu [
        {
            menu-id "site"
            title "Parks Computing"
            icon "/favicon-32x32.png"
            nav [
                { title "Home" url "/" }
                { slug "about" title "About Parks Computing" }
                { separator true }
            ]
        }
        {
            menu-id "view"
            title "View"
            nav [
                { title "Windowed view" command view args { value window } }
                { title "Classic view"  command view args { value classic } }
                { separator true when window }
                { title "Theme" nav [
                    { title "Light" command theme args { value light } }
                    { title "Dark"  command theme args { value dark } }
                ] }
            ]
        }
    ]
    nav [
        {
            slug "articles"
            title "Articles"
            feed true
            nav [
                {
                    slug "so-much-more-exotic"
                    </ This title begins and ends with a quote character, which pure
                       repetition cannot express (see 3.3). It is the one entry in the
                       live file that needs the hash-run form, written here that way: />
                    title #""That sounds so much more exotic than here!""#
                    description "Exotic is just a matter of perspective"
                    tags [ travel ]
                }
                {
                    slug "barcode-flash-cards"
                    excerpt "I'm getting back into the retail point-of-sale space, and I needed a way to brush up on my skills at identifying barcode symbologies on sight, so I cooked up
a little web app that generates barcode flash cards."
                    tags [ "web apps" barcodes ]
                }
                </ { slug "sudoku" tags [ "web apps" games ] } />
            ]
        }
        {
            slug "theme-studio"
            icon "/images/icons/theme-studio.svg"
            win [ 0.9 0.9 ]
            title "PUDL Theme Studio"
        }
    ]
}
```

Note that `command view` and `value window` have become symbols where the live file has strings. Either is legal; the engine decides which it wants, and a schema can require one.

**Canonical form.** The canonical writer produces one spelling per document: four-space indentation, one element per line inside collections, minimal delimiter runs, numbers as written, keys in source order. Given the input `{a 1   b [1 2]  c "x"}` the canonical output is

```xfer
{
    a 1
    b [
        1
        2
    ]
    c "x"
}
```

and the conformance suite checks that every implementation produces exactly those bytes.

**JSON mapping.** The document

```xfer
{
    name "Alice"
    level admin
    joined 2025-08-01
    id 9007199254740993
    score 1.50
}
```

maps to JSON as

```json
{ "name": "Alice", "level": "admin", "joined": "2025-08-01", "id": 9007199254740993, "score": 1.50 }
```

with two diagnostics: the symbol and the date became strings (informational), and the integer exceeds 2^53, which JavaScript consumers will round (warning; `--stringify-big-ints` emits it as a string instead). The reverse mapping from JSON is lossless apart from duplicate keys, where the last value wins with a diagnostic.

**Diagnostics.** Parsing never throws. The document

```xfer
{
    a 1
    b "unterminated
    c 2
}
```

yields a result whose value is absent and whose diagnostics include

```
XFER0201 error   3:7-5:2   string literal is not terminated (opened with a run of 1 quote at 3:7)
```

With recovery, the parser resynchronizes at the next element boundary where it can, so a document with several faults reports all of them.

**What a Level 2 document looks like, for contrast.** This is outside Level 0 and is shown only to make the separation concrete. A Level 2 preprocessor would read

```xfer2
<! let host "db.example.com" !>
<! if env.PRODUCTION !>
{ database { host _host port 5432 } }
```

and write the Level 0 document

```xfer
{ database { host "db.example.com" port 5432 } }
```

Under the proposal's own identifier rule, `_host` is a legal symbol, so without a guard the first document is *valid* Level 0 with `host` set to the symbol `_host`, which is worse than an error. Level 2 documents therefore carry a mandatory head annotation that a Level 0 parser rejects (for example `<! xfer-template 1 !>`) and use their own file extension. A Level 0 parser given a Level 2 document stops at the head annotation; it never evaluates anything. (Review round 1, finding 10.)

## 4. What is lost, honestly

Dropping the sigils removes the one thing that made an XferLang document recognizable at a glance, and an author who liked `~true` and `*19.99` will miss them. The language becomes closer to TOML and KDL in appearance. What remains distinctive is the combination: whitespace separation, run-length delimiters with no escapes anywhere, nesting comments, symbols as a type, and a lossless tree, all under a spec and suite. That is a smaller claim than "a universal data and scripting format", and it is one the language can keep.

The comparison with KDL should be made plainly, because the design has converged on something KDL-adjacent. KDL 2.0 has composing raw strings, no commas, no indentation significance, comments, a specification with a test suite, and implementations in a dozen languages. XferLang's remaining differences are that it has no escape sequences at all rather than as a second mode, that its strings are verbatim rather than dedented, that its comments nest, and that its data model is JSON's rather than a node tree. Those differences are real and they are modest. The other reason to build rather than adopt is ownership: a format used across the whole portfolio, under a spec Paul controls, with the .NET implementation shaped for the portfolio's needs. That reason is legitimate, and the decision to proceed should be taken with it stated rather than implied.

## 4a. Positioning: why a developer would choose it

**The hook is the embedding invariant** (3.3). Any content goes into a document unchanged, at any depth, for the cost of one delimiter character per level. No other format in the space leads with this, and every escape-based format fails it by construction: JSON inside JSON doubles the backslashes per layer, JSON inside YAML strings is the daily misery of Kubernetes ConfigMaps and CI pipelines, and SQL, regular expressions, HTML and shell inside any of them need a rewrite that stacks. The mechanism itself is not new: Rust and KDL raw strings, Lua long brackets, PostgreSQL dollar quoting, shell heredocs and Markdown fences all compose the same way, and YAML block scalars compose by indenting. What is distinctive is set out in 4b: in XferLang it is the only string syntax, it is verbatim, and it applies to keys and comments as well. The demonstration is section 3.13's three-level nesting next to its JSON equivalent, and the one-line pitch is "embed anything, at any depth, byte for byte." The audiences who feel this pain daily are the people writing infrastructure templates, CI pipelines, webhook payloads, prompt templates, test fixtures with embedded documents, and anything with code inside data.

Beyond that single hook, feature by feature, the rebooted language has no construct that no other format has. TOML has comments and typed dates. KDL has raw strings, no commas, and a slash-dash that comments out a whole node. JSON5 has comments and relaxed quoting. Formats are not chosen from feature tables anyway; JSON won on ubiquity, YAML on Rails and Kubernetes, TOML on Cargo and pyproject, and KDL is still waiting for its flagship. A format wins by filling a positioning gap and by having a consumer people can point at.

**The gap, as a second line.** Nobody holds "JSON's data model, exactly, with a syntax people can write and the types JSON forgot." TOML is not JSON-shaped (tables and arrays of tables make deep nesting awkward). KDL is not JSON-shaped (nodes with arguments and properties, so a JSON round trip needs conventions). YAML is JSON-shaped but heuristic and indentation-bound. JSON5 and HJSON are JSON-shaped but keep escape sequences and have no types. XferLang after the reboot is objects and sequences, one to one with JSON, with real integers, exact fractions, dates, symbols, no commas and no escapes. The one-line pitch is "JSON for people, with the types it forgot."

**The hook that gets a developer to try it** is the JSON off-ramp: lossless from JSON, defined and diagnosed to JSON. `xfer to-json` and `xfer from-json` are in the CLI from the first release, the mapping is in the spec, and every existing tool keeps consuming JSON. Adoption carries no ecosystem risk: write XferLang at the edge where humans edit, emit JSON where machines read. Jsonnet was adopted this way.

**The features that keep them** are the ones the live nav file already demonstrates. No commas means reordering is moving blocks. No escape sequences anywhere means prose, HTML, regular expressions and code paste in unchanged. Comments that nest and can wrap a whole entry mean notes and temporary removals without deleting. A lossless tree with a stable editing API means a tool can rewrite one key and leave the author's comments and formatting alone, which is the thing `toml_edit` and `ruamel.yaml` exist to approximate and most ecosystems lack. Symbols mean enum values without quotes that a schema can check.

**The bet that could make it distinctive** is agent authorship. Language models routinely break YAML indentation and JSON escaping, and both faults are silent until a parser rejects the file. A format with no indentation significance, no escapes, no commas, explicit literals, a one-page grammar that fits in a prompt, a canonical form, and diagnostics with stable codes a model can act on is a better target for generated data. Nobody is designing for this audience. The claim must be measured before it is made: generate the same structures in JSON, YAML, TOML and XferLang across several models and publish the error rates. If the numbers hold, it is the headline; if they do not, it is dropped.

**The credibility feature** is the hardened, dependency-free C parser with a public fuzzing record and a conformance suite other implementations pass. It does not attract users, and it is what stops them leaving.

**Distribution** is the portfolio. PUDL applet manifests are JSON today, the site navigation is XferLang, and Tela, Andoneer, Dinah and Overt all have configuration and content that a reader of their repositories will see. Making XferLang the native format across them puts the comparison with JSON and YAML in public code where the format can be judged on real files, which is the only kind of advocacy that has ever worked for a data format.

**What not to claim:** scripting, speed, and universality. CUE and Pkl own the first, simdjson and yyjson own the second, and JSON owns the third.

## 4b. Does the escape-free design beat the others, and is it compact?

Both questions were checked rather than asserted. The documents and byte counts are reproducible from the descriptions below.

### 4b.1 How the escape-free mechanism compares

Run-length and tagged delimiters exist in Rust (`r#"..."#`), KDL 2.0 (`#"..."#`), Lua (`[==[ ... ]==]`), C++ (`R"tag(...)tag"`), PostgreSQL (`$tag$...$tag$`), shell and Ruby heredocs, and Markdown fences. All of them compose across nesting levels. YAML block scalars (`|`) compose too, with each level indenting the content further. So the mechanism is well understood and the claim cannot be that XferLang invented it. The claims that hold up are these.

**It is the only string syntax.** Every other data format keeps an escaped string form, and it is the default: KDL's `"..."` has `\n` and `\"`, TOML's basic strings do, YAML's double-quoted scalars do, JSON has nothing else. In XferLang there is no mode in which escaping can occur, so a writer never chooses, a generator never implements an escaper, and a reader never decodes. NestedText also has no escape sequences, but it has no types either (every value is a string) and depends on indentation. Among typed data formats, XferLang would be alone in having no escape sequences anywhere in the grammar; verify again before claiming it in public.

**It is verbatim.** KDL 2.0 multi-line strings dedent to the closing quote's indentation, TOML multi-line strings drop the newline after the opener, and YAML block scalars strip indentation and apply chomping rules to trailing newlines. Each of those is a rewrite, and each has rules an author must know. XferLang's content is the bytes between the delimiters, with no exceptions. The cost is that embedded multi-line content cannot be indented to match its surroundings without the indentation becoming part of the value, so a deeply embedded document is written flush left or carries its indentation. Dedenting formats look tidier in that case; XferLang is exact.

**It is uniform.** Keys that need quoting are strings and follow the same rule. Comments follow the same rule and therefore nest, so a region that contains comments can be commented out whole. No other format in the space has nesting comments (KDL's slash-dash comments out one node; `/* */` does not nest in any of them).

**Other formats fail the invariant at identifiable points.** TOML literal strings cannot contain `'''` at all, so content with three consecutive apostrophes forces a switch to escaped strings. YAML block scalars cannot hold a line indented less than the block, and a first line with leading spaces needs an indentation indicator. XML CDATA cannot contain `]]>`. JSON and JSON5 escape everything. KDL raw strings have no such point, and are the closest competitor on this axis; the difference with KDL is the three paragraphs above.

**For parsers** the benefit is concrete. Lexing a string is "count the run, find the matching closer", with no escape state machine and no decode pass, so the string's value is a slice of the input and can be returned without copying. There are fewer states for a fuzzer to explore and no decoder to get wrong, which is where a large share of historical JSON and YAML parser faults have been. A generator is one line: scan the content for its longest delimiter run and use one more.

### 4b.2 Compactness, measured

The same server configuration (five sections, seventeen values, one list) written idiomatically in each format with two-space indentation:

| Format | Pretty bytes | Relative to XferLang |
|---|---|---|
| YAML | 374 | 7% smaller |
| KDL | 373 | 7% smaller |
| XferLang (proposed) | 402 | |
| TOML | 412 | 2% larger |
| JSON | 497 | 24% larger |

Minimal forms with no optional whitespace: XferLang 321 bytes, JSON 389 bytes (21% larger). TOML and YAML have no minimal form; KDL's would be close to XferLang's.

XferLang's seven percent against YAML and KDL is the cost of braces around nested objects and quotes around string values; YAML has neither and KDL puts small objects on one line as properties. XferLang's saving against JSON is the removal of commas, colons and quoted keys. So the honest statement is that XferLang is a little smaller than TOML, a fifth smaller than JSON, and a few percent larger than YAML and KDL for ordinary configuration.

The picture changes under nesting. The same document, a module with a name and a JavaScript string, wrapped in another document at each level, in minimal form:

| Nesting depth | XferLang | JSON |
|---|---|---|
| 1 | 91 | 95 |
| 2 | 108 | 124 |
| 3 | 127 | 173 |
| 4 | 148 | 262 |
| 5 | 171 | 431 |

XferLang grows by a constant per level (the wrapper plus two delimiter characters). JSON's escape sequences double at every level, so its growth is exponential, and the quote before `Hello` is preceded by 2ⁿ−1 backslashes at depth n, which is thirty-one at depth five. YAML and TOML sit between: YAML's growth is linear in the number of content lines (each level indents every line) and TOML's literal strings hold this particular content until one contains `'''`. For a two-level pretty document YAML (126 bytes), TOML (130) and KDL (132) are all smaller than XferLang (146), because their inner documents are brace-free; XferLang's advantage is not size at shallow depth but that the inner bytes are untouched and the growth stays constant at any depth.

**Claim to make:** as compact as or smaller than JSON and TOML, within a few percent of YAML and KDL, and the only one whose size grows linearly under nesting with the embedded content unchanged. **Claim not to make:** the most compact format.

## 5. Repositories and archive

### 5.1 Archive the current repository

1. Tag the current head `v0.15-final`.
2. Replace the README's introduction with a short archive notice pointing at the new repositories and at `REBOOT_ASSESSMENT.md`, and leave the rest intact for reference.
3. Rename the GitHub repository from `Xfer` to `xfer-legacy`. GitHub redirects the old URL, so existing links and the NuGet project link keep working.
4. Mark it archived (read-only) on GitHub once the site has moved to 1.0.
5. On NuGet, leave `ParksComputing.Xfer.Lang` 0.15.x in place and mark it deprecated with a message pointing at the new package. The site stays pinned to 0.15.25 until the managed 1.0 passes the suite and `sitenav.xfer` has been migrated; the migration is mechanical: `<! document { xferlang "0.15" } !>` becomes `<! xfer 1 !>`, every `~true` becomes `true`, every `<"...">` becomes `"..."` (or `""...""` where the text contains a quote), and the one title written `<'"..."'>`, whose text begins and ends with a quote, becomes `#"..."#`.
6. Copy `REBOOT_ASSESSMENT.md` and this document into the new spec repository under `history/`; leave the other whitepapers behind.

### 5.2 New repositories

- **`xferlang`** (the spec): `spec/` (grammar, semantics, canonical form, JSON mapping, diagnostics), `tests/valid/`, `tests/invalid/`, `history/`. Versioned by semver on the spec; 1.0.0 is the first frozen release. Every implementation vendors or submodules this repository's `tests/`.
- **`libxfer`** (C core): C11, no dependencies beyond the standard library and none in freestanding mode, CMake, builds for MSVC, clang and gcc on Windows, Linux and macOS, plus wasm32. Ships `xfer` CLI with `validate`, `fmt`, `to-json`, `from-json`. CI runs the suite, sanitizers and the fuzzer on every push.
- **`xfer-dotnet`** (managed): one library, net8.0 and netstandard2.0, no native code, the object mapper carried forward from `XferConvert`, and an API surface small enough to document on one page. New package id `ParksComputing.XferLang`, 1.0.0 when it passes the suite.
- **`xfer-vscode`**: TextMate grammar regenerated from the spec, and a language server over `xfer-dotnet` or `libxfer`.

### 5.3 Order of work

1. The blocking findings of `REBOOT_REVIEW_1.md` ruled on and recorded in section 7.
2. Spec draft.
3. Conformance suite written against the spec, including negative cases for every diagnostic code and every limit. The probe failures from the assessment (`?,` hang, `""`, `</` inside a string, underscore references) become the first negative tests.
4. `xfer-dotnet` lexer, parser, CST, value model, canonical writer, developed with the spec and suite until the suite is green; spec faults found on the way are fixed in the spec and the suite, not worked around.
5. Migrate `sitenav.xfer` and `admin-menu.xfer`; the site is the first production consumer.
6. Freeze spec 1.0.0; publish `xfer-dotnet` 1.0.0.
7. `libxfer` against the frozen suite, differentially tested against `xfer-dotnet`, with sanitizers and the fuzzer from the first commit; publish; archive the legacy repository.

## 6. Industrial-strength from the first line

The practices that distinguish a library people depend on from one they try, applied from the first commit rather than retrofitted:

- Spec before code; suite before parser; no feature without a test in the suite.
- Sanitizers and the fuzzer in CI, with any finding a release blocker.
- No recursion, no undefined behavior, bounds-checked input, arena allocation, mandatory limits.
- A frozen ABI with a written stability policy, semver on spec and libraries, a changelog, and a security policy with a disclosure address.
- Deterministic canonical output verified across implementations.
- Benchmarks tracked per commit so a regression is visible when it lands.
- Coverage measured and held above a stated floor on the C core.
- Documentation written for someone who has never seen the format, with the JSON mapping on the first page.

## 6a. Open items from review round 1

`REBOOT_REVIEW_1.md` found nine blocking gaps, all lexical. Paul ruled on all nine on 2026-10-18 (section 7), and the rulings are written into sections 3.1, 3.2, 3.3, 3.5, 3.5a and 3.8. Round 1 is closed; see `REBOOT_REVIEW_2.md` for the next round.

## 7. Decisions recorded

This section is for the rulings as Paul makes them. Each entry gives the date, the decision and the reason.

- **2026-10-18. No line-end comments.** Line endings carry no meaning outside string literals; a document can be collapsed to one line or re-flowed without changing its value, and a comment that ran to end of line would break that. The only comment form is `</ ... />`, which nests by lengthening the run. The whitespace invariant in 3.1 records the principle, and the conformance suite tests it.
- **2026-10-18. Token separation (review finding 1).** A scalar token must be followed by whitespace, a bracket, a comment or annotation opener, or end of input. Brackets and `<` are the only self-delimiting tokens. Reason: without it two parsers can disagree on `"a""b"` and `1-2`, and the current parser's closing-by-accident fault recurs.
- **2026-10-18. Whitespace set and invariant (finding 2).** Whitespace is space, tab, LF and FF; no Unicode spaces. The invariant is stated in terms of self-delimiting tokens. Reason: the separator must be visible and the rule must be mechanical.
- **2026-10-18. Line-ending normalization (finding 3).** CRLF and lone CR are normalized to LF before lexing; the embedding invariant is stated over LF text; a lone CR is transported encoded. Reason: verbatim strings would otherwise change value under version control.
- **2026-10-18. Symbols kept; displaced-error cost accepted (finding 4).** The odd-length object diagnostic must say a value is missing and list the inferred pairings. Reason: the config idiom is worth it and a schema is the real defense.
- **2026-10-18. `inf`, `-inf`, `nan` reserved (finding 5).** They are fractional literals, as in TOML, so a generator always has a representation. They have no JSON mapping and `to-json` reports an error for them.
- **2026-10-18. Numeric and temporal lexemes pinned (finding 6).** As written in 3.2: RFC 3339 productions only; `_` between digits only; no `+`; no leading zeros; any exponent makes a literal fractional; lowercase radix prefixes.
- **2026-10-18. Identifier alphabet (finding 7).** The table-free Unicode rule in 3.5a: ASCII letters, digits, `_`, `-`, `.` plus any code point at or above U+0080 except the listed Unicode spaces and format/bidi controls; keys compare by scalar sequence; no normalization. Reason: admits native-language keys without a Unicode-version dependency, classification tables, or a normalization rule.
- **2026-10-18. Exact closer matching (finding 8).** The closer of a repetition string or a comment is the first maximal run of exactly n delimiter characters; hash-run closers are a quote followed by a maximal run of exactly m hashes. Repetition cannot write content that begins or ends with a quote; hash-run has no exception; the canonical writer uses the choice rule in 3.3. Reason: a stray delimiter fails loudly instead of silently changing a value, and the rule matches Rust, Lua and C++.
- **2026-10-18. Section 3 decision tables confirmed.** Drop all type sigils; one fractional type in syntax; symbols as a type; bare RFC 3339 temporals; hash-run strings alongside repetition; remove the explicit `<...>` value forms; any element at the document root. Reason: each is argued in its section, and the nine review rulings presuppose them. The proposal is closed; the spec is next.
- **2026-10-18. Annotations (finding 9).** Attach to the pair inside objects and to the element elsewhere; a trailing annotation is an error; no nested annotations; `xfer` head optional with absence meaning 1; one parse mode in which an unknown annotation is an error.

Decisions made after the proposal closed, during the first implementation (this log is the record until the freeze):

- **2026-10-18. A run of exactly two quotes is always the empty string.** The first lexer found that the rule in 3.3 ("two quotes followed by a non-quote is the empty string") contradicts the `""..""` examples throughout the proposal, the specification and the suite: by that rule `""He said "hello" and left.""` is the empty string followed by `He`. Of three fixes, Paul chose this one. A string whose content contains a quote opens with one quote or with three or more, as in Python, C#, Kotlin, Swift and Java, so `""` is never ambiguous and a slip fails loudly. The alternatives were to make `""` empty only before a terminator (which keeps `""..""` but parses `[ ""` / `{ a "x" }` / `"" ]` silently as three values, and needs another canonical-writer special case) and to have no special case, writing the empty string `#""#`. The embedding invariant still holds at one delimiter character per level.
- **2026-10-19. Callers may lower the limits.** Until now the implementation raised any limit below the specification's minimum back to the minimum, so a server parsing untrusted input could not refuse a large document, and a hostile large document could exhaust a JavaScript engine's heap (the lossless tree costs about 100 bytes per input byte). Paul ruled that a caller may lower (or raise) each limit as an explicit option, that the defaults stay at the minimums, and that the specification says the minimums bind a parser's default configuration, not a deployment's policy (spec 3.4). A parser so configured is not conformant for documents beyond the limits its caller chose.
- **2026-10-19. An annotation after the root is content after the root.** Spec 7 gave "last in a document" as an example of an annotation with no target (`XFER0502`), while the suite case `502-trailing-annotation-at-end` (`1 <! tag x !>`) required `XFER0402`. Paul ruled for the suite: an annotation after the root is `XFER0402`; `XFER0502` is for an annotation last in a collection, with nothing after it in a document that has no element, or between a key and its value (spec 7).
- **2026-10-19. A JSON string that XferLang cannot hold is reported, not converted.** A string with U+000D (from `\r`) or a lone surrogate cannot be written as XferLang, because a document is normalized before it is read and neither would survive. The claim that the mapping from JSON is lossless therefore has an exception. Paul approved the exception: the converter reports it (`XFER0907`, an error, for a value or a key) and never substitutes another character; malformed JSON is `XFER0906` (spec 8.2, 10).
- **2026-10-18. The document-size and token-length minimums are 2^28−1 bytes.** The draft's 2^31−1 cannot be met by a JavaScript engine (V8 holds about 2^29 UTF-16 units) or by .NET strings (about 2^30 characters), so no JavaScript or .NET implementation could have been conformant. At 2^28−1 (256 MiB) every implementation can be; C and others may accept more. Found by the cold review of the JavaScript implementation's step 3.
- **2026-10-19. JSON nested past the depth limit is refused.** Converting such JSON would give a value that cannot be written within the depth limit, so `XFER0602` is reported and no value is produced (spec 8.2, suite case `828`). Paul accepted the recommendation.
- **2026-10-19. A canonical writer may refuse a value nested past the depth limit.** Indentation makes canonical output grow with the square of the depth, so a short, deeply nested document can have output larger than a string can hold. The refusal must be reported as an error and not as a platform exception (spec 6). Paul accepted the recommendation.
- **2026-10-19. A writer refuses a value no document could hold.** Spec 9 lists what a value built by a program may not contain (U+000D, a lone surrogate, an invalid symbol, a mismatched number or temporal spelling, an annotation other than a scalar `id`, `tag` or `lang`, a repeated `id`), and requires a refusal and not a repair, so every implementation writes the same bytes or refuses the same input. Paul ruled yes on the recommendation.
- **2026-10-19. Invalid UTF-8 is reported where it is, when the platform can say.** Spec 9: `XFER0701` SHOULD be at the first invalid byte and MAY be at the origin when the decoder cannot locate it (the JavaScript decoder cannot; .NET can). The suite keeps `*` for these cases. Paul ruled yes on the recommendation.
- **2026-10-19. Object mappers share a code range.** Spec 9 and 10: `XFER0801` wrong kind, `XFER0802` unknown key when refused, `XFER0803` required member missing, `XFER0804` two keys for one member; `XFER0303` stays for a value the target cannot hold exactly. Raised by the xfer-dotnet design. Paul ruled yes on the recommendation.
- **2026-10-20. A JSON converter may refuse text past the document size limit.** An indented form grows with the depth, as canonical output does: a 2.2 MB document produced about 550 million characters of indented JSON, more than a JavaScript string holds, so xfer-js failed where xfer-dotnet succeeded. Spec 8.1 now gives the converter the canonical writer's permission. The refusal is `XFER0601` with no text, and never a platform exception. Raised by the xfer-dotnet step 9 review. Paul ruled yes on the recommendation.
- **2026-10-19. xfer-dotnet targets `net10.0` only, with C# 14.** .NET 8 and 9 leave support on 2026-11-10, before the package ships. Paul ruled.

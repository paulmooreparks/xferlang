# XferLang Specification, version 1.0, draft 1

Status: draft. This document and `tests/` are the contract. Where this prose and `xferlang.abnf` disagree, the disagreement is a defect to be fixed in both; where either disagrees with a conformance case, the case is the defect, because cases are written from the rules.

The words MUST, MUST NOT, SHOULD and MAY are used as RFC 2119 defines them.

## 1. Scope

XferLang is a text format for structured data. A document denotes one value. Values are objects (ordered key/value pairs), sequences (ordered values), strings, numbers, temporals, booleans, null and symbols. The format is designed so that a person or a program can write it without escaping anything, so that line breaks and indentation never carry meaning, and so that every value's type is stated by its spelling and never guessed.

This specification defines Level 0, the data format, and Level 1, a closed set of annotations that carry metadata and never evaluate. Anything that computes (bindings, conditionals, interpolation, external values) is outside this specification; a document that needs it is written in a separate language that produces XferLang.

## 2. Invariants

Three properties hold for every conformant implementation. They are stated first because every later rule serves one of them, and the conformance suite tests each directly.

**The embedding invariant.** For any sequence of Unicode scalar values *s* that contains no U+000D, there is a string literal whose content is exactly *s*, and the scalar values of *s* appear in the literal unchanged and in order. Because the rule is the same at every level, a document that contains a document that contains a string holds the innermost content unchanged, at a cost of one delimiter character per level.

**The whitespace invariant.** Outside string literals and comments, whitespace carries no meaning beyond separating tokens. Whitespace between a scalar token and a following scalar token is mandatory and may be any non-empty run; whitespace adjacent to a self-delimiting token is optional. Replacing any whitespace run with any other run permitted by these rules does not change the document's value. In particular, line breaks never carry meaning, and no construct runs to the end of a line.

**The explicitness invariant.** Every value's type is determined by its spelling alone. No value is ever coerced to another type by its content, position or neighbours. The only bare words with meaning are the five keywords in section 4.7.

## 3. Text

### 3.1 Encoding

A document is a sequence of bytes that MUST be valid UTF-8. A leading U+FEFF (byte order mark) MUST be accepted and removed. Any other encoding is outside this specification. Invalid UTF-8 is an error (`XFER0701`).

### 3.2 Line-ending normalization

Before lexing, every sequence U+000D U+000A and every U+000D not followed by U+000A MUST be replaced by U+000A. All later rules, including the embedding invariant, apply to the normalized text. A string value therefore never contains U+000D, and a document means the same thing whatever a version-control system or editor has done to its line endings.

### 3.3 Whitespace

Whitespace is U+0020 (space), U+0009 (tab), U+000A (line feed) and U+000C (form feed). No other code point is whitespace. Unicode space characters outside this set are not whitespace; where they are permitted at all (inside strings, comments and identifiers) they are ordinary content.

### 3.4 Limits

Every conformant parser MUST accept any document within the following minimums, and MAY accept larger. The spec fixes minimums so that two conformant parsers never disagree on a document within them.

| Limit | Minimum a parser MUST accept | Diagnostic when exceeded |
|---|---|---|
| Document size | 2^31 − 1 bytes | `XFER0601` |
| Nesting depth (objects and sequences) | 256 | `XFER0602` |
| Scalar token length | 2^31 − 1 bytes | `XFER0603` |
| Delimiter run length (quotes, hashes, slashes) | 64 | `XFER0604` |
| Elements in a document | 2^31 − 1 | `XFER0605` |

Nesting depth counts every open object or sequence including the root, so `[[1]]` has depth 2 and a document of 256 nested sequences MUST be accepted while one of 257 MAY be rejected by a parser at the minimum. Exceeding a limit is a diagnostic, never a crash, an allocation failure or a stack overflow. Implementations SHOULD expose the limits as options with the minimums as defaults.

## 4. Lexical structure

The normalized text is divided into tokens and trivia. Trivia is whitespace and comments and is discarded from the value but kept by lossless tools. Tokens are **self-delimiting** or **scalar**.

The self-delimiting tokens are `{`, `}`, `[`, `]`, the `<` that opens a comment or annotation, and the `!>` that closes an annotation. Every other token is scalar.

**Separation rule.** A scalar token MUST be followed by whitespace, a comment, a self-delimiting token, or end of input (`XFER0101`). So `1-2` is an error (two scalars with nothing between them), `"a"1` is an error, and `{a 1 b [1 2]}` is well formed because every omitted space is beside a bracket. Note that `"a""b"` is not a separation case at all: under exact run matching (4.4) the run of two quotes is content, and the whole is the single string `a""b`.

A `<` not followed by `/` or `!` is an error (`XFER0102`). A `#` not followed by `#` or `"` is an error (`XFER0202`). Any other character that cannot begin a token, including `(`, `)`, `~`, `?`, `+` and a leading `.`, is an error (`XFER0103`).

### 4.1 Comments

A comment opens with `<` followed by a run of one or more `/`, and closes at the first maximal run of exactly the same number of `/` that is immediately followed by `>`. A run of a different length is comment content. Comments may appear anywhere trivia may and count as whitespace for the separation rule. Comments do not nest syntactically; a comment that must contain `/>` is written with a longer run (`<// ... //>`), which is how a region containing comments is commented out. A comment's content may not begin or end with `/` (a leading slash is absorbed into the opening run; a trailing one merges with the closing run); write a space. An unterminated comment is an error (`XFER0104`).

```xfer
</ a comment />
<// a comment containing </ a comment /> //>
```

### 4.2 Keywords

The keywords are `true`, `false`, `null`, `inf` and `nan`, together with `-inf`, which is lexed as a number (4.5). Keywords are case-sensitive. A bare word that differs from a keyword only in case is a symbol. A keyword is never an identifier, in any position.

### 4.3 Identifiers and symbols

An identifier begins with an ASCII letter, `_`, or an admitted non-ASCII scalar value, and continues with ASCII letters, digits, `_`, `-`, `.`, or admitted non-ASCII scalar values. It MUST NOT end with `.` (`XFER0105`). An admitted non-ASCII scalar value is any scalar value at or above U+0080 except the Unicode space characters (U+00A0, U+1680, U+2000 to U+200A, U+2028, U+2029, U+202F, U+205F, U+3000, U+FEFF) and the format and bidirectional controls (U+200B to U+200F, U+202A to U+202E, U+2060 to U+2064, U+2066 to U+2069). An identifier that spells a keyword is the keyword.

Identifiers compare by scalar-value sequence. No normalization is applied; `é` as U+00E9 and as `e` U+0301 are different identifiers. Implementations MUST NOT normalize.

An identifier in value position is a **symbol**. A symbol is a value of its own type; it is never a string, a boolean or a number. `.` and `-` have no meaning inside an identifier: `a.b` is one identifier, not a path, and `v1.2.3` and `read-only` are symbols.

### 4.4 Strings

A string has two spellings. Both yield a sequence of scalar values taken verbatim from the text between the delimiters: there are no escape sequences, and nothing between the delimiters is interpreted.

**Repetition.** The opening delimiter is a maximal run of n ≥ 1 `"`. The closing delimiter is the first following maximal run of exactly n `"`. A run of any other length is content. Consequently the content may contain quote runs of any length other than n, and may not begin or end with `"` (a leading quote would be part of the opening run; a trailing quote would merge with the closing run into a run that is not n). One special case: exactly two `"` followed by anything other than `"` is the empty string.

```xfer
"plain"
""
""He said "hello" and left.""
"""Content with "" inside"""
"Content with "" inside"
```

The last two are the same string. The fifth is well formed because a run of two is not a run of one; the canonical form (section 6) would write the fourth.

**Hash-run.** The opening delimiter is a run of m ≥ 1 `#` followed by `"`. The closing delimiter is the first following `"` that is followed by a maximal run of exactly m `#`. A `"` followed by a hash run of any other length, or by no hash, is content. The content may begin and end with anything. Hash-run has no exception: every string is representable with m one greater than the longest run of `#` that follows a `"` in the content.

```xfer
#"say "hi""#
#""To be, or not to be""#
##"a "# inside, and ends with #"##
```

An unterminated string is an error (`XFER0201`). A string's content may contain any scalar value, including U+0000 and other control characters; implementations MUST represent strings as length-delimited data, never as NUL-terminated text.

The newline that follows an opening delimiter, and the indentation of every continuation line, are content. The idiom for multi-line text is to open the string on the same line as its key and continue flush left.

### 4.5 Numbers

A number is an **integer** or a **fractional**. The two are different types.

An integer is an optional `-` followed by one of:

- a decimal form: `0`, or a digit `1` to `9` followed by digits;
- a hexadecimal form: `0x` followed by hexadecimal digits of either case;
- an octal form: `0o` followed by digits `0` to `7`;
- a binary form: `0b` followed by `0` or `1`.

A fractional is an optional `-` followed by a decimal integer form, then a fraction (`.` and at least one digit), an exponent (`e` or `E`, optional `+` or `-`, at least one digit), or both; or one of the special values `inf`, `-inf`, `nan`.

In any number a single `_` MAY appear between two digits and nowhere else. A leading zero on a non-zero decimal integer is an error (`XFER0301`), so `0755` never means 755 or 493. `1.`, `1.e5`, `1__0`, `1_`, `1_a`, a `-` followed by neither digits nor `inf`, and a radix prefix with no digits after it are errors (`XFER0302`). The radix prefixes are lowercase only, and an uppercase one (`0XFF`, `0O17`, `0B1`) is also `XFER0302`. `.5` and `+1` are not numbers at all: `.` and `+` cannot begin a token (`XFER0103`, section 4). Any exponent makes a literal fractional: `1e5` is fractional and `100000` is an integer.

`-inf` is a complete token wherever it appears, so `-infinity` is `-inf` followed by a separation error at the second `i`. Without the minus sign, `infinity` is an identifier like any other.

A number's written form has no width. The value is the exact number the text denotes. An implementation MUST keep enough of the lexeme to deliver that value exactly to a consumer whose type can hold it, and MUST report a diagnostic (`XFER0303`) rather than wrap, truncate or silently promote when a consumer's type cannot. Whether a fractional becomes a decimal or a binary floating-point value is the consumer's choice; the implementation MUST say when the conversion is inexact.

### 4.6 Temporals

A temporal is one of three RFC 3339 forms, spelled bare:

- a date, `full-date`: `2025-08-01`;
- a time, `partial-time`: `09:30:00`, `09:30:00.250`;
- a date-time, `full-date "T" partial-time [time-offset]`: `2025-08-01T09:30:00Z`, `2025-08-01T09:30:00+08:00`, and with no offset a local date-time `2025-08-01T09:30:00`.

The separator between date and time is `T` only, and the UTC offset `Z` is uppercase only. RFC 3339's lowercase `t` and `z` and its space separator are excluded, the space because it would split the literal under the separation rule. A temporal takes no sign and no digit separator: `-2025-08-01` is the integer `-2025` followed by a separation error. Reduced forms (`09:30`, `2025-08`) and a fraction point with no digits after it are errors (`XFER0304`). Durations are not part of this version.

A temporal whose fields are well formed but out of range is an error (`XFER0305`). The ranges are: the year is any four digits, `0000` to `9999`; the month is `01` to `12`; the day is `01` to the length of the month, where February has 29 days in a year divisible by 4, except a century year not divisible by 400; the hour is `00` to `23`; the minute is `00` to `59`; the second is `00` to `60`, where `60` is a leap second as RFC 3339 permits; an offset's hour is `00` to `23` and its minute `00` to `59`. A fraction of a second may have any number of digits. So `2024-02-29` and `23:59:60` are valid, and `2023-02-29`, `1900-02-29`, `24:00:00` and `+24:00` are not.

The three forms are three types. A date is not a date-time with a missing time; the consumer decides what to do with each.

### 4.7 Booleans and null

`true` and `false` are the booleans. `null` is the null value. These are keywords (4.2).

## 5. Syntax

### 5.1 Document

A document is optional trivia, an optional version annotation (section 7), optional trivia, exactly one element, and optional trivia. A document with no element is an error (`XFER0401`). Anything other than trivia after the element is an error (`XFER0402`). A sequence of concatenated documents is not a document.

The root element may be any value, including a scalar. `42` and `"text"` are complete documents.

### 5.2 Elements and values

An element is zero or more annotations (section 7) followed by a value. A value is an object, a sequence or a scalar; a scalar is a string, a number, a temporal, a keyword value or a symbol.

### 5.3 Objects

An object is `{`, trivia, zero or more pairs separated by trivia, trivia, `}`. A pair is zero or more annotations, a key, trivia, and a value. A key is an identifier or a string; a keyword is not an identifier, so a key that spells a keyword is written as a string. Any other token in key position (a keyword, a number, a temporal, a bracket) is an error (`XFER0407`). The whitespace between a key and its value may be omitted only when the value begins with a self-delimiting token (separation rule): `{a[1]}` is well formed, and `{a"x"}` is not, because a string is not self-delimiting.

Pairs are ordered, and an implementation MUST preserve their order in the tree and in canonical output. Two pairs with the same key (compared as scalar-value sequences, whether spelled bare or as a string) are an error (`XFER0403`).

An object body with an odd number of scalar tokens is an error (`XFER0404`). Because a bare word may be a key or a symbol value, a missing value shifts every later pair; the diagnostic MUST say that a value is missing and MUST list the key/value pairings the parser inferred, so the author can see where the shift began.

### 5.4 Sequences

A sequence is `[`, trivia, zero or more elements separated by trivia, trivia, `]`. Elements may be of different types. An unterminated object or sequence is an error (`XFER0405`); an unexpected closing bracket is an error (`XFER0406`).

## 6. Canonical form

The canonical form of a document is a single spelling of its value, defined so that two implementations given the same document produce identical bytes. It is what the conformance suite compares. It is not a general-purpose formatter: comments are dropped (they are not part of the value), while annotations are kept (they are part of the Level 1 meaning). A lossless formatter that preserves comments and the author's layout is a tool, built on the lossless tree (section 9), and is outside this specification.

Rules:

1. Output is UTF-8 with LF line endings and no byte order mark, ending with one LF.
2. The version annotation is written first, as `<! xfer 1 !>` on its own line, if the source had one.
3. Indentation is four spaces per nesting level.
4. An object or sequence with no members is written `{}` or `[]`. Otherwise the opening bracket ends its line, each member is on its own line at the next indentation level, and the closing bracket is on its own line at the enclosing level.
5. A pair is the key, one space, the value. A key is written bare if it is a valid identifier that is not a keyword, otherwise as a string.
6. Each annotation is written on its own line immediately before the element or pair it attaches to, at that element's indentation, as `<! name value !>` with the value in canonical form.
7. Numbers are written as in the source except that the hexadecimal digits of a hexadecimal integer are lowercased. Digit separators and radix are preserved. Special floats are written `inf`, `-inf`, `nan`.
8. Temporals are written as in the source.
9. Keywords and symbols are written as spelled.
10. Strings use the choice rule: if the content contains no `"`, write `"content"`. Otherwise, if the content neither begins nor ends with `"`, write repetition with n one greater than the longest quote run in the content. Otherwise write hash-run with m one greater than the longest run of `#` that follows a `"` in the content. If "one greater than the longest" would exceed the run-length limit, use instead the smallest length that is not the length of any run in the content, which exists and is within the limit.

Canonical form is idempotent: the canonical form of a canonical document is itself.

## 7. Annotations (Level 1)

An annotation is `<!`, trivia, an identifier (the name), trivia, a value, trivia, `!>`. The value is a Level 0 value and MUST NOT itself carry annotations (`XFER0501`). An annotation attaches to the next pair when it appears inside an object and to the next element otherwise. Several annotations may precede one target and all attach to it. An annotation with no target (last in a collection, or last in a document) is an error (`XFER0502`).

The defined annotations are:

| Name | Value | Meaning |
|---|---|---|
| `xfer` | integer | The major version of this specification the document was written against. If present it MUST be the first token of the document (`XFER0503`). If absent the version is 1. A parser MUST reject a major version it does not implement (`XFER0504`). |
| `id` | string | An identifier for the target, unique within the document (`XFER0505` on a repeat). |
| `tag` | string or symbol | A classification of the target. Repeats are permitted. |
| `lang` | symbol or string | For a string target: the language its content is written in, for editors. No effect on parsing. |

Any other annotation name is an error (`XFER0506`). There is one parse mode. A tool that must carry unknown annotations through does so from the lossless tree (section 9), where every annotation is present whether or not it is defined here.

Annotations never evaluate, never read the environment or the file system, and never change the value of their target.

## 8. JSON mapping

### 8.1 XferLang to JSON

| XferLang | JSON | Diagnostic |
|---|---|---|
| object | object, same key order | |
| sequence | array | |
| string | string, with JSON escaping applied | |
| integer | number, full precision | `XFER0901` (warning) above 2^53 or below −2^53; an implementation MUST offer an option to emit such integers as strings instead |
| fractional | number, as written with `_` removed | |
| `inf`, `-inf`, `nan` | no mapping | `XFER0902` (error) |
| `true`, `false`, `null` | `true`, `false`, `null` | |
| symbol | string | `XFER0903` (informational) |
| temporal | string, as written | `XFER0904` (informational) |
| comments, annotations | dropped | |

### 8.2 JSON to XferLang

| JSON | XferLang |
|---|---|
| object | object; on duplicate keys the last value wins with `XFER0905` (warning) |
| array | sequence |
| string | string, with JSON escapes decoded; never a symbol |
| number with no fraction or exponent | integer |
| number with a fraction or exponent | fractional, spelled as in the source |
| `true`, `false`, `null` | keywords |

The mapping from JSON is lossless apart from duplicate keys. The mapping to JSON is defined and diagnosed, not lossless; a document that round-trips through JSON may come back with symbols and temporals as strings.

## 9. Parsing results and the lossless tree

A conformant parser MUST NOT fail by exception, signal or abort on any input. It returns a result containing zero or more diagnostics and, where any part of the input was understood, a tree.

A **diagnostic** has a code from this specification, a severity (error, warning, informational), a position (UTF-8 byte offset and length, and line and column counted in scalar values from 1), and a message. The suite compares codes and positions, not messages.

Positions are measured in the normalized text (section 3.2): after the leading byte order mark is removed and line endings are replaced by LF. So the first character after a byte order mark is at offset 0, column 1, and a lone CR in the input ends a line. A diagnostic about the input as a whole, such as invalid UTF-8 or the document size, is reported at offset 0, line 1, column 1.

A parser SHOULD recover at the next element boundary it can isolate and continue, so that a document with several faults reports each one the parser can separate. Some faults have no boundary to recover at (an unterminated string consumes the rest of the document) and the spec does not require recovery from them. A tree returned with errors is partial and MUST mark the regions it could not parse.

Implementations SHOULD provide a **lossless tree**, in which every byte of the source is reachable: tokens, trivia, delimiter run lengths, and the spelling of every number. A **value model** derived from it drops trivia and spellings and is what most consumers use. The canonical form (section 6) is a function of the value model plus annotations.

## 10. Diagnostic codes

Codes are stable. A code is never reused for a different meaning; a retired code is listed as retired.

| Code | Severity | Meaning |
|---|---|---|
| XFER0101 | error | scalar token not followed by a separator |
| XFER0102 | error | `<` not followed by `/` or `!` |
| XFER0103 | error | character cannot begin a token |
| XFER0104 | error | unterminated comment |
| XFER0105 | error | identifier ends with `.` |
| XFER0201 | error | unterminated string |
| XFER0202 | error | `#` not followed by `#` or `"` |
| XFER0301 | error | leading zero on a decimal integer |
| XFER0302 | error | malformed number |
| XFER0303 | error | number out of range for the requested type (conversion, not parsing) |
| XFER0304 | error | malformed temporal |
| XFER0305 | error | temporal denotes no valid instant |
| XFER0401 | error | document has no element |
| XFER0402 | error | content after the root element |
| XFER0403 | error | duplicate key |
| XFER0404 | error | object body has a key without a value |
| XFER0405 | error | unterminated object or sequence |
| XFER0406 | error | unexpected closing bracket |
| XFER0407 | error | token in key position is not an identifier or string |
| XFER0501 | error | annotation value carries an annotation |
| XFER0502 | error | annotation has no target |
| XFER0503 | error | `xfer` annotation is not the first token |
| XFER0504 | error | unsupported major version |
| XFER0505 | error | duplicate `id` |
| XFER0506 | error | unknown annotation |
| XFER0601 | error | document exceeds size limit |
| XFER0602 | error | nesting exceeds depth limit |
| XFER0603 | error | scalar token exceeds length limit |
| XFER0604 | error | delimiter run exceeds length limit |
| XFER0605 | error | element count exceeds limit |
| XFER0701 | error | invalid UTF-8 |
| XFER0901 | warning | integer outside JSON's safe range |
| XFER0902 | error | special float has no JSON mapping |
| XFER0903 | informational | symbol mapped to JSON string |
| XFER0904 | informational | temporal mapped to JSON string |
| XFER0905 | warning | duplicate key in JSON source; last value kept |

## 11. Conformance

An implementation is conformant with this version when it:

1. accepts every document in `tests/valid.txtar`, produces its canonical form byte for byte, and produces its value model as the case states;
2. rejects every document in `tests/invalid.txtar` with the code the case states at the position it states;
3. accepts every document within the limits in 3.4;
4. never fails by exception, signal or abort on any input;
5. passes the property tests described in `tests/README.md` for the embedding and whitespace invariants.

## Appendix A. Differences from XferLang 0.x

This specification is a clean-sheet redesign and is not compatible with documents written for the 0.x library. The differences, for anyone migrating:

- All type sigils (`#`, `&`, `*`, `^`, `~`, `?`, `@`, `=`, `:`, `_`, `|`) are gone. Types are stated by literal form.
- The explicit `<...>` wrapper around values is gone; `<` introduces only comments and annotations.
- Tuples are gone and sequences are heterogeneous.
- Bare words in value position are symbols rather than errors.
- Keys may be any string.
- The recursive key/value form (`a b "c"`) is gone.
- Processing instructions that compute (`let`, `if`, `script`, `dynamicSource`, interpolation) are gone from the format. `document` is replaced by `xfer`, `id` and `tag` remain, `lang` is new.
- Line endings are normalized; strings are verbatim otherwise.
- Hash-run strings are new.
- Parsing never throws and never reads the environment or the file system.

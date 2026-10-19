# What the first implementation found

Written 2026-10-19, after the JavaScript implementation (`xfer-js`, published as `xferlang@0.1.0-next.1`) passed the whole conformance suite. It is addressed to whoever writes the next implementation, starting with `xfer-dotnet`. Section 1 lists the defects the first implementation found and the commits that fixed them, so the specification you read is already corrected. Section 2 lists the places where the specification is still silent or ambiguous, with a proposed resolution for each, so that you do not have to rediscover them. Section 3 lists practical lessons that are not specification matters. Section 4 lists gaps in the suite.

Nothing in section 2 has been decided. Each item needs either a specification edit or a ruling from Paul, and the proposals say which.

## 1. Defects found and already fixed

| Defect | Fixed in |
|---|---|
| Two byte-level cases had a raw newline inside a string in their `value.json`. | cb5c2d0 |
| Section 4.5 listed `.5` and `+1` as `XFER0302` while section 4 and the suite say `XFER0103`. | 55cdffc |
| The ranges of a valid temporal were missing from 4.6, and second 60 (a leap second) was unmentioned. | 55cdffc |
| The rules for `-inf`, signed temporals, uppercase radix prefixes and a lowercase `z` were unstated. | 55cdffc |
| Section 9 did not say that positions are measured in the normalized text. | 55cdffc |
| The case `530-object-minimal-whitespace` contained `c"x"`, which the separation rule forbids. | 55cdffc |
| Four invalid cases were named for the wrong diagnostic code. | 55cdffc |
| A run of exactly two quotes was both the empty string and the opening of a string whose content begins with a quote. Ruled: `""` is always the empty string. | e5451c0 |
| The minimum document and token size of 2^31-1 bytes cannot be met by a JavaScript engine or by .NET. Ruled: 2^28-1. | e5451c0 |
| Section 5.1 and the grammar allowed a comment before the `xfer` head, while section 7 and the suite forbid it. | cd3a038 |
| Section 2 cited the wrong subsection for the keywords. | cd3a038 |
| The grammar did not say when the space between an annotation's name and its value may be omitted. | cd3a038 |
| The case `440-string-hash-run` had the wrong canonical form for a string with a quote in the middle. | 41067b0 |
| The last sentence of canonical rule 10 claimed the fallback delimiter length is always within the run limit, which is false for content with hash runs after quotes of every length up to 64. | 32ce17a |
| Section 3.4 said a parser may raise the limits and never lower them. Ruled: the minimums bind the default configuration, and a deployment may choose stricter limits. | 2415d35 |
| Section 7 gave "last in a document" as an example of an annotation with no target, while the suite requires `XFER0402` for an annotation after the root. Ruled: it is content after the root. | 2415d35 |
| JSON strings holding U+000D or a lone surrogate cannot be represented, so section 8.2 now reports them (`XFER0907`) and malformed JSON has a code (`XFER0906`). | 2415d35 |
| Section 8.1 did not say how an integer is written, what the compact JSON form is, or where a diagnostic from the JSON direction points. | 2415d35 |

## 2. Still open

**2.1 Recovery after a malformed annotation.** Section 7 says nothing about where parsing resumes after a malformed annotation, and the reference parser does not skip to the next `!>`, because that swallows a following closing bracket when the `!>` is merely missing (`{ <! tag x }`). The consequence is that one malformed annotation can produce a second error from the tokens left over. Proposal: say in section 9 that recovery is a matter of quality and that the suite pins only the first diagnostic of a document with several faults. That is already how the suite's runner treats extra diagnostics. This needs a specification sentence and no ruling.

**2.2 One code for every malformed annotation.** The reference parser reports a malformed annotation, a wrong-typed annotation value and the legacy form `<! let x 1 !>` all as `XFER0506`, with the difference in the message. Proposal: confirm that `XFER0506` covers all three and add no new codes, because the suite compares codes, not messages. This needs one sentence in section 7.

**2.3 An annotation between a key and its value.** The grammar has no such position, so the reference parser reports it as `XFER0502` and keeps it in the tree so the tree stays lossless. The specification lists this case under `XFER0502` already, but it is worth a suite case in `invalid.txtar` if one is not there.

**2.4 Which position a repeated key keeps.** For a repeated key (already `XFER0403`), a tool that reads the tree has to choose between the first and the last position and the first and the last value. The reference implementation keeps the first key's position and the last value. Proposal: say that the value model keeps the last value and leave the position unspecified.

**2.5 Depth of a JSON document.** Section 8.2 is silent on a JSON document nested deeper than 256 levels. Converting it succeeds, but the result cannot be written as a document, because a document deeper than the limit is `XFER0602`. The reference implementation applies the depth limit to the JSON reader, reports `XFER0602`, and gives no value. Proposal: add that sentence to section 8.2. This is the reference implementation's own choice, not a ruling.

**2.6 Hand-built values.** The specification defines documents, not the in-memory value model, so it does not say what a program may put in one. The reference implementation refuses to write a value whose string holds U+000D or a lone surrogate, whose symbol is not an identifier, whose number lexeme is not exactly one number token, or whose annotation is anything but `id`, `tag` or `lang` with a scalar value. Proposal: add a short paragraph to section 9 stating the constraints on a value model, so that every implementation refuses the same inputs.

**2.7 Canonical output size.** The 4-space indent makes canonical output grow with the square of the nesting depth, so a 24 KB document nested 12,000 deep produces more output than a string can hold. The reference writer enforces the depth limit and a size limit and reports an error instead. Proposal: say in section 6 that a canonical writer MAY refuse a value nested deeper than the depth limit.

**2.8 Memory.** The reference parser's lossless tree costs about 100 bytes of heap per byte of input, so the specification's minimum of 256 MiB would need 25 GB. A runtime that dies on exhausting memory (V8 does, uncatchably) can be crashed by a hostile document that is within the limits. The parser is linear in time. Proposal: add a note to section 3.4 that the minimums are a floor for the default configuration and that a parser of untrusted input should choose lower limits. The limits ruling in section 1 already covers most of this, so only the note is needed. A more compact tree is an implementation task, not a specification one.

**2.9 Where an integer becomes a warning.** Section 8.1 says `XFER0901` applies "above 2^53 or below -2^53", so 2^53 itself is not flagged, although it is not a safe integer in the usual sense, since 2^53 and 2^53+1 are the same double. The suite pins the boundary (`811-integers-past-the-safe-range`). Proposal: keep it, and say so explicitly in the table row.

## 3. Practical lessons

- **Measure positions in the normalized text.** After the byte order mark is removed and CR and CRLF become LF, offsets are in UTF-8 bytes and columns are in scalar values. A runtime whose strings are UTF-16 must convert; the reference implementation tracks both as it lexes.
- **The suite's `.canonical.xfer` files also end in `.xfer`.** A runner that splits entry names by suffix must try the longest suffix first and stop at the first match. The reference runner got this wrong once.
- **Use the expected-failures ratchet.** Start with every case listed as an expected failure and make the runner fail when a listed case passes. The reference implementation went from 123 listed cases to none over four steps, and the list showed exactly how far along it was.
- **Compare against the specification text where you can.** The reference implementation has a test that parses the exclusion sentence in section 4.3 and the diagnostic table in section 10 and compares them with its own tables. A change to the specification then fails the build until the code follows.
- **Make every walk over the input iterative.** The depth limit is a diagnostic, not a guard against stack overflow, so the lexer, parser, canonical writer and JSON converters all use explicit stacks. Each was tested at 200,000 levels.
- **Wrong-closer handling can be quadratic.** The first parser scanned the whole stack for a matching bracket on every stray closer, which took 5.7 seconds on a 5 MB document. Keep counts of open objects and sequences instead, and save and restore them around an annotation.
- **Fractionals keep only their lexeme.** The reference implementation does not decide between binary floating point and decimal. It offers helpers that report whether a conversion to a double is exact, by comparing the exact binary expansion of the double with the decimal value in arbitrary-precision arithmetic. A .NET implementation should offer the same for `double` and `decimal`.
- **Temporal libraries disagree with section 4.6.** The JavaScript `Temporal` clamps second 60 to 59 and has no offset-carrying instant, so the reference helper refuses second 60 and documents that an instant loses its offset. Expect similar trouble with `DateTimeOffset` and `TimeOnly`.
- **Generated tests found what the suite could not.** A seeded generator that builds random documents from a model, renders each twice (random spacing and minimal spacing) and requires the parser to reproduce the model found real bugs. The first version of the generator was too weak to find them and a review found that out, so the generator now has a coverage guard that fails if a feature drops below a stated share of the documents.
- **Run a cold adversarial review after each stage.** Each stage of the reference implementation was reviewed by an agent with no prior context. The reviews found a lossless-order bug, a quadratic path, four error cascades, an unsafe API (writing from a bare tree), a missing JSON depth limit, and a wrong position for a duplicate-key diagnostic. None of them failed the suite.

## 4. Gaps in the suite

- No case has CR, LF or tab between tokens in JSON text, which must not be an error. It cannot live in an archive, because a text editor may change the line endings, so it belongs in `tests/bytes/`.
- `to-json.txtar` pins only tab and LF among the control characters. A case for `\b`, `\f`, `\u001f` (lowercase hexadecimal) and for U+007F and U+2028 written literally would pin the compact form completely.
- Nothing asserts that a diagnostic is absent. The runners check that every listed code is present and permit others, so a spurious `XFER0901` at exactly 2^53 or a spurious `XFER0905` on a clean document would pass. The reference implementation covers this with its own unit tests; a suite format that lists forbidden codes would cover it for everyone.
- There is no case for a JSON document nested past the depth limit, pending the decision in 2.5.
- The reference implementation's JSON round-trip and differential tests, which compare its reader against the platform's `JSON.parse` on tens of thousands of generated inputs, are language-specific and are not part of the suite.

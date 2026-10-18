# Adversarial review of REBOOT_PROPOSAL.md, round 1

Reviewed 2026-10-18 against the proposal as it stood after the line-comment decision. The reviewer's brief was to read cold and try to break it. Findings are numbered, graded, and each says what would make it pass. **Blocking** means the spec cannot be written without a ruling. **Required** means the proposal is wrong or incomplete and must change before implementation, but the fix is clear. **Advisory** means worth knowing.

## Verdict

Not yet viable. The implementation strategy (sections 1, 2, 5, 6) passes with the amendments in findings 20 to 25. The language design (section 3) has nine blocking gaps, all in the lexical layer, which is where the current parser's faults also live. None of them is fatal to the design; all of them are the kind of thing that, left to the implementer, produces two parsers that disagree.

## Blocking findings

**1. Adjacent tokens are undefined.** The proposal never says what separates two scalars. `1-2` could be the integer `1` followed by `-2`, or an error. `"a""b"` under the stated closing rule (first run of at least n quotes, surplus to content) lexes as the string `a"` followed by garbage, which is wrong in a way the author would never guess. The current parser's "closer is whatever character the code happens to stop at" fault recurs unless this is pinned. **Fix:** a scalar token must be followed by whitespace, a closing bracket, a comment or annotation opener, or end of input. Only the brackets `{ } [ ]` and the `<` of a comment or annotation are self-delimiting. This also tightens the whitespace invariant (finding 2).

**2. The whitespace invariant is vague where it matters.** "Removed entirely where the adjacent tokens remain distinguishable" is a judgment, and judgments diverge between implementations. **Fix:** restate it in terms of finding 1: whitespace between a scalar and anything other than a self-delimiting token is mandatory and may be any non-empty run; whitespace adjacent to a self-delimiting token is optional. Also define the whitespace set. **Recommend:** space, tab, LF, CR and FF only; no Unicode spaces (they are invisible and a safety hazard in a language where whitespace is the separator).

**3. CRLF breaks the embedding invariant in practice.** Multi-line strings are verbatim, so a string's value changes when Git's `autocrlf` or a Windows editor rewrites line endings, and a file that round-trips on Linux fails on Windows. The invariant as written ("bytes unchanged") cannot survive contact with version control. **Options:** (a) normalize CRLF and lone CR to LF across the whole document before lexing, as KDL 2.0 does, and state the invariant over LF-normalized text, accepting that a lone CR is unrepresentable and must be transported encoded; (b) keep bytes verbatim and require `.gitattributes` from every user. **Recommend (a).** The same editor behavior strips trailing whitespace inside multi-line strings; the spec cannot fix that and should warn about it.

**4. Symbols move errors away from their cause.** Objects are strict key-value alternation. Today a bare word can only be a key, so a missing value is caught at the next token. Under the proposal a bare word is also a value, so `{ a b c d e f g }` with one value missing shifts every pair and reports the error at `g`, far from the fault. **Fix:** keep symbols (the benefit is real) but require the diagnostic for an odd-length object body to say so explicitly and to list the pairings it inferred, and note in the spec that a schema is the real defense. Record the trade in the decisions.

**5. `inf` and `nan` are symbols.** A generator serializing a floating-point infinity or NaN has no representation, and a consumer mapping the symbol `nan` to a double gets a type error. TOML reserves `inf`, `+inf`, `-inf`, `nan`. **Decision needed:** reserve them as fractional literals, or declare them unrepresentable (JSON's position). Either is defensible; the proposal must pick.

**6. Numeric and temporal lexemes are not pinned.** Is `09:30` a time? Is `2025-08` a date? Is `1.` or `.5` a number? Is `+1` legal? Is `0755` an error (leading zero) or decimal 755? Where may `_` appear in a number? Is `1e5` an integer or a fractional? Is `0X1F` legal? **Fix:** the spec adopts RFC 3339's full-date, partial-time and date-time productions and nothing else for temporals; integers are `0` or `[1-9][0-9]*` with single `_` between digits, optional leading `-`, no `+`; `0x`, `0o`, `0b` lowercase prefixes with any-case digits; fractional requires digits on both sides of the point or an exponent, and any exponent makes the literal fractional. Canonical form lowercases prefixes and preserves the rest as written.

**7. Identifiers are not pinned.** "Letters" is undefined (ASCII? Unicode? which normalization?). **Recommend:** ASCII letters, digits, `_`, `-`, `.` in 1.0, starting with an ASCII letter or `_`; anything else is a quoted key. This keeps the C lexer trivial and avoids the NFC question entirely. State explicitly that `.` has no path meaning (`a.b` is one flat key), since TOML's dotted keys make readers expect otherwise.

**8. Content that is only quotes cannot be written with repetition.** `""` + two quotes of content + `""` is six quotes, which the maximal-run rule reads as a run of six with no closer. Hash-run covers it (`#""""#` is the two-character string `""`), so the invariant holds, but 3.3 lists only "begins or ends with a quote" as the exception. **Fix:** state the full exception (begins with, ends with, or consists only of quote characters) and define the canonical writer's choice rule: repetition with minimal run unless the content falls in the exception, then hash-run with minimal hashes. Without that rule two implementations produce different canonical bytes.

**9. Annotations are underspecified.** What does `<! id "x" !>` inside an object attach to: the key, the value, or the pair? May an annotation be the last thing in a collection? May an annotation's value itself carry annotations? Is `<! xfer 1 !>` optional, and if so what does absence mean? The proposal also says unknown annotations are "a diagnostic in strict mode", and strict mode is mentioned nowhere else. Two modes are two dialects. **Fix:** attach to the pair in objects and to the element elsewhere; a trailing annotation is an error; annotation values are plain Level 0 elements with no nested annotations; `xfer` is optional and absence means version 1; and there is exactly one mode, in which an unknown annotation is an error. Tooling that wants to carry unknown annotations uses the lossless tree, which keeps them regardless.

## Required findings

**10. The Level 2 example is wrong by the proposal's own rules.** 3.13 says a Level 0 parser given `{ database { host _host port 5432 } }` "reports a symbol where an object was expected". It does not. `_host` is a legal symbol under 3.5, so the document is valid Level 0 and `host` is the symbol `_host`. That is worse than an error: a template fed to a Level 0 consumer parses cleanly with the wrong values. **Fix:** Level 2 documents carry a mandatory head annotation that Level 0 rejects in all modes (for example `<! xfer-template 1 !>`), use a different extension, and the example's last paragraph is rewritten to say so.

**11. "The only data format with zero escape sequences" is false.** NestedText has no escape sequences at all. **Fix:** either "the only typed data format" or drop "only" and describe the property. The sentence already says to verify before claiming; the verification has now failed once.

**12. "Lossless JSON off-ramp" overstates.** JSON to XferLang is lossless. XferLang to JSON loses symbols, temporals, big integers and all comments and annotations, and a round trip through JSON does not return the original. 4a's wording implies both directions. **Fix:** "lossless from JSON, defined and diagnosed to JSON."

**13. Big integers to JSON strings is a semantic change.** RFC 8259 allows arbitrary-precision numbers; only JavaScript loses them. Emitting `"9007199254740993"` turns an ID into a string and breaks any consumer that compares numbers. **Fix:** emit as a JSON number by default with a warning, with a `--stringify-big-ints` option for JavaScript consumers.

**14. Limits as options break conformance.** If depth is a parser option, two conformant parsers can disagree on whether a document is valid. **Fix:** the spec defines minimum limits every conformant parser must accept (for example depth 256, run length 64, document size 2^31−1) and the defaults; an implementation may raise them, never lower them.

**15. Diagnostic positions need a unit.** The C core thinks in UTF-8 bytes, .NET in UTF-16 units, the Language Server Protocol in UTF-16 by default. The suite compares positions, so they must agree. **Fix:** diagnostics carry byte offset (UTF-8) and line plus column in Unicode scalar values; the suite compares code, line and column.

**16. The suite needs a value oracle, not only canonical bytes.** Canonical form preserves number spellings, so it is a formatting oracle. **Fix:** each valid document also has an expected value model in the tagged-JSON form toml-test uses (`{"type":"integer","value":"42"}`), and the suite checks both.

**17. Recovery is overstated.** An unterminated string has no element boundary to recover at; the example's own document swallows `c 2 }` into the string. "A document with several faults reports all of them" should read "reports every fault the parser can isolate", and the result should be defined as a possibly partial tree with error nodes plus diagnostics, which is what `NavService` needs to keep good entries.

**18. Duplicate keys and from-json.** JSON permits duplicate keys; the proposal makes them an error; from-json is therefore not lossless on such input. **Fix:** from-json reports a diagnostic and keeps the last value, matching what every JSON parser does, and says so.

**19. The `schema` annotation points at nothing.** 1.0 would ship an annotation whose target language does not exist. **Fix:** defer `schema` to the schema specification or define it as an opaque string reference with no 1.0 semantics.

## Strategy findings

**20. Order of work is wrong for one person.** Section 1 and 5.3 put the C core third and the managed implementation fourth. The only consumer is .NET; a suite written with no parser to run it against is unvalidated; and the first implementation of any spec finds the spec's faults. **Fix:** spec and suite and the managed implementation are developed together until the suite is green and the site has migrated, then the spec freezes, then the C core is written against a frozen suite with differential testing against the managed one. The C core stays the reference once it exists.

**21. "Without a translation layer" is false.** Java needs JNI or Panama, Go needs cgo, Node needs N-API. **Fix:** "every language has a well-trodden path to a C header."

**22. The trusted-parsers list undercuts the safety argument.** libyaml, expat and cJSON all have CVE histories. **Fix:** cite them as what gets embedded, and cite SQLite alone for reliability.

**23. The Rust fallback trigger is undefined.** **Fix:** evaluate at 1.0 freeze and at each minor release; any memory-safety finding after 1.0 triggers a written review; a second within twelve months triggers the rewrite.

**24. Fuzzing "on every push" is not how fuzzing works.** **Fix:** a time-boxed fuzz run per pull request, a continuous scheduled job, and OSS-Fuzz enrolment once the project is public.

**25. "Roslyn-free API" means nothing.** Delete it. If the intent was "no source generators", say that.

## Advisory

**26. The agent-authorship bet is weaker than stated.** Models have no training data on this syntax, so first-contact error rates will be higher, not lower; the claim is about in-context learning from a grammar in the prompt. Structured-output decoding already makes JSON generation error-free through APIs, which removes the API case; what remains is coding agents editing files. The measurement must include a grammar-in-prompt condition and should say which case it is measuring.

**27. The honest comparison with KDL belongs in the proposal.** The design has converged on something KDL-adjacent: KDL 2.0 has composing raw strings, no commas, no indentation significance, comments, and a spec with a suite and many implementations. XferLang's remaining differences are: zero escapes anywhere rather than as a second mode, verbatim rather than dedented strings, nesting comments, and a JSON-shaped data model. Those are real and they are modest. The proposal should say plainly that the other reason to build rather than adopt is ownership across the portfolio, so the decision is made with that on the table.

**28. Verbatim multi-line strings will annoy authors.** The newline after the opener and every continuation line's indentation are content, so an indented excerpt either carries its indentation or is written flush left. The proposal accepts this; it should show the idiom (open the string on the same line as the key, continue flush left) so authors are not surprised.

**29. Control characters and NUL.** The invariant admits any scalar value, so raw C0 controls and NUL are legal string content. JSON forbids them unescaped. Say so, and say that invalid UTF-8 is an error.

**30. Jsonnet and Dhall as adoption evidence.** Dhall's adoption is small. Cite Jsonnet alone.

## What passing looks like

Round 2 passes when: findings 1 to 9 each have a ruling recorded in the proposal's section 7; findings 10 to 25 are applied; the 3.13 examples are re-checked against the pinned lexical rules; and section 5.3 reflects the re-ordered work. At that point a spec writer can start without having to invent anything.

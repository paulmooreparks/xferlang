# XferLang: a fresh assessment before the reboot

Written 2026-10-18 from a clean read of the repository at commit 3f3d599 (last change 2025-08-27), the live `sitenav.xfer` in the parkscomputing.com engine, the NuGet publication history, and a probe program run against the current parser. The question asked was whether XferLang is a viable alternative to JSON and YAML, what its real strengths and weaknesses are, and where a reboot should take it.

## 1. What XferLang is today

The repository contains one working implementation, the .NET library `ParksComputing.Xfer.Lang` (about 14,700 lines, of which the parser is 2,543), a 595-test MSTest suite that passes, a VS Code extension with a 152-line TextMate grammar and no language server, a Rust port of 190 lines that stopped in June 2025, a schema layer whose parser and validator bodies are both wrapped in `#if false`, a JSON-to-Xfer tool with no reverse direction, and about a dozen design whitepapers that pull in several directions at once. Twenty-two published prerelease versions exist on NuGet; the GitHub repository has 17 stars and no open issues.

There is exactly one real consumer. The parkscomputing.com engine deserializes `sitenav.xfer` into `NavNode` objects with `XferConvert.Deserialize`, and the admin site reads `etc/admin-menu.xfer` the same way. The live file (420 lines, under `OneDrive\Documents\parkscomputing.com\wwwroot`, newer than the copy in the engine repository) uses objects, arrays, strings, `<"...">` strings, booleans, comments, the `document` processing instruction, and one interpolated string. It uses no numbers, no dates, no references, no `let`, `if`, `script` or dynamic values. Every other feature of the language appears only in demo files written to show that feature.

That consumer is the most useful data point in the repository. It shows which part of the language a real author reaches for when nobody is watching, and that part is small. Section 4 reads it closely.

## 2. Strengths

The core idea holds up. Writing the type into the delimiter, separating elements with whitespace, and dropping commas and escape sequences produces documents that read better than JSON and are far less fragile than YAML. `sitenav.xfer` is pleasant to read and to edit, and it would be worse in either of those formats.

The run-length delimiter rule (write `""..""` when the text contains a quote) is proven elsewhere. Rust raw strings (`r#"..."#`), Lua long brackets, C++ raw string delimiters and Markdown fences all use the same device, and users understand it quickly. It is XferLang's single best feature, because it removes escaping from the language entirely, and embedded HTML, Markdown, regular expressions and code snippets are exactly what YAML and JSON handle worst.

Typed scalars solve problems JSON actually has. JSON cannot distinguish an integer from a float, loses int64 precision through JavaScript, has no decimal type and no date type. XferLang has all four, in the syntax, with no heuristics.

Comments exist. This alone rules JSON out of the human-edited configuration niche, and XferLang's comment syntax nests safely by the same run-length rule.

The object model, the attribute-driven object mapper and the converter and contract-resolver extension points are a reasonable .NET surface. They follow Newtonsoft conventions and the site's `NavNode` class shows they work for ordinary cases.

## 3. Weaknesses

### 3.1 The language does not know what it is

The documents disagree with each other and with the code on fundamentals. The reference element is written with `_` in the README and in `ReferenceElement.cs`, with `&` in `xfer.bnf`, with `;` in `PROJECT_CONTEXT.md`, and with a backtick a few lines later in `RECENT_DECISIONS.md`. The query element is `?` in the BNF, a backtick in the agent notes, and `;` in `QueryElement.cs`. The BNF gives keywords the `:` specifier; the code uses `=`. The BNF says an implicit keyword matches `[A-Za-z_][A-Za-z0-9_]*`, and the README says a key such as `menu-id` must be written `=menu-id=`, but `CharExtensions.IsKeywordChar` accepts `-` and `.`, and the live site file writes `menu-id` bare on every menu. The one production document depends on behavior that the specification forbids.

The conditional and scripting designs show the same drift. Expression syntax is `eq[a b]` in one paper, `(eq a b)` in another, `eq(a, b)` in a third, and `if defined |x|` in the README. Comparison operators are `gte` in two papers and `ge` in two others. One paper says `endif` is unnecessary and then uses it in its own examples. `MERGE.md` is empty.

None of this is unusual for an experiment. It is disqualifying for a format, because a format is a contract, and there is no document here that could serve as one.

### 3.2 The grammar is an accident of parse order

The parser has no lexer. It dispatches on the leading character by trying about twenty openers in a fixed order, and the compact forms end wherever the code happens to stop, which is defined as "whitespace or any character from a partial list of specifiers", not by a grammar rule. That makes the BNF a description after the fact, and it is wrong in the places listed above.

Comments are stripped inside `Advance()`, below the level of syntax, so they are removed from inside string literals too. The probe confirmed that `{ a "</ not a comment />" }` is a parse error.

The implicit forms have sharp edges that the probe also confirmed:

- `{ a ?, b 1 }` hangs the parser. The null element's loop does not advance when the next character is not in its expected set.
- `{ a "" }`, an empty string, is a parse error, because `""` is read as an opening run of length two.
- `_my_var` cannot be parsed as a reference, because `_` is both the opening and closing specifier, so any `let` binding with an underscore in its name is unreachable in compact form.
- `{ x hello }` is an error. A bare word can never be a value; it must be written `"hello"` or `:hello:`.
- `1.5` is an error (no implicit decimal), `3000000000` is an error (no promotion to long), and `-$FF` is an error (sign and radix prefix do not combine).
- `[ 1 "two" ]` is an error because arrays must be homogeneous, and `[ 1 2.5 ]` is an error for the same reason, which makes converting ordinary JSON awkward.

The 595 passing tests did not catch any of these. The API audit in the repository records why. Tests were loosened to match whatever the implementation produced, and the expected serialized text was never fixed as a specification.

### 3.3 Parsing evaluates, and does so by default

The parser's constructor registers the `if`, `script`, `let` and `dynamicSource` handlers, and every handler runs during parsing. A `|NAME|` element calls the resolver at parse time, and the default resolver falls back to `Environment.GetEnvironmentVariable` for any name and supports reading files from disk. The probe confirmed that `{ a |PATH| }` parses to a document whose serialized form contains the machine's PATH. The `DynamicElement` stores the resolved value rather than the variable name, so round-tripping a document writes secrets into the output. The `dynamicSource` PI writes to a static global registry shared across documents and threads.

For a data format this is the most serious problem in the repository, for three reasons. Parsing untrusted input must not read the environment or the file system. A document's meaning must not depend on the machine that parses it. And a second implementation in another language cannot be conformant unless it reproduces the .NET resolver, the scripting engine and the operator registry, which no one will do.

The Turing-completeness, object-binding and LINQ papers would extend this further, into reflection calls on .NET methods, loops, functions and HTTP. That is the Helm and Ansible problem, where YAML stopped being data, and it is the thing the README's own pitch ("no heuristics or guesswork") promises to avoid.

### 3.4 The type sigils are .NET's types, not a format's types

Four numeric sigils (`#` int32, `&` int64, `*` decimal, `^` double) expose CLR storage classes in the syntax. A format meant for other languages should say what the number is (an integer, an exact decimal, a binary float) and let each implementation choose storage. The sigils are also not mnemonic; nothing about `*` suggests decimal or `^` suggests double, and `&` is used for both long and, in the BNF, reference. In the 22 sample files, `&` and `^` appear zero times.

### 3.5 Diagnostics and limits

Errors are thrown as `InvalidOperationException` on the first failure, with no recovery and no structured result. `ParseError` exists and is never set. Row and column tracking is not restored on rewind, so positions drift. Array errors are re-wrapped at every nesting level and report the array's start rather than the fault. There is no depth limit (deep nesting overflows the stack), no size limit and no cap on delimiter run length. These are the properties a parser needs before it can accept input from anyone but its author.

### 3.6 Ecosystem

One implementation, one platform, one consumer, no language server, no Xfer-to-JSON, no conformance suite, no schema. Every competing format in this space has at least a spec and a test suite, and most have implementations in several languages.

## 4. What the live file teaches

The live `sitenav.xfer` was written by the language's author for his own site, so every workaround in it is a place where the language made the obvious thing hard.

**The file reaches for two different quote workarounds.** Excerpts that contain an apostrophe or a quote are written `<"...">`, which is the documented form. One title that begins and ends with a quote character is written `title<'"That sounds so much more exotic than here!"'>`, an explicit interpolated string with no embedded elements and no space before the value. It works by accident, and it shows that when the author hit a quote problem in the moment, two mechanisms presented themselves and neither was obviously the one to use. The spec should have one answer.

**Excerpts use `<"...">` even when they contain no quote.** Most of them do not need it. The author stopped trusting the compact form and defaulted to the heavy one, which is a readability cost the language is supposed to avoid.

**Structured values are written as strings.** Window shapes are `win "0.9,0.9"` and `min "420,220"`, parsed by the engine. The native form would be `win ( *0.9 *0.9 )`, and the author chose the string. The sigil tax on numbers pushed structured data out of the type system. Under a design with implicit decimals, `win ( 0.9 0.9 )` would have been written.

**Bare keys use hyphens.** `menu-id` and `window-commands` appear throughout both files. The README says these keys need the `=menu-id=` form; the parser accepts them bare; the site depends on the parser. Whichever way the spec rules, the live file has voted.

**Comments disable whole entries.** Two articles are switched off by wrapping their entire `{ ... }` block in `</ ... />`, nested braces and all, and the file opens with a twelve-line comment documenting the menu grammar. This is the language working as intended, and it is the feature JSON most conspicuously lacks. It also exposes a hard requirement: the library's round trip drops every comment. Parsing the live file and serializing it back produces a document that re-parses identically but has lost the usage note and both disabled entries. The site roadmap plans a structured navigation editor with the raw XferLang kept as an advanced view, which cannot be built on a tree that forgets comments and formatting. A lossless syntax tree is a requirement of the one consumer, not a refinement.

**One bad character takes down the whole menu.** `NavService.ParseRoot` catches any exception from `Deserialize`, logs it, and returns an empty root, so the site renders with no navigation until the file is fixed. With structured diagnostics and recovery, the parser could return the entries it understood and report the one it did not.

**The consumer is being eroded from both sides.** The books design (2026-10-01) says the Markdown front-matter work is "the trial for replacing `sitenav.xfer` with `index.md` files: if a folder of Markdown with front matter can describe a book well, it can describe the whole site." The applet-packages design moves window sizes out of `sitenav.xfer` and into a per-applet `applet.json`. Given two new sub-problems on his own site, the author reached for YAML front matter and JSON rather than for his own format. That is the most honest market signal in this investigation, and it is worth asking why before the reboot. The likely answer is locality (metadata next to the content it describes) and tooling (every editor understands front matter and JSON), and neither is a fault of the syntax; the first is a question of where data lives, and the second is the ecosystem gap in section 3.6.

**The author's own stated reasons for choosing XferLang over JSON** are in the draft article `xferlang-site-navigation.md`: no trailing commas, no escaping in long excerpts, comments for notes and temporary removals, and alias keys without collisions. Every one of these is a Level 0 feature as section 8 defines it. None of them involves typing, interpolation, references or processing instructions.

## 5. The competitive field

The "human-written typed configuration" niche is not empty, and the reboot has to be designed knowing who is in it.

- **TOML** owns small flat configuration. It has a frozen 1.0 spec, a language-neutral conformance suite (toml-test), and implementations everywhere. It is poor at nesting and at multi-line text.
- **KDL** is the closest relative. It is node-based, whitespace separated, comma free, has raw strings and type annotations, reached a 2.0 spec in 2024, and has implementations in a dozen languages. Anything XferLang claims about readability KDL can also claim.
- **JSON5 and HJSON** are JSON with comments and relaxed quoting. They win on zero learning curve.
- **CUE, Dhall, Pkl, Jsonnet and Nickel** own "configuration with logic". They are funded, have type systems designed by people who do that for a living, and each one decided deliberately how much computation to allow. XferLang's scripting direction competes with them and would lose.
- **YAML** is the incumbent for everything else, and its weaknesses (indentation, the Norway problem, implicit typing, escaping) are exactly where XferLang is strongest.
- **JSON** is the interchange format and will not be displaced by anything, because its value is universality rather than quality.

## 6. Viability

As a JSON replacement for machine-to-machine interchange, no. Nothing displaces JSON there, and XferLang should not try. The right relationship with JSON is to be a lossless superset for the typed cases JSON cannot carry (int64, decimal, dates), with a well-defined projection in both directions.

As a YAML replacement for human-written configuration and content, the design is good enough to compete and the implementation is not. The path TOML and KDL took is the only one that works: a frozen spec small enough to read in an afternoon, a language-neutral conformance suite with negative cases, at least one implementation in a language other than the author's, and a canonical serialization. Without those, nobody adopts a format, however good the syntax is. With them, broad adoption is still unlikely for an independent format in 2026, but a well-made small spec can hold a niche, and it is a legitimate goal to have one excellent format for your own portfolio of systems.

## 7. Niches worth owning

Three niches fit the language's actual strengths and have no strong incumbent.

**Content and configuration with embedded text.** Site navigation, CMS front matter, test fixtures, prompt templates, documentation metadata, anything where values are often paragraphs that contain quotes, HTML or code. The run-length delimiter makes this painless in XferLang and painful everywhere else. This is also the niche the one real consumer already lives in.

**Agent-authored data.** LLMs routinely break YAML indentation and JSON escaping, and both faults are invisible until a parser rejects the file. A format with no indentation sensitivity, no escape sequences, whitespace separation and explicit types is easier for a model to generate correctly and for a reader to check. Nobody is designing a data format for this audience yet, and it connects directly to the Overt work.

**Typed interchange inside a controlled estate.** Across Tela, Andoneer, PUDL and the site, where both ends are yours, XferLang can carry dates, decimals and int64 losslessly, and the JSON projection remains available for the boundary.

The scripting and Turing-complete direction is not a niche to own. CUE and Pkl are already there, and pursuing it would cost XferLang the simplicity that makes the three niches above reachable.

## 8. Where a reboot should go

The direction follows from the above. The reboot should produce a small, frozen data format with a specification that is the contract, and move everything that evaluates into a separate, optional layer that emits that format.

**Level 0, the data format.** Objects, sequences, strings with run-length delimiters, comments, booleans, null, integers of arbitrary written length, exact decimals, binary floats, and date, time, datetime and duration values. A single ABNF grammar that the parser is generated from or verified against. A canonical serialization. A conformance suite of positive and negative documents in a language-neutral form. Parsing is pure, does no I/O, and runs with depth, size and run-length limits. Diagnostics are structured results with codes and spans.

**Level 1, annotations.** Document metadata, element ids and tags, as grammar with no evaluation. These are needed by tooling and cost nothing in portability.

**Level 2, a preprocessor.** `let`, `if`, dynamic sources and anything else that computes, defined as a separate language that reads a Level 2 document and writes a Level 0 document. It is opt-in, never the default parse path, and a Level 0 parser never sees it. This is the shape the TURING_COMPLETE paper itself arrives at in its last paragraph.

**Schema** as a separate specification written in Level 0, replacing the dead `Schema/` folder and the two incompatible drafts in `schemas/`.

**Implementation** should start with the spec and the conformance suite, before any parser code, so that the second implementation can exist. The parser itself should be a lexer, a lossless syntax tree that preserves the source form, and an evaluator kept out of the core. Whether the reference implementation is C# (where the consumer is) or Rust with a C ABI (the stated roadmap) is a decision for the design phase; the spec-first order is what makes either one viable.

## 9. Design questions the reboot must settle

These are the open decisions a Level 0 spec has to answer, with the current state of each.

1. **Bare words as values.** Today `{ x hello }` is an error. Allowing bare identifiers as values would improve readability but collides with the recursive key/value rule (`key1 key2 "v"`), so one of the two has to go.
2. **Type sigils.** Keep the leading-character scheme, reduce it to a format's types rather than the CLR's, or move typing to a different position. The scheme is distinctive and worth keeping in some form, but the current set is both .NET-shaped and unmnemonic.
3. **Numbers.** One integer type with no written width, one exact decimal, one binary float, and an implicit form for all three.
4. **Array homogeneity.** Drop it from the grammar. It is a schema concern.
5. **Two escape mechanisms.** Run-length delimiters and `<...>` explicit forms solve the same problem. Interpolation currently depends on the explicit form; the spec should decide whether it needs two.
6. **Keyword characters.** Fix the bare-key alphabet to match what the site already uses, or require the quoted form; the spec and the parser must agree.
7. **Interpolation and references.** Decide whether they belong in Level 0 at all. The one real document uses neither.
8. **Processing instruction placement.** PIs currently attach to the next sibling and interleave with content. A reboot should decide whether metadata lives in a header, on elements as annotations, or both.

## 10. Summary

XferLang's syntax idea is good and deserves to survive. Its implementation is an experiment that has been allowed to grow a scripting language, and it has no contract. The reboot should freeze a small data format with a grammar, a conformance suite and a pure parser, move computation into an optional layer above it, and aim at the niches where the run-length delimiter and explicit typing are decisive advantages. On that footing it can be a real alternative to YAML for human-written and agent-written content, and a typed companion to JSON rather than a competitor to it.

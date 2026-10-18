# Adversarial review of REBOOT_PROPOSAL.md, round 2

Reviewed 2026-10-18 after all nine round-1 rulings were recorded and written into the spec sections. Same brief as round 1: read cold, try to break it. Grades as before: **blocking** needs a ruling; **required** is wrong or incomplete with a clear fix; **advisory** is worth knowing.

## Verdict

Viable for the spec to be written, subject to one formal step. Round 1's lexical gaps are closed, and the new rules compose: every example in 3.13 was re-checked token by token against the pinned separation, closing, and literal rules and holds. Round 2 found no new blocking gap in the design. It found one lexical hole that the round-1 ruling had opened, several places where the text had not caught up with the rulings, and the formal step below.

## The formal step

The decision tables in sections 3.2, 3.3, 3.4 and 3.7 still read "open": drop all sigils, one fractional type, symbols as a type, bare temporals, hash-run strings, remove the explicit `<...>` forms, any element at root. The round-1 rulings presuppose every one of them (there is no symbol trade without symbols, no inf/nan ruling without one fractional type, no exact-matching ruling without hash-run), and "proceed" was given after those rulings. The review does not treat that as a ruling on the tables. **Required:** a one-line confirmation of each, recorded in section 7, so the spec writer starts from rulings rather than inferences.

## Required findings, applied

**1. RFC 3339 allows a space between date and time.** `2025-08-01 09:30:00` is a valid RFC 3339 date-time, and under the separation rule it is two tokens: a date and a time. The round-1 ruling adopted the RFC's productions without excluding this. **Applied:** the separator is `T` only; lowercase `t` and the space are excluded (3.2).

**2. `!>` was not self-delimiting.** `<! xfer 1!>` was an error under the separation rule while `<! xfer 1 !>` was fine, for no reason an author would guess. **Applied:** `!>` joins the self-delimiting set (3.1).

**3. Keywords as bare keys were undefined.** `{ true 1 }` had no rule. **Applied:** a keyword is not an identifier; the key is written `"true"`; a bare word lexes the same way in every position (3.5).

**4. Unicode identifiers could not start with a non-ASCII character.** 3.5 said identifiers start with a letter or `_`, and 3.5a defined only continuation. **Applied:** any admitted non-ASCII code point may start an identifier (3.5a).

**5. Identifiers could end with `.`.** `a.` was a legal key, which bites in prose-like documents. **Applied:** an identifier may not end with `.` (3.5).

**6. Sign before radix integers was unspecified.** The current parser rejects `-$FF`; the proposal said nothing about `-0xFF`. **Applied:** a leading `-` is permitted before any integer form (3.2).

**7. Digit separators in fractional literals were unspecified.** **Applied:** `_` between digits anywhere in any numeric literal (3.2).

**8. The canonical choice rule could exceed the run-length limit.** Content containing a run of 64 quotes would make "longest plus one" exceed a 64 limit, though exact matching only needs the delimiter run to differ from every run inside. **Applied:** fall back to the smallest length not present among the content's runs, which always exists within the limit (3.3).

**9. Case of keywords was unstated.** `True` is a symbol under the rules, which is correct and also the likeliest typo. **Applied:** keywords are case-sensitive; a linter warns on a case-variant of a keyword; the parser does not, because a symbol is legal (3.2).

**10. Empty document and trailing content were undefined.** **Applied:** both are errors; concatenated document streams are not a 1.0 feature (3.7).

**11. Stale text.** 3.3 said line-ending normalization was still open and that "three cases" followed (there are four); the JSON mapping example still showed the big integer as a string after 3.12 changed it to a number; 5.1's migration list said the quoted title becomes `""...""`, which repetition cannot write, and omitted the head annotation and the `<"...">` strings. **Applied.**

**12. Limits did not cover token length or all three delimiter kinds.** An unbounded numeric lexeme is an unbounded token. **Applied:** scalar token length added; run-length limit stated to cover quotes, hashes and slashes (3.11).

**13. NUL in strings implies a C API shape.** Legal NUL content means the C API cannot return C strings. **Applied:** every string in every API is a pointer and a length (3.3). This belongs in the libxfer API design as a hard rule.

## Advisory

**14. Symbols look like versions and paths.** `v1.2.3`, `a.b.c` and `read-only` are all symbols. That is a feature for configuration, and the spec should show it, because readers of TOML will expect `a.b` to mean nesting and the proposal says it does not.

**15. `-inf` is a two-character-class token.** It begins with `-` like a number and continues with letters like a symbol. The lexer handles it as a keyword lookup after a `-` that is not followed by a digit; `-infinity` and `-x` are errors by the separation rule. Worth a conformance case each.

**16. Annotations on the root versus the `xfer` head.** `<! xfer 1 !> <! id "doc" !> { ... }` attaches both to the root, which is fine; the spec should say `xfer` must be first if present, or the "document head only" rule has no teeth.

**17. The whitespace-collapse test and comments.** Collapsing inter-token whitespace does not touch comment content, which is a token; the test description should say "between tokens" so nobody collapses inside comments and then wonders why the lossless trees differ.

**18. The agent-authorship measurement should be designed before the spec freezes,** not after, so the grammar's size is known when the "fits in a prompt" claim is made. One page of ABNF is the budget.

## What passing looks like

Round 2 passes when the formal step is recorded. No further design round is needed before the spec draft; round 3 should review the spec draft itself, with the ABNF and the first fifty conformance cases, rather than this proposal.

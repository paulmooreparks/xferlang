# XferLang: design and roadmap

Written 2026-10-18. This is the one document that ties the programme together. The reasoning behind each part lives in the documents it points to; this one says what is being built, in what order, how each piece depends on the others, when each is done, and where to begin. Decisions made against it are recorded in section 9.

## 1. What is being built

XferLang is a data format with the data model of JSON, the types a configuration or content file actually contains, no escape sequences anywhere in its grammar, and no meaning in line breaks or indentation. Its founding property is that any text can be placed inside a document unchanged at any depth, because the delimiter is lengthened to clear the content instead of the content being rewritten to clear the delimiter.

The programme produces:

1. **A specification and conformance suite** that are the contract, so that independent implementations can exist and be checked.
2. **Four implementations**, each independent, each passing the suite: JavaScript (`xferlang` on npm), .NET (`ParksComputing.XferLang` on NuGet), Go (`xfer-go`), and a hardened C reference (`libxfer`) with a stable ABI.
3. **A site**, xfer.parkscomputing.com, that is the language's home: documentation, a workbench with the language in its editor, a converter that shows what other formats cannot say, the real CLI in a browser terminal, and the conformance suite running live.
4. **Adoption across the portfolio**, beginning with parkscomputing.com and extending to the Go applications (Tela and others) once the Go implementation exists.

## 2. Design, by reference

| Concern | Where it is decided |
|---|---|
| Why the reboot, and what the old implementation got wrong | `history/REBOOT_ASSESSMENT.md` |
| The language design, the implementation strategy, positioning, and every ruling so far | `history/REBOOT_PROPOSAL.md` (section 7 is the decision log) |
| The adversarial reviews the proposal passed | `history/REBOOT_REVIEW_1.md`, `history/REBOOT_REVIEW_2.md` |
| The specification | `spec/xferlang.md`, `spec/xferlang.abnf` |
| The conformance suite and its formats | `tests/README.md`, `tests/valid.txtar`, `tests/invalid.txtar`, `tests/bytes/` |
| The JavaScript package | `xfer-js/DESIGN.md` (in its own repository) |
| The site | `XFER_SITE_DESIGN.md` (in the legacy repository's whitepapers for now; moves with the site) |

The design principles that cut across all of it, stated once:

- **The suite is the product.** A rule that has no conformance case does not exist. A rule changes in the same commit as its cases.
- **Implementations are independent.** No implementation is ported from another. When two disagree, the spec or the suite is corrected first, and then whichever implementation is wrong.
- **One module, every surface.** The JavaScript implementation is the same code in the browser editor, the browser terminal, the server behind MCP, and npm. No behaviour differs by surface.
- **Nothing in Level 0 or 1 evaluates.** Parsing never reads the environment or the file system and never throws. Anything that computes is a separate language that emits XferLang, and whether to build it is a question for after 1.0.
- **Industrial from the first line.** Sanitizers and fuzzing in the C core from its first commit; property tests for the invariants in every implementation; provenance-attested releases; stable diagnostic codes; normative limits.

## 3. The repositories

| Repository | Holds | Depends on |
|---|---|---|
| `xferlang` | spec, suite, this roadmap, history | nothing |
| `xfer-js` | TypeScript implementation, npm package `xferlang`, CLI | `xferlang` (suite as submodule) |
| `xfer-dotnet` | managed implementation, NuGet package, object mapper | `xferlang` |
| `xfer-go` | pure Go implementation | `xferlang` (after freeze) |
| `libxfer` | C11 reference, C ABI, `xfer` CLI, WASM build | `xferlang` (after freeze) |
| `xfer-vscode` | TextMate grammar, language server | `xfer-dotnet` or `libxfer` |
| xfer.parkscomputing.com content | the site's pages, applets, books | the parkscomputing engine, `xfer-js` |
| `Xfer` (legacy) | the 0.x library and history | archived as `xfer-legacy` at M5 |

## 4. Milestones

Each milestone has an exit criterion that can be checked rather than judged.

### M0. Contract drafted (done 2026-10-18)

Spec draft 1, ABNF, suite formats, about a hundred cases, the proposal closed with every decision recorded.

### M1. First implementation green

`xfer-js` passes every case in the suite and both property tests, on Node, in a browser, in Deno and in Bun. The spec and suite have absorbed every fault the implementation found (round 3 of review is this process, not a separate reading). The hand-computed diagnostic positions in `tests/invalid.txtar` have been verified by a real parser.

**Exit:** CI green on `xfer-js` with the suite pinned at a tagged commit of `xferlang`; `xferlang@0.1.0` published to npm as a prerelease.

### M2. Second implementation green, first consumer migrated

`xfer-dotnet` passes the same suite. The two implementations have been run against each other on the suite, on the legacy repository's sample files, and on generated inputs, and every disagreement has been resolved in the spec and suite first. parkscomputing.com reads `sitenav.xfer` and `admin-menu.xfer` through `xfer-dotnet`.

**Exit:** both CI green at the same `xferlang` tag; the live site serving from the new library; `ParksComputing.XferLang` 0.x published to NuGet.

### M3. The site, phase 1

xfer.parkscomputing.com stood up on the engine in desktop mode: the editor with XferLang highlighting and diagnostics linking to a page per code; `xfer` in the terminal; the JSON converter with loss flags; the specification as a book; the nest demonstration on the home page.

**Exit:** the site public; the npm README links to it; every diagnostic code has a page.

### M4. Freeze

Specification 1.0.0 tagged. Both implementations at 1.0.0. From here, every valid document stays valid and keeps its value within the major version.

**Exit:** the tag; `xferlang@1.0.0` and `ParksComputing.XferLang` 1.0.0 published; the site's book at 1.0.

### M5. Reference and Go

`libxfer` written against the frozen suite under the rules in the proposal's section 2.2, differentially tested against both existing implementations, with its WASM build running the suite on the site beside `xfer-js`. `xfer-go` written against the same suite. The legacy repository archived.

**Exit:** three or four implementations green at the 1.0.0 tag; the site's live suite page showing them agreeing; `libxfer` fuzzed for a stated number of CPU-hours with no findings; the first Go application reading XferLang.

### M6. The site, phase 2 and 3

TOML, YAML and KDL in the converter; the ten-minute course and the puzzle bank; semantic diff; share links; the assistant with tools; the site's actions over MCP; the agent-authorship measurement page with its results, whichever way they come out.

**Exit:** each item live; the measurement published.

### Later, not scheduled

The schema specification and workbench. The lossless formatter in the editor. Level 2, if it is ever worth building. `xfer-vscode` with the language server. Durations. Streams of documents.

## 5. Dependencies and the critical path

```
M0 spec+suite
  └─> M1 xfer-js ──┬─> M3 site phase 1
                   │
  └─> M2 xfer-dotnet ──┴─> M4 freeze ──> M5 libxfer + xfer-go ──> M6 site phases 2 and 3
```

`xfer-js` is on the critical path for both the site and the freeze. `xfer-dotnet` can proceed in parallel and is required for M2 and M4. Nothing after M4 can start before it, by design: the C core and the Go port are written once, against a fixed target.

## 6. Risks

- **One person, four implementations and a site.** The mitigation is the order above: the site needs only `xfer-js`; the post-freeze work is ports against a finished suite, which is the fastest kind of implementation to write; and agents can carry much of the port work when the suite is the oracle.
- **The spec is wrong somewhere.** It will be. M1 and M2 exist to find out before the freeze, and the rule that the suite changes in the same commit as the spec keeps the two from drifting.
- **The differential test finds a disagreement nobody can decide.** The decision log in the proposal is where it gets decided, by Paul, with the reasoning recorded.
- **C memory-safety findings after 1.0.** The fallback trigger is defined in the proposal (section 2.2): a written review on the first, a Rust rewrite behind the same ABI on the second within twelve months.
- **The agent-authorship claim does not hold.** Then it is not made; the measurement page says what was measured.
- **Scope creep on the site.** Phase 1 is deliberately what `xfer-js` gives for free. Anything not in section 4 of this document is not scheduled.

## 7. Where to begin

Begin with `xfer-js`, today, in this order, because it is on the critical path for everything and because the first implementation is the real review of the spec.

1. **Scaffold the package** as `xfer-js/DESIGN.md` describes: TypeScript strict, zero runtime dependencies, vitest, biome, the suite as a submodule, CI matrix. Half a day.
2. **The txtar runner and the conformance harness first**, before any parser: one generated test per case, all failing. This makes the suite executable and is the first thing the .NET implementation will copy in spirit.
3. **Normalization and the lexer** against the lexical cases (the `1xx`, `2xx`, `3xx` and `4xx` groups). The lexer is where the spec's pinned rules get their first real test, and where the hand-computed diagnostic positions get checked.
4. **The parser, the lossless tree and the value model** against the `5xx` and `7xx` groups.
5. **The canonical writer** against every `.canonical.xfer`, then the property tests.
6. **The JSON mapping and the CLI.**
7. **Publish `xferlang@0.1.0`** as a prerelease the moment the suite is green, so the site work can start against a real package.

Spec faults found along the way are fixed in `xferlang` in the same change as their cases, and the submodule is bumped. Expect the first week to produce as many spec commits as code commits; that is the process working.

In parallel, and independent of the JavaScript work: **stand up the site's container** (content volume, database, tunnel route, desktop mode, a `sitenav.xfer` that is itself the first document on the site), so that M3 is waiting only on `xfer-js`.

`xfer-dotnet` starts when `xfer-js` reaches step 4, so that its author (or its agent) inherits a suite whose positions have been verified and a spec that has survived one implementation.

## 8. How the work is tracked

Each milestone's steps are cards. The spec repository's issues hold spec and suite defects, one per finding, closed by the commit that fixes both. Each implementation repository's issues hold its own. The decision log stays in `history/REBOOT_PROPOSAL.md` section 7 until the freeze, after which decisions are recorded here in section 9.

## 9. Decisions recorded

(none yet against this document)

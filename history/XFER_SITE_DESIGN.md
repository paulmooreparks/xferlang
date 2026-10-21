# xfer.parkscomputing.com: a site for the language

Written 2026-10-18 from Paul's framing (a second site on the parkscomputing.com engine, XferLang everywhere, an editor with language support, a side-by-side converter that flags what the other format cannot say, guidance for beginners, the real CLI in the terminal) and from a survey of what the engine already provides. This is an ideas document and a phasing proposal, not a specification. Decisions go in section 7.

## 1. What the engine already gives us

The survey of the engine (2026-10-18) found that nearly everything the site needs exists as infrastructure:

- **Applets** are PUDL packages (`applet.json`: script, stylesheet, page, window shape, actions, continuity) that the engine catalogues from `wwwroot/applets/*/`, lists in the Applets menu, and exposes as pages and windows. Windows are URL-addressable (`?open=editor&p.editor=...`), so any state of the site is a link.
- **The virtual file system** has `~` in the browser's localStorage for guests (256 KB per file, 2 MB total, never sent to the server), `/site` built from `sitenav.xfer`, and server-side homes for admins. Every applet sees the same files and gets `pc:fs-change` events, including across tabs.
- **The terminal** is a custom shell with pipes, redirection, variables and `~/.profile`, with real interpreters (Lua, POSIX sh via WASM) in workers, and JavaScript modules can register commands with `pcTerminal.register({name, run(args, io)})` where `io` gives stdin, stdout, the file system and a pager.
- **The editor** is CodeMirror 6 with `linter`, `hoverTooltip` and `autocompletion` already exported from the bundle; adding a language is a mode in `entry.js` and two branches in `editor.js`. JSON already has a linter wired this way.
- **Pure applet actions** run on the server in Jint and are callable over the site's MCP endpoint, so a JavaScript function that validates or converts a document is, with no extra code, a tool an AI agent can call.
- **Chat** is an applet with selectable backends (local Ollama and cloud models).
- **Books** turn a folder of Markdown with front matter into a navigable reference with chapters and deep links; `import-mdbook.mjs` imports an existing book.
- **Diff Viewer**, **comments**, **RSS**, **Theme Studio**, passkey identity, roles, access tokens.

The consequence is that `xfer-js` is the only new thing the site strictly needs. The same module runs in the browser editor, in the terminal as the CLI, in Jint on the server behind MCP, and from npm on a developer's machine. One implementation, four surfaces, identical behaviour, which is a claim no other format's playground can make.

## 2. The workbench

The centre of the site is the editor with the language in it. Not a toy playground in a text box; the site's own Editor applet with XferLang as a first-class language.

- **Highlighting** from a Lezer grammar written from the spec's ABNF (a CodeMirror stream mode would do for a first cut, but run-length delimiters want a real tokenizer, and Lezer handles the `n` quotes then content then `n` quotes rule with an external tokenizer).
- **Diagnostics** from `xfer-js`, as CodeMirror lint marks, with the spec's codes. Each mark links to `xfer.parkscomputing.com/diag/XFER0404`, a page per code with the rule, an example, and the fix. The same code the CLI prints is the page the editor links to.
- **Hover** shows a value's type (integer, fractional, symbol, local date-time) and, for a string, its decoded content with the delimiter run explained; for a key, its path.
- **Completion** of keywords, annotation names, and keys seen elsewhere in the document.
- **Format** (canonical form, and later the lossless formatter) on a key or on save; **collapse to one line** and **expand** as a demonstration of the whitespace invariant.
- **Structure view**: a tree beside the text, PUDL's tree component, synchronized with the cursor.
- Files live in `~`, so a visitor's documents persist in their browser without an account; **share** produces a link with the document compressed into the URL, and members can save server-side later.

The browser is the language server. No round trip, no latency, and the identical module is what `npm install xferlang` gives. The engine's existing stdio LSP for VS Code is a separate deliverable (`xfer-vscode`) and shares the same parser once `xfer-dotnet` exists.

## 3. The converter

Two panes, XferLang on the left and a chosen format on the right (JSON first; then TOML, YAML, KDL, JSON5). Either side is editable; the other re-renders on each change.

The point of the applet is the flags. When XferLang says something the other format cannot, the loss is marked **on the other side**, in the place where the loss happens: a symbol became a string, a date became a string, an integer above 2^53 will round in JavaScript, a comment vanished, `inf` has no JSON form, an `id` annotation is gone. The right pane shows the format's escaping in a muted colour so the reader sees the backslashes they didn't have to write. In the other direction the flags are fewer and honest: YAML anchors and tags have no XferLang form, KDL's node model needs a convention, a JSON document with duplicate keys loses all but the last.

Two companions to the converter:

- **The nest demonstration.** Paste anything (a script, a regex, some HTML), choose a depth, and see it embedded one, two, three levels deep in XferLang beside the same in JSON, with the backslash count climbing on the right. This is the hook from the proposal (section 4a) made tangible in ten seconds.
- **Real-world samples.** A gallery of documents people actually fight with (a Kubernetes ConfigMap with JSON inside YAML, a GitHub Actions workflow with shell in strings, a `package.json`, a `.eslintrc`) shown in their native format and in XferLang, with the flags.

## 4. The terminal

`xfer` in the site's terminal is the npm CLI, because it is the same module loaded as a terminal command: `xfer validate`, `xfer fmt`, `xfer canon`, `xfer to-json`, `xfer from-json`, with pipes into and out of the existing `sh` and Lua, files in `~`, and the pager for long output. A visitor who learns the tool here has learned the real one. Two commands worth adding to the CLI because the site makes them visible:

- `xfer diff a.xfer b.xfer`: a **semantic diff** on the value model, ignoring formatting and comments, printed as a path-by-path change list. Nothing like it ships with JSON or YAML tooling by default, and the site's Diff Viewer can show it beside the textual diff.
- `xfer get <path> file.xfer`: read-only path lookup for scripts (`xfer get server.port config.xfer`). Lookup is not Level 2; it computes nothing.

## 5. Learning

- **The specification as a book**, through the books feature: the grammar with every production clickable, each rule linking to the conformance cases that exercise it, each diagnostic code with its page. Every code block in the book is a live editor instance, validated as the reader edits it.
- **"XferLang in ten minutes"**: a guided sequence where each step is a document the reader completes or repairs, checked by the parser in the page. The invalid cases of the conformance suite are a ready-made puzzle bank ("this document has one fault; find it").
- **"From JSON" and "From YAML"**: a page per incoming format, each a converter preloaded with that format's idioms and the one-sentence rule for each.
- **The conformance suite, live**: every case as a page, run in the visitor's browser against `xfer-js` on load, with a green or red mark. When `libxfer` and `xfer-go` exist as WebAssembly builds, the same page runs all three and shows them agreeing, which is the differential test made public. A "run the suite" button is the most credible thing a format site can have.

## 6. Agents

- **The assistant.** The Chat applet with a system prompt that knows the spec and with the validate, format and convert actions available as tools. "Why does this fail?" pastes the diagnostic; the assistant explains and offers the fix, and the fix is applied through the editor.
- **The site as a tool.** The pure actions (validate, canonical, to-json, from-json) are reachable over the site's MCP endpoint, so a developer's own Claude or any MCP client can use xfer.parkscomputing.com as its XferLang toolbox without installing anything.
- **The measurement.** The proposal's agent-authorship claim (section 4a) must be measured before it is made. The site is where the measurement lives: a page that, for a set of structures, asks several models to produce JSON, YAML, TOML and XferLang (with the one-page grammar in the prompt), validates each result with the real parsers, and publishes the error rates, re-run on a schedule. If the numbers hold, the headline is on the home page with the data behind it; if they don't, the page says so.

## 7. The site eats its own cooking

Everything the site reads is XferLang: `sitenav.xfer` already, then `applet.xfer` manifests in place of `applet.json`, the site's own configuration, Markdown front matter in XferLang (which closes the gap the assessment found, where front matter was eroding the format's one consumer), chat transcripts exported as XferLang, and the conformance suite served from the spec repository as a submodule. A visitor who views source anywhere sees the format in use.

## 8. Phasing

The site should go live early and grow, rather than wait for everything.

**Phase 1, with the first usable `xfer-js`.** The site stood up (new container, content volume, database, tunnel route, desktop mode); the Editor with highlighting and diagnostics; `xfer` in the terminal; the converter with JSON only; the spec as a book with the diagnostic pages; the nest demonstration on the home page. This is enough to be worth linking to from the npm package README.

**Phase 2, before the 1.0 freeze.** TOML, YAML and KDL in the converter with their flags; the live conformance suite; the ten-minute course and the puzzle bank; semantic diff; share links.

**Phase 3, after the freeze.** WebAssembly builds of `libxfer` and `xfer-go` running the suite side by side; the assistant with tools; the agent-authorship measurement page; MCP exposure of the actions; the schema workbench once the schema specification exists; the lossless formatter in the editor.

## 9. Decisions recorded

- **2026-10-22. The site runs on the parkscomputing engine as it is.** xfer.parkscomputing.com is another container from the engine image built in the parkscomputing repository, as padmajairam.com is, configured through the engine's `Site` settings in desktop mode. There is no fork of the engine. Features the site needs that the engine lacks (XferLang in the Editor, `xfer` in the terminal, serving a site's own tree beside the engine's built-in assets) are added to the engine for every site. The agent that maintains the parkscomputing repository does that work, from a brief this project writes.
- **2026-10-22. The site's content is the repository `xfer-site`,** mounted into the container, rather than a folder in OneDrive as for parkscomputing.com and padmajairam.com, so that the content is versioned and reviewed like code. It holds the pages, the book, the diagnostic pages, the site's own applets and its `sitenav.xfer`.
- **2026-10-22. The tunnel route exists.** Paul added xfer.parkscomputing.com to the Cloudflare tunnel. What remains is its entry in the tunnel's `config.yml`, once a container listens on a loopback port.
- **2026-10-22. `xfer` in the terminal is the npm CLI itself.** `xfer-js` exports the CLI as a function that takes its files and streams from the caller, so the Node command and the site's terminal run the same code and print the same bytes.

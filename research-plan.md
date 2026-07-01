# Research & Education Plan

**Talk:** Receipts from the Go Linker — Auditing the Cost of Dependencies
**Speaker:** Mikayla Toffler
**Updated:** 2026-06-28 — Talk in ~6 weeks → 2026-08-07
**Working assumption about you:** strong Go writer, library-maintainer instincts. Phase 1 complete. Cultural context (Go community podcasts, module history, origins of MVS) done on time off. Now entering the technical depth phase with 6 weeks left.

This plan is paced for **accessible, intuition-first** learning — blog posts, talks, hands-on labs — not academic papers. The goal is to make you *fluent*, not *encyclopedic*. You should be able to answer audience questions confidently and demo tools without notes.

---

## Status snapshot (as of 2026-06-28)

| Phase | Status |
|---|---|
| Phase 0 — Frame the topic | ✓ done |
| Phase 1 — Module fundamentals | ✓ done |
| Phase 1b — Cultural context (podcasts, history, comparative pkg managers) | ✓ done (time off) |
| Phase 2 — Toolchain mental model | → **current** |
| Phase 3 — Linker & DCE | not started |
| Phase 4 — Hands-on lab & demo | not started |
| Datadog prep | not started |
| Capstone tool | not started |
| Blog posts 2–6 | not started |

**Critical path:** Capstone tool must be live before talk day. Build it during Phase 4 — the implementation will reinforce what you're learning.

---

## Blog series arc

| Post | Working title | Status |
|---|---|---|
| 1 | What I thought I knew about Go modules | drafted |
| 1b | Go's dependency machinery is scar tissue | drafted |
| 2 | The Handoff: from compiler to linker | drafted |
| 3 | The linker, dead code elimination, and why `init()` is a landmine | not started |
| 4 | Auditing a real binary: receipts | not started |
| 5 | The attack surface you didn't know you imported: CVEs, reachability, and what `govulncheck` actually tells you | not started |
| 6 (capstone) | I built the tool: AI-assisted dependency vetting | not started |

Posts 2–5 can be rough drafts — they're the paper trail, not the headline. Post 6 must be polished and live before talk day.

---

## Week-by-week plan (6 weeks)

### Phase 2 (June 28 – July 4): Phase 2 — Toolchain mental model

**Goal:** Know what `go build` is *actually* doing. Be able to draw compile → link on a whiteboard.

**Read / watch**
- `go help build` and `go help buildmode` — yes, the help text. Surprisingly readable.
- Search YouTube for: "GopherCon Go compiler" or "GopherCon Go linker". Pick one ~25–40 min talk, most recent you can find.
- Dave Cheney's blog (`dave.cheney.net`) — search for "linker", "binary size", "init". Short, opinionated, ages well.
- `src/cmd/link/` in the Go source — read `doc.go` and any top-level comments. You want to see how the Go team frames its own component.

**Do**
1. `go build -x -o /tmp/hello ./hello.go` on a tiny program. Read the `-x` output. Identify each phase.
2. `go build -work .` and inspect the temp work directory.
3. `go tool compile -h` and `go tool link -h` — read the flag lists.
4. Inspect object files and the linker's resolution process: `cd` into a small project and run `WORK=$(go build -a -work -o /tmp/out . 2>&1 | grep WORK | awk -F= '{print $2}') && find $WORK -name "*.a"`. Pick `b001/_pkg_.a` (your package) and run `go tool nm $WORK/b001/_pkg_.a` to see the symbol table. Read the `importcfg` file in the same folder. Trace how `U` entries in the symbol table map to packages listed in `importcfg`. Compare the object file symbols to the final binary: `go tool nm /tmp/out | grep "main\."` — notice all `U` entries are gone.

**Self-check**
1. In your own words, what's the difference between `cmd/compile` and `cmd/link`?
2. What's an object file (`.a` archive)? What's inside one?
3. Where do `gopclntab`, `runtime`, and `reflect` come from in every binary?

---

### Phase 3 (July 5 – July 18): Phase 3 — The linker, DCE & the security angle

**Goal:** Be able to draw the mark-and-sweep on a whiteboard from memory. Connect DCE reachability to SCA reachability — this link is the talk's sharpest insight.

**Read / watch**
- **[Reducing the size of the Datadog Agent's Go binaries](https://www.datadoghq.com/blog/engineering/agent-go-binaries/)** — on-topic case study, internal authors you can reach.
- Search for: "Go linker dead code elimination" blog posts — Cloudflare, Tailscale, Uber, Datadog authors. These show *measured* before/after.
- GopherCon YouTube — "Go binary size" or "shrinking Go binaries". Even 2018–2022 talks cover concepts that are still current.
- **`src/cmd/link/internal/ld/deadcode.go`** in the Go source. Read the comments, not the code. The comments explain the algorithm in plain English better than any blog post.
- **`govulncheck` documentation** — specifically how it determines whether a vulnerability is "called" vs. "imported but unreachable." `govulncheck` performs its own call-graph reachability analysis, not just module-level matching. A CVE in a module you import may not appear in your results if the vulnerable function is never reachable from your code. This is DCE's logic applied to security — and it's a real, live connection between your two technical threads.
- **xz-utils backdoor (2024)** — a multi-year supply chain attack where a malicious actor built trust as a maintainer before inserting a payload. Use this as the anchor story for Post 5: it's not about a CVE in a library you're scanning for — it's about code that passes all your tooling and still ships malice. The uncomfortable implication: `govulncheck` and dependabot are necessary but not sufficient.
- Pierre is giving a GopherCon EU talk on this topic — reach out if you haven't yet. You have enough vocabulary now to ask good questions.
- **[prometheus/prometheus#18824](https://github.com/prometheus/prometheus/pull/18824)** — a colleague found this and it's directly applicable to dd-trace-go. The pattern: Prometheus was storing large SDK clients (Azure `armcompute`, `armnetwork`) as concrete types in a struct that satisfied an interface. Once a concrete type is reachable through an interface, the linker conservatively keeps *all* its exported methods — in this case ~60 per client, dragging in the full serializer graph even though only 2-3 methods were ever called. The fix: wrap each client in a small adapter that captures only the needed operations as method-value closures. The concrete client then lives only inside a closure context, which reflection cannot traverse, so DCE drops the unused operations. Result: `UsedInIface` markers drop from 244+66 to 0; binaries shrink ~3.2 MB each. Study this PR closely — the adapter/closure pattern is the technique your colleague's doc recommends applying to dd-trace-go's own large interface clients.

**Do**
- Build the same `main.go` against two libraries (one lean, one with a heavy `init()`). Compare:
  - `ls -l` binary size
  - `go tool nm -size -sort size <binary> | head -50`
  - `go version -m <binary>`
- Try `go build -ldflags="-s -w"` and measure the difference. Form an opinion on whether stripping is "free."
- Force a leak: write a tiny package whose `init()` references a heavy function. Confirm it survives DCE.
- **SCA/DCE experiment:** find a module with a known CVE in a specific function (check `pkg.go.dev/vuln`). Write a `main.go` that imports the module but never calls the vulnerable function. Run `govulncheck ./...` and observe whether it marks the vuln as "called" or just "imported." Then confirm via `go tool nm` that the vulnerable symbol is absent from the binary. This is the talk's sharpest slide: *the scanner and the linker agree on what's reachable.*
- **Prometheus adapter pattern applied to dd-trace-go:** identify one place in dd-trace-go where a large concrete client is stored as a field in a struct that satisfies an interface (your colleague's doc identifies candidates). Use `go tool nm -size -sort size` on a dd-trace-go binary to confirm the unused methods are present. Then apply the closure-adapter pattern from prometheus#18824 and measure the before/after. This is concrete Datadog-specific evidence for the talk — not just "here's a technique," but "here's what it did to our numbers."

**Self-check**
- Why is `init()` always reachable?
- Why does reflection defeat DCE?
- What does the linker do with an interface method set?
- What's the difference between *dead code* and *unreachable code*?
- What's the difference between a module-level CVE match and a reachability-confirmed one? When would you care?

---

### Phase 3b (woven into Phase 3): Post 5 prep — the attack surface angle

Post 5 ("The attack surface you didn't know you imported") needs its own prep thread. Work this in parallel with Phase 3.

**The core argument of Post 5:**
Every dependency you import is attack surface — not just code, but trust. A dep that never gets called still had to be fetched, verified, and compiled. If it had a malicious `init()`, DCE won't save you. The question isn't just "does this CVE affect me?" but "how much trust am I extending, and to whom?"

**Prep tasks:**
- Read the xz-utils post-mortem carefully. Identify: at what point would any Go tooling have caught this? (`govulncheck`? No — there was no CVE filed until after discovery. `go mod verify`? No — the checksum was valid. Dependabot? No — the version being pulled was the "latest.") Write down the honest answer: *nothing in our standard toolchain would have caught xz-utils early.* That's the uncomfortable truth that makes Post 5 worth reading.
- Look up how `govulncheck` uses the Go vulnerability database (`pkg.go.dev/vuln`) vs. how traditional SCA tools work (NVD/OSV module-level matching). Articulate the difference: govulncheck is reachability-aware; most SCA tools are not.
- **Datadog's security hygiene — honest assessment:** 
  - Does `govulncheck` run in dd-trace-go CI? (Check `.github/workflows/`.) If not, that's a gap worth naming on stage.
  - Does dependabot cover transitive deps or just direct ones? (It covers what's in `go.mod`, which includes indirects — but it doesn't know which ones are reachable.)
  - Is there any tooling that connects "this dep has a CVE" to "this vulnerable function is/isn't reachable in our binary"? If not — that's the gap the capstone tool could start to address.

---

### Phase 4 (July 12 – July 25): Phase 4 — Hands-on lab, demo, & Datadog audit

These two things happen in parallel. The Datadog audit *is* the real-binary work from Phase 4.

#### Demo (talk)

- Pick one lean utility module and one init-heavy module that solve roughly the same problem. Build a `main.go` that uses each. Capture:
  - binary size at: pristine, `-trimpath`, `-ldflags="-s -w"`, `-buildmode=pie`
  - symbol count, top-20 heaviest symbols
- **You must be able to run this demo live if your slides crash.** Practice until smooth.

#### Datadog audit (Q&A prep)

You are a dd-trace-go maintainer. Audience members will ask: *"I use Datadog tracing and you bloat my binary."* You need to be able to answer that with data, honesty, and a bit of humor. Here's the prep:

**Run the numbers on dd-trace-go**
- In your local checkout, build a minimal `main.go` that just imports `gopkg.in/DataDog/dd-trace-go.v2` and starts a tracer. Run:
  - `go mod graph | wc -l` — how many dependency edges?
  - `go tool nm -size -sort size <binary> | head -50` — what are the heaviest symbols?
  - `go version -m <binary>` — which modules made the cut?
- Find the top 3 surprising things in that output.

**Understand what we currently do**
- Check the `dependabot.yml` (or `.github/dependabot.yml`) in dd-trace-go. Know what it covers.
- Run `govulncheck ./...` on the repo. Know whether it's in CI.
- Look at `init()` functions across the main library packages: `grep -rn "^func init()" --include="*.go" .` — how many are there, and what do the heavy ones do?
- Look at package-level globals: `grep -rn "^var " --include="*.go" . | wc -l` — know the rough shape.

**Real tradeoff example: [dd-trace-go#4803](https://github.com/DataDog/dd-trace-go/pull/4803)**

You were a core reviewer on this PR. It adds OTel Go runtime metrics (`go.memory.used`, `go.goroutine.count`, `go.schedule.duration`, etc.) emitted from `ddtrace/tracer`. The central design question was: import the full OTel SDK into `ddtrace/tracer` (convenient for the feature, cascades as a transitive dep into every `contrib/` module), or import only the OTel **API** package (forces a small amount of manual SDK wiring for one metric, but keeps the SDK out of contrib's dep graph entirely).

The PR chose API-only. That choice was made explicitly to protect `contrib/` users from an uninvited transitive dependency — the exact tension the talk is about. Use this as a concrete example: *here's a real PR where we sat in the review thread and made a deliberate call about what we were willing to put in a customer's binary.*

**Prepare your honest answer**
Write 3–5 bullet points you could say on stage:
1. What dd-trace-go adds to a binary, and *why* (what functionality justifies it)
2. What we currently do for dependency hygiene (dependabot, govulncheck, etc.)
3. One honest gap — something we don't do yet but should
4. A humble acknowledgment that as a library maintainer, you now think about this differently

The goal is not defensiveness — it's the talk's thesis in miniature: *you understand the receipt now, here's ours.*

---

### Phase 5 (July 26 – August 1): Capstone tool + blog posts

**Capstone tool** (must be live before talk day)

The tool automates Cox's dependency evaluation checklist:
- maintenance posture (last commit, open issues, bus factor)
- vulnerability history (govulncheck or the vuln DB)
- transitive dependency count
- license
- security patch recency

Build it during this week. The talk's closing slide references it — "here's the tool that does everything we just covered, in one command." It needs to be deployed (even a simple `go install` or a hosted URL) and linked from the capstone blog post.

**Blog posts**
- Rough-draft posts 2 and 3 from your Phase 2 and 3 notes. They don't have to be polished — they're the paper trail.
- Rough-draft posts 4 and 5 from your demo/audit work.
- Write and polish **post 6 (capstone)** — this is the one that must be live.

---

### Week 6 (August 2–7): Final polish & readiness

**Talk polish**
- Replace all "illustrative numbers" in the outline with your real measured numbers.
- Fill the open question in Part 4 (the decision framework for audit output — *when* do you say "don't import this"?).
- Rehearse the demo until you can do it without looking at your hands.

**Readiness check — you're ready when you can, without notes:**
1. Define MVS in one sentence.
2. Draw the compile→link pipeline on a whiteboard.
3. Explain why DCE is a *graph reachability* problem.
4. Name the three leak patterns and give a one-line example of each.
5. Run `go tool nm -size -sort size` on a binary and read the top of the output out loud, identifying "expected" vs "suspicious" symbols.
6. Run `go mod why -m <pkg>` and explain the path to a non-Go engineer.
7. Tell one story of a surprising bloat source you found — ideally from the Datadog audit.
8. Answer *"I use Datadog and you bloat my binary"* with data, honesty, and a smile.
9. Explain the difference between a module-level CVE match and what `govulncheck` actually tells you. Why does reachability matter for security, not just binary size?
10. In one sentence: why wouldn't any standard Go tooling have caught the xz-utils backdoor?

If any of these aren't smooth, you know which week to revisit.

---

## Resource shortlist

| Resource | Format | Why |
|---|---|---|
| "Our Software Dependency Problem" — Russ Cox | Essay | Frames the *why* of the whole talk |
| Go Modules Reference — official | Docs | Canonical answer to "is this true?" |
| Dave Cheney's blog — "linker" / "init" / "binary" | Blog | Short, opinionated, accessible |
| `src/cmd/link/internal/ld/deadcode.go` | Source w/ comments | Authoritative on DCE algorithm |
| Datadog Agent Go binaries post | Blog | On-topic case study, internal authors |
| GopherCon YouTube — "binary size" / "shrinking Go binaries" | Video | Pick one ~30 min talk |
| `go help mod` and `go help build` | CLI | Written by the people who built it |

**Talk delivery — mentor recommendation:**
- **Bill Kennedy** (`@goinggodotnet` on X, Ardan Labs) — recommended by your mentor as a model for how to give an engaging, memorable Go talk. He runs the "Ultimate Go" training and is known for making low-level Go concepts land for a broad audience. Watch a few of his GopherCon talks specifically for *structure and delivery*, not just content: how he builds up mental models, when he slows down vs. speeds up, how he uses live code. This is about learning to teach, not just learning the material.

**Verify before citing on stage:**
- Specific posts from Filippo Valsorda on Go security/deps
- Specific posts from Jaana Dogan (rakyll) on Go performance
- Specific GopherCon talks by Austin Clements / Than McIntosh / Cherry Mui

---

## Capstone tool

**What it is:** A tool that automates Cox's "evaluate a dependency like a new hire" checklist — maintenance posture, vulnerability history, transitive dep count, license, last security patch.

**Narrative role:** The blog series teaches you *how* to evaluate a dependency manually. The tool is the synthesis: "you understand what I built, and you can use it."

**Talk ending:** After covering `go tool nm` + `go mod why` as the manual audit toolkit, the final slide references the tool. "We went over everything you need to decide whether to use a dependency. Now you can get the benefit of all that from this tool."

**Hard deadline:** Tool finished and capstone post live before 2026-08-07.

---

## A note on scope discipline

You will be tempted to learn about: linker relocations, DWARF debug info, Go's plan9-style assembly, the runtime's pclntab, escape analysis, generics monomorphization, …

**Don't.** They are interesting and almost none of them belong in this talk. If a rabbit hole doesn't pay off in audience understanding within 30 seconds of stage time, it's a blog post for later, not slide content for August.

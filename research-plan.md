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
| 5 | The attack surface you didn't know you imported | not started |
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

### Phase 3 (July 5 – July 18): Phase 3 — The linker & DCE

**Goal:** Be able to draw the mark-and-sweep on a whiteboard from memory. This is the technical heart of the talk.

**Read / watch**
- **[Reducing the size of the Datadog Agent's Go binaries](https://www.datadoghq.com/blog/engineering/agent-go-binaries/)** — on-topic case study, internal authors you can reach.
- Search for: "Go linker dead code elimination" blog posts — Cloudflare, Tailscale, Uber, Datadog authors. These show *measured* before/after.
- GopherCon YouTube — "Go binary size" or "shrinking Go binaries". Even 2018–2022 talks cover concepts that are still current.
- **`src/cmd/link/internal/ld/deadcode.go`** in the Go source. Read the comments, not the code. The comments explain the algorithm in plain English better than any blog post.
- Pierre is giving a GopherCon EU talk on this topic — reach out if you haven't yet. You have enough vocabulary now to ask good questions.

**Do**
- Build the same `main.go` against two libraries (one lean, one with a heavy `init()`). Compare:
  - `ls -l` binary size
  - `go tool nm -size -sort size <binary> | head -50`
  - `go version -m <binary>`
- Try `go build -ldflags="-s -w"` and measure the difference. Form an opinion on whether stripping is "free."
- Force a leak: write a tiny package whose `init()` references a heavy function. Confirm it survives DCE.

**Self-check**
- Why is `init()` always reachable?
- Why does reflection defeat DCE?
- What does the linker do with an interface method set?
- What's the difference between *dead code* and *unreachable code*?

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

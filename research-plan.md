# Research & Education Plan

**Talk:** Receipts from the Go Linker — Auditing the Cost of Dependencies
**Speaker:** Mikayla Toffler
**Today:** 2026-05-07 — Talk in ~3 months → ~2026-08-07
**Working assumption about you:** strong Go writer, library-maintainer instincts, lots of intuition about *impact* of bad dependency choices, but limited deliberate study of the dep-management *machinery* (modules, the toolchain, the linker, DCE).

This plan is paced for **accessible, intuition-first** learning — blog posts, talks, hands-on labs — not academic papers. The goal is to make you *fluent*, not *encyclopedic*. You should be able to answer audience questions confidently and demo tools without notes.

---

## Blog series arc

The posts are a paper trail of this learning journey, written from the voice of a library maintainer who realized she didn't fully understand the machinery she ships inside other people's binaries. Not a purist, not an importer — someone further along the path who discovered a gap.

Each post maps roughly to a phase. The arc across the series: *from vocabulary-without-mechanics → measurement-driven practice*.

| Post | Phase | Working title | Status |
|---|---|---|---|
| 1 | 0–1 | What I thought I knew about Go modules | drafted |
| 2 | 2 | What `go build` actually does | not started |
| 3 | 3 | The linker, dead code elimination, and why `init()` is a landmine | not started |
| 4 | 4 | Auditing a real binary: receipts | not started |
| 5 | 5 + security | The attack surface you didn't know you imported | not started |
| 6 (capstone) | — | I built the tool: AI-assisted dependency vetting | not started |

**The capstone post and talk ending are the same deliverable.** Post 6 introduces the tool; the talk closes with "you now know how to read the receipt manually — here's the tool that does it for you." The tool needs to be finished, live, and linked before talk day.

---

## How to use this plan

- One "phase" ≈ one to two weeks. Don't be a hero — short, regular sessions beat a marathon.
- Every phase has **(a) read/watch**, **(b) do**, **(c) self-check** prompts.
- If a self-check feels easy, skip ahead.
- Keep a `notes.md` file as you go — quotes, surprises, "wait, why?" moments. Those become slide content.

---

## Phase 0 — Frame the topic for yourself (now, ~2 days)

Before reading anything new, write one page in your own voice answering:

1. What does "dependency management" actually *do*, in your current mental model?
2. What's the difference between `go.mod`, `go.sum`, the module cache, and the build cache? (Even if you're guessing.)
3. What does a Go binary contain? Walk through it the way you'd walk a junior engineer through it.
4. When the linker runs, what does it have as input, and what does it produce?

You'll come back to this in Phase 5 and laugh at parts of it. That's the point — you're capturing your starting point so you can teach it.

---

## Phase 1 — Module fundamentals (~1 week)

**Goal:** Stop treating `go.mod` as magic.

### Read / watch

✓ **Go Modules Reference** (official) — `go.dev/ref/mod`. Skim, don't memorize. Know which sections exist so you can come back.
✓ **The Go Blog — "Using Go Modules" series.** 
- **Russ Cox — "Our Software Dependency Problem"** (2019, on `research.swtch.com`). Not Go-specific, but Russ wrote modules and this is *the* essay that frames the philosophy. Highest leverage single read on this list.
- **Sam Boyer — the history of Go dependency management.** Why did Go ship with GOPATH instead of a module system? Boyer drove the `dep` tool and the design work that eventually became Go modules. Understanding the *why* behind the evolution (GOPATH → dep → Go modules) gives you a story arc for the talk and the security post. Search for his GopherCon talks and his "The Cargo Cult of Versioning" writing.
- **npm left-pad incident** (2016) — a 17-line package being unpublished broke thousands of builds worldwide. Good concrete anchor for "why dependency hygiene matters" before you get to the Go-specific machinery.
- **xz-utils backdoor** (2024) — a multi-year supply chain attack. A malicious actor built trust over two years as a maintainer before inserting a payload. This is the "attack surface" post's anchor story.
- Understanding how other tools handle package management (other package managers, like ruby gems)

### Do

- In a scratch repo: `go mod init`, `go get` something, then read the resulting `go.mod` and `go.sum` line by line. Look up every directive (`require`, `replace`, `exclude`, `retract`, `toolchain`).
- Run `go env GOMODCACHE` — go to that directory, browse it. Internalize that the module cache is just files on disk.
- `go mod download -x github.com/something/something` and watch what it fetches.

### Self-check

- Explain `go.sum` to a friend in two sentences without using the word "hash."
- What's MVS (Minimum Version Selection)? Why does Go use it instead of "latest wins"?
- What does `go mod tidy` actually change?

---

## Phase 2 — The toolchain mental model (~1.5 weeks)

**Goal:** Know what `go build` is *actually* doing under the hood.

### Read / watch

- **`go help build` and `go help buildmode`** — yes, the help text. Surprisingly readable.
- **Search YouTube for: "GopherCon Go compiler" and "GopherCon Go linker".** There are several talks, including one by Austin Clements / Than McIntosh / Cherry Mui at various GopherCons walking through the compiler+linker pipeline. Pick whichever is most recent and ~25–40 min.
- **Dave Cheney's blog (`dave.cheney.net`)** — search his archive for "linker", "binary size", "init". He has accessible, short pieces from over the years. Not all are current, but the conceptual parts age well.
- **The Go source tree** — open `src/cmd/link/` and just read the top-level `doc.go` if it exists, plus `README.md`. You don't need to understand the code; you want to see how the Go team frames its own component.

### Do

- `go build -x -o /tmp/hello ./hello.go` on a tiny program. Read the `-x` output. Identify each phase (compile → link).
- `go build -work .` and inspect the temp work directory before it's deleted.
- `go tool compile -h` and `go tool link -h` — read the flag lists.

### Self-check

- In your own words, what's the difference between `cmd/compile` and `cmd/link`?
- What's an object file (`.a` archive)? What's inside?
- Where do `gopclntab`, `runtime`, and `reflect` come from in every binary?

---

## Phase 3 — The linker & Dead Code Elimination (~2 weeks — heart of the talk)

**Goal:** Be able to draw the mark-and-sweep on a whiteboard from memory.

### Reach out (in parallel — time-sensitive)

- Pierre is giving a talk at **GopherCon EU** on this exact topic — reach out now to sync. His GopherCon EU date sets the deadline; you want enough phase-2/3 vocabulary to ask good questions, but don't wait until you "feel ready."

### Read / watch

- **[Reducing the size of the Datadog Agent's Go binaries](https://www.datadoghq.com/blog/engineering/agent-go-binaries/)** — directly on-topic case study, internal authors you can reach.
- **Search for: "Go linker dead code elimination" blog posts.** There are several from individual engineers (often from Cloudflare, Tailscale, Uber, Datadog) walking through real shrinking exercises. These are gold because they show *measured* before/after.
- **GopherCon talks on binary size.** Search YouTube for "Go binary size" or "shrinking Go binaries". A handful of talks exist; even older ones (2018–2022) cover concepts that are still current.
- **`src/cmd/link/internal/ld/deadcode.go`** in the Go source. Read the comments, not the code. The comments explain the algorithm in plain English better than any blog post.
- **TinyGo's documentation** — even though TinyGo is a different compiler, their docs on "why is my binary small?" indirectly illuminate what mainline Go's linker doesn't do.

### Do

- Build the same `main.go` against two different libraries (one lean, one with a heavy `init`). Compare:
  - `ls -l` of the binary
  - `go tool nm -size -sort size <binary> | tail -50`
  - `go version -m <binary>`
- Try `go build -ldflags="-s -w"` and measure the difference. Form an opinion on whether stripping is "free."
- Force a leak: write a tiny package whose `init()` references a heavy function. Confirm the heavy function survives DCE.

### Self-check

- Why is `init()` always reachable?
- Why does reflection defeat DCE?
- What does the linker do with an interface method set?
- What's the difference between *dead code* and *unreachable code*?

---

## Phase 4 — Hands-on lab (~2 weeks)

This is where the demo for your talk is born. Treat it like a research notebook.

### Build the demo binary

- Pick **one** real, lean utility module and **one** real, init-heavy module that solve roughly the same problem. (Examples to consider: a tiny logger vs. a feature-rich logger; a minimal HTTP client wrapper vs. a heavy SDK.)
- Write a `main.go` that exercises one trivial function from each.
- Build both. Capture binary size, symbol count, top-20 heaviest symbols.
- Capture **the same numbers** at: pristine, with `-trimpath`, with `-ldflags="-s -w"`, with `-buildmode=pie`. Note which flags do anything.

### Build the audit story

- Pick a binary you actually use at work (with permission!) and run the same audit on it. Find one surprising symbol. `go mod why` it. Write down the story.
- That story is a slide.

### Self-check

- Could you do this audit live, on stage, if your demo crashed? (If no — practice until yes.)

---

## Phase 5 — Real-world stories & polish (~2 weeks)

**Goal:** You stop sounding like a textbook and start sounding like someone who's been in the trenches.

### Read

- Engineering blogs from companies that ship Go binaries at scale. Search for:
  - "Tailscale" + "binary size" or "Go"
  - "Cloudflare" + "Go" + "binary"
  - "Datadog" + "Go" + binary/size (you have insider access — talk to colleagues!)
  - "Discord" + "Go"
  - "Uber" + "Go"
- Note: companies often write *one* really good post on this topic and then move on. Look for posts from 2020–2025 with measured numbers.
- **Filippo Valsorda on dependabot and dependency maintenance** — Filippo is a Go security maintainer at Google who has written about the practice of keeping dependencies up to date as a security discipline, not just a chore. His perspective is a good counter-weight to pure binary-bloat framing: *why* you want fresh deps (security patches) vs. *why* you want fewer deps (smaller attack surface). That tension is the "attack surface" post.
- **Mitchell Hashimoto on dependencies** — he has a notable post/thread on minimizing dependency graphs in mature projects. Good practitioner voice to cite alongside Cox.

### Talk to humans

- Find one library maintainer outside Datadog (Slack, Gophers Slack, Twitter/Bluesky) who has thought about this. Ask: *"What's the most surprising bloat source you've found?"* Their answer is your best slide.
- At Datadog: ask the dd-trace-go team and any folks who've worked on `go-libddwaf` or shipped to customer binaries. They've felt this pain.

### Self-check

- Re-read your Phase 0 page. What's wrong? What's right? Update.
- Can you give the talk in 5 minutes? In 25? In 60? Each forces different prioritization.

---

## Resource shortlist (confident, accessible)

These are things I'm reasonably sure exist and are accessible. I've left URLs off intentionally — search by title and you'll find them.

| Resource | Format | Why |
|---|---|---|
| "Our Software Dependency Problem" — Russ Cox | Essay | Frames the *why* of the whole talk |
| Go Modules Reference — official | Docs | The canonical answer to "is this true?" |
| "Using Go Modules" / "Migrating to Go Modules" / "Publishing Go Modules" — Go Blog | Blog | Foundational, easy reading |
| Dave Cheney's blog — search "linker" / "init" / "binary" | Blog | Short, opinionated, accessible |
| `src/cmd/link/internal/ld/deadcode.go` — Go source | Source w/ comments | Authoritative on DCE |
| TinyGo docs — "Why is my binary small?" | Docs | Contrast helps |
| GopherCon YouTube — "linker" / "binary size" / "compiler" | Video | Pick one ~30 min talk |
| `go help mod` and `go help build` | CLI | Underrated, written by the people who built it |

**Resources I've heard of but can't vouch for off the top of my head** (verify before citing on stage):
- Specific posts from Filippo Valsorda on Go security/deps
- Specific posts from Jaana Dogan (rakyll) on Go performance
- Specific GopherCon talks by Austin Clements / Than McIntosh / Cherry Mui

If you want, hand me back any of these later and I'll help you confirm they exist and summarize them.

---

## Capstone: AI-assisted dependency vetting tool

*Idea flagged by mentor as potentially high-traffic. This is the talk's closing deliverable.*

**What it is:** A tool that automates Cox's "evaluate a dependency like a new hire" checklist — maintenance posture, vulnerability history, transitive dep count, license, last security patch. Readers/audience leave with something they can run before every `go get`.

**Narrative role:** The entire blog series and talk teach you *how* to evaluate a dependency manually (modules → toolchain → linker → DCE → audit commands). The tool is the synthesis: "you now understand what I built, and you can use it." It closes the loop between the technical education and a practical daily-use artifact.

**Talk ending:** After covering `go tool nm` + `go mod why` as the manual audit toolkit, the final slide references the tool. "We went over everything you need to decide whether to use a dependency. Now you can get the benefit of all that from this tool."

**Timeline constraint:** Tool must be finished and the capstone post must be live before talk day (~2026-08-07). Build it during Phase 4–5 prep, in parallel with the hands-on lab work — the implementation will reinforce what you're learning.

---

## Readiness check (do this 2 weeks before the talk)

You're ready when you can, **without notes**:

1. Define MVS in one sentence.
2. Draw the compile→link pipeline on a whiteboard.
3. Explain why DCE is a *graph reachability* problem.
4. Name the three leak patterns and give a one-line example of each.
5. Run `go tool nm -size -sort size` on a binary and read the top of the output out loud, identifying which symbols are "expected" and which are "suspicious."
6. Run `go mod why -m <pkg>` and explain the path to a non-Go engineer.
7. Tell **one story** of a real surprising bloat source you (or someone) found.

If any of those aren't smooth, you know exactly which phase to revisit.

---

## A note on scope discipline

You will be tempted to learn about: linker relocations, DWARF debug info, Go's plan9-style assembly, the runtime's pclntab, escape analysis, generics monomorphization, …

**Don't.** They are interesting and almost none of them belong in this talk. If a rabbit hole doesn't pay off in audience understanding within 30 seconds of stage time, it's a blog post for later, not slide content for August.

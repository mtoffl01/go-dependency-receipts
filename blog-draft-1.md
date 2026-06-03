# What I thought I knew about Go modules

*Post 1 of N. I'm preparing a GopherCon 2026 talk called "Receipts from the Go Linker — Auditing the Cost of Dependencies." This blog is the receipts on my own learning: what I knew, what I assumed, and where I learned.*

---

## Why this post exists

I've been writing Go for almost four years, and officially maintaining Datadog's Go Instrumentation tool (dd-trace-go and, more recently, orchestrion) for two. My tool lives inside our customers' binaries -- performance, security, and UX are all crucially important to its success. 

One key component of our binary -- which is a dependency for our customers' binaries -- is the dependencies that we import. That should mean I have a strong mental model of Go's dependency machinery — modules, the module cache, the build cache, the linker, dead code elimination, etc. 

Not exactly...

I came in with a *user's* model: I kinda knew which commands did what, and vaguely knew what a `go.mod` looks like. I had intuitions about which dependencies are "heavy" and potentially unreliable, but not enough to feel confident in my PRs that introduce new libraries or to block others' code on the basis of bad import practice. And I knew nothing at all about the compile → link pipeline, Dead Code Elimination, and how our binaries are really created.

Many junior developers fall into two extreme (and under-informed) categories:

- **The Purist:** *"No imports. Why depend on anyone else?"*
- **The Importer:** *"Import everything. Don't reinvent the wheel."*

Both are mantras; neither is critical thinking. The position I seek — and the one I think the language deserves — is **measurement-driven**: every import is a transaction, we just need to learn how to measure the costs.

In order to get there, I've got a lot to learn. So, I decided to pursue a phased study plan, and blog each phase. This is Phase 0 + Phase 1.

---

## Phase 0 — My starting point, before I learn anything new

The exercise: from memory, write down what I *think* is true. The point is to capture my starting model so that I can track my learning.

Here's what I wrote, unedited:

### Q: What does "dependency management" actually do, in your current mental model?

> Dependency management is the practice of thinking critically about your dependencies. By importing a third-party library, you avoid "reinventing the wheel" — you use logic that has likely already been vetted over time and across other users with the same goals. So you not only save time by not reimplementing things, but you also get better quality code in many cases. But importing third-party code comes at a cost: you become susceptible to their vulnerabilities, and your binary carries records of their source code. Your binary increases in size because of theirs. Multiply this by the number of libraries you import.

### Q: What's the difference between `go.mod`, `go.sum`, the module cache, and the build cache?

> `go.mod` is a list of all of your direct dependencies. `go.sum` is a list of all your direct and indirect dependencies and the commit SHA of them that your binary uses. The module cache is a local, temporary download of all the modules' code that you will use. The build cache is all of that compiled into symbols.

### Q: What does a Go binary contain? Walk through it the way you'd walk a junior engineer through it.

> It contains all of the compiled symbols of "reachable" code from your source code and the imports.

### Q: When the linker runs, what does it have as input, and what does it produce?

> N/A.

---

## Phase 1 — What I learned reading the actual docs


### I was wrong on both ends of my `go.sum` answer

My blind answer said `go.mod` tracks direct deps and `go.sum` tracks direct + indirect deps with their commit SHAs.

But `go.mod` also includes indirect imports — marked with an `// indirect` directive. And `go.sum` doesn't record commit SHAs at all. It stores "cryptographic hashes of the module zip files" ([go.dev/ref/mod](https://go.dev/ref/mod))...

...*What the hell is that?*

The hashes attached to your dependencies in go.sum are effectively a security check. When you download a dependency and version for your project locally (`go get ...`), Go computes a hash for the module zip file it's fetching, and stores that hash in your `go.sum`. The next time someone (or even CI) uses your program and fetches that dependency, `go get` runs, re-computes the module hash, and compares it against the one listed for that depdency in the `go.sum`. If they don't match, the build fails. This protects against an attack attempting to inject into your code and serve you different code under the same version tag. The design is deliberately paranoid. <Discuss xz utils backdoor incident here>

### The module cache is permanent, not temporary

I called it "a local, temporary download." It's not temporary — it's shared across all your projects and persists at `$GOPATH/pkg/mod` (or wherever `GOMODCACHE` points). It's a directory full of source trees organized by module path and version. 

### Modules vs. packages — a distinction I'd been lazily ignoring

A **module** is a collection of packages released, versioned, and distributed together. A **package** lives at a specific import path inside a module. For years, I'd been using the words interchangeably in my head, but they are distinct.

### Reading Russ Cox's "Our Software Dependency Problem"

This is a foundational essay on Go's dependency philosophy, written in 2019 by the person who designed the Go module system. In the opening paragraph, he states:

> "The shift to easy, fine-grained software reuse has happened so quickly that we do not yet understand the best practices for choosing and using dependencies effectively, or even for deciding when they are appropriate and when not."

That's the entire thesis of the learnign exploration I am on -- written almost seven years ago! 

Cox argues you that should evaluate a new dependency the way you'd evaluate a new hire at your company --  "interviews," check its references, audit its maintenance posture, and look at its own transitive dependencies. I was surprised by the rigor he proposed -- honestly, I don't know anybody who does all this! But is "nobody does this" a fair defense against his proposal? Automation tooling has exploded since 2019, especially with the rapid advancements in AI. So while the manual process he proposed may sound impractical, the *aspiration* still fits. These two criteria, in particular, stand out:

1. **Maintenance posture** — is the project actually maintained? Star count and a recent commit are not enough. A project that hasn't had a security patch in two years is a liability.
2. **Transitive dependencies** — how much code does the code you're depending on *itself* depend on? This is the question the rest of this series is going to answer mechanically.

Given how much tooling has advanced, Cox's checklist could be fused into a script -- like an automated pre-import audit tool. Perhaps that's something I can create by the end of my learning journey.

### MVS — Minimum Version Selection

When your dependency graph has multiple packages that all require the same library, the resolver has to pick one version. One approach is: pick the highest/newest version that satisfies everyone's constraints. The intuition is "newer is better" because newer versions get bug fixes and have the latest features.

But that's not what Go does; Go uses *Minimum Version Selection*: given the constraints in your `go.mod` and your transitive dependencies' `go.mod` files, Go picks the *minimum* version that satisfies all of them. This is because Go prioritizes stability; new might be shiny, but it often breaks things too. 

---

## What surprised me most

Given everything I learned today, the fact that we can pull in third party code at all -- and with a single command -- seems like a modern miracle! Before package managers, you found a library, copied the source files manually, tracked the version yourself, and maintained that copy forever. `go get` seems obvious to us now, but it wasn't always.

And at the same time, I'm shocked we still do this as casually as we do. We know about incidents like xz-utils and left-pad; we know that every dependency is code we ship, vouch for, and inherit the vulnerabilities of. And yet `go get` remains a one-liner most of us run without a second thought.

## What's next

Phase 2 is the toolchain itself: what `go build` actually does, what an object file is, what `cmd/compile` and `cmd/link` each contribute. Then Phase 3 is the linker and DCE in depth — the heart of the talk. I'll write a post per phase.

If you're a Go developer and any of this feels uncomfortably familiar — vocabulary without mechanics — that's exactly the audience I'm writing the talk for. More soon.


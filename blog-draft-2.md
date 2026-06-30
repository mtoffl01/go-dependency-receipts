# Go's dependency machinery is scar tissue

*Post 2 of <I don't know how many yet>, Phase 1 continued. I'm preparing a GopherCon 2026 talk called "Receipts from the Go Linker — Auditing the Cost of Dependencies." This blog is the receipts on my own learning: what I knew, what I assumed, and where I learned.*

---

## Where we left off

In Post 1, I corrected my mental model of the mechanics: `go.sum` stores hashes, not commit SHAs. The module cache is global and permanent, not temporary. MVS picks the *minimum* satisfying version, not the newest.

But I kept bumping into a question I hadn't answered: *why does Go's system look this specific way?* Why hashes? Why minimum versions? Why a central checksum database?

The answer, it turns out, is that Go's dependency machinery isn't a clean design from first principles. It's a system built by people who watched other ecosystems fail — and encoded those failures into the tooling. Every decision has a backstory. Today I went looking for those stories.

---

## How we got here

I started with a 2016 essay by Sam Boyer — ["The Saga of Go Dependency Management"](https://blog.gopheracademy.com/advent-2016/saga-go-dependency-management/) — written while he was building `dep`, the tool that eventually became Go modules. It's a document that reads like a postmortem.

Go 1.0 shipped with `go get` as the only dependency tool. `go get` pulled the latest code from the default branch of a repository — no version pinning, no lockfile, no reproducibility. Run it on Monday, get one version of a library. Run it on Friday, get a different one. Two teammates could be building against completely different dependency code and have no idea.

There's a cultural side effect worth naming: because `go get` ignored version tags, most Go developers stopped tagging releases. Why tag if nothing uses the tags? Which made things worse — when people eventually tried to build versioning tools, there was nothing to pin to.

The community's first real fix was a convention: the `vendor/` directory. Instead of having everyone fetch dependencies from the internet, you committed your dependencies' source code directly into your own repository. All of it — the entire library, not just what you used. Your repo got bigger, but your builds became reproducible. Everyone on the team built against the same code because it was *right there, checked in*.

Go 1.5 formalized `vendor/` as an official mechanism. Go 1.6 turned it on by default. That's the inflection point Boyer was writing from — the community finally had a blessed approach to encapsulation, but no unified tooling story. `dep` was the attempt to provide one, and Go modules were the eventual answer.

The point isn't the tooling timeline. The point is that **Go's module system was built by people who had spent years watching `go get` fail in production, watching the community fragment around a dozen incompatible tools, and watching the absence of versioning create real incidents.** That context is baked into every design decision.

---

## What the machinery is defending against

Once I had that historical frame, I went looking for the specific incidents that shaped the threat model. Two stand out.

### Left-pad (2016): the availability story

In 2016, a developer unpublished a small npm package called `left-pad` — 11 lines of JavaScript that left-pads a string. Thousands of projects that depended on it, directly or transitively, broke immediately. The Node.js build for React failed. The Babel build failed. CI pipelines across the industry went red simultaneously.

The attack surface here wasn't malicious code — it was *availability*. The open source ecosystem had quietly built a dependency on a single individual's willingness to keep a package published. Nobody audited that risk because nobody thought about it. npm had optimized so aggressively for reuse that the ecosystem had become fragile in ways nobody had measured.

Go's response is partly structural: the module proxy (`proxy.golang.org`) caches module versions centrally. If an author deletes a repo, the proxy still has it. The ecosystem doesn't break because one person changed their mind.

### XZ Utils (2024): the trust story

XZ Utils is a compression library that lives deep in the dependency trees of essentially every Linux distribution — usually as a transitive dependency of systemd. In 2024, a backdoor was discovered in versions 5.6.0 and 5.6.1.

The attacker hadn't found a code vulnerability. They had spent *two years* building trust as a legitimate contributor to the project — submitting real fixes, being helpful, gradually earning commit access — before inserting the payload into a release. The maintainer wasn't negligent; they were targeted.

The xz-utils attack reframes the dependency risk model entirely. Your attack surface isn't just the code in your `go.sum`. It's every human who has ever had commit access to every module in your dependency graph. You can audit source code. You can't audit the social engineering of a maintainer you've never heard of.

This is also the most honest argument against "never upgrade dependencies unless forced." It sounds cautious, but it implicitly assumes that the version you're pinned to is safe — and the xz-utils backdoor made it into official stable release tarballs. Being pinned to 5.6.0 didn't save you. Being pinned to 5.5.x did. The only defense was knowing *which* version was clean, which requires active monitoring, not avoidance.

---

## How other ecosystems made different choices

Go wasn't building in a vacuum. It had decades of package management history to learn from — and in some cases, deliberately diverge from.

**Bundler (Ruby)** pioneered the pattern Go modules are built on: a human-editable manifest (`Gemfile`) expressing intent, plus a machine-generated lockfile (`Gemfile.lock`) pinning exact versions. This is structurally identical to `go.mod` + `go.sum`. Bundler proved around 2010 that the two-file model works at scale. Go didn't invent it; Go inherited it.

**npm (Node)** is the cautionary tale. It optimized for reuse so aggressively that it produced `node_modules` folders with thousands of packages, many maintained by a single person, most never audited. Left-pad was a symptom of that culture, not an anomaly. Go's emphasis on a relatively flat dependency graph and the `go mod tidy` discipline of pruning unused deps is a conscious reaction.

**Cargo (Rust)** is the closest direct ancestor to Go modules. Go borrowed semantic versioning as a first-class contract, the lockfile pattern, and the central registry model. But Go diverged on one critical decision: version selection. Cargo (like most package managers) uses *maximal* version selection — when resolving a conflict, pick the newest version that satisfies all constraints. Go uses *Minimum Version Selection*: pick the oldest version that satisfies constraints. You only get a newer version when you explicitly ask for it.

Sam Boyer wrote an essay called "The Cargo Cult of Versioning" specifically about this decision — whether Go should just copy Rust's approach. The Go team's bet was that MVS produces more predictable, reproducible builds. Newer might be shinier, but it breaks things more often. The goal isn't to always have the latest; the goal is to always have the same thing everyone else has.

---

## Reading go.sum with new eyes

After all of this, I went back to my scratch module from the last post and re-read the `go.sum` file.

```
github.com/DataDog/dd-trace-go/v2 v2.8.2 h1:ZqF2M7j5DPG7PxkJpLIjF4L62LU/QnI86oOSAZjQC/U=
github.com/DataDog/dd-trace-go/v2 v2.8.2/go.mod h1:o+fhXzd1mPT4Ji5YYcqIjORnNKWcS6m2eW4xqdJplRA=
```

Two lines. I'd already understood the mechanics — one hash for the source tree, one for the go.mod. But now I could see the threat model in them.

The first hash is the xz-utils defense: if someone swaps the source behind a version tag, this hash won't match and your build fails. The second hash exists because Go fetches go.mod files *before* fetching full source trees (to resolve the dependency graph cheaply without downloading everything) — so it needs to verify that file independently too.

Two lines, two attack vectors, two checksums. It's not bureaucracy. It's paranoia that earned its place.

I also experimented with `go mod download` vs `go get` and found a subtle distinction: `go get` adds the dependency to `go.mod` *and* downloads it. `go mod download` only downloads and verifies — it'll add entries to `go.sum` but leaves `go.mod` untouched. Running `go mod tidy` after will clean up any orphaned `go.sum` entries for modules that aren't actually declared. The separation exists because `go mod download` is mainly useful for pre-warming caches (like in a Dockerfile layer before copying source), not for actually declaring dependencies.

---

## What surprised me most

The thing that stuck with me after today: **the open source trust model is a handshake, not a contract.**

When you run `go get`, you're trusting a version tag, which was created by a human, whose commit access was granted by another human, based on a track record of contributions that could have been manufactured. The go.sum hashes verify the *code*. They can't verify the *person*.

The xz-utils attacker didn't break any of the technical safeguards. They worked around them entirely, through social engineering that played out over years. That's the part of the attack surface I hadn't thought about before — and it's the part that no checksum database can protect against.

This doesn't mean the technical safeguards aren't worth having. It means they're necessary but not sufficient.

---

## What's next

Phase 2 is the toolchain itself: what `go build` actually does, what an object file is, what `cmd/compile` and `cmd/link` each contribute. Then Phase 3 is the linker and DCE in depth — the heart of the talk.

If any of this reframed how you think about your `go.sum`, that's exactly the point. More soon.

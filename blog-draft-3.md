# The Handoff: from compiler to linker

*Post 3 of <I don't know how many yet>, Phase 2. I'm preparing a GopherCon 2026 talk called "Receipts from the Go Linker — Auditing the Cost of Dependencies." This blog is the receipts on my own learning: what I knew, what I assumed, and where I learned.*

---

## Where we left off

In Posts 1 and 2, I covered the *why* of Go's dependency machinery — the historical context, the incidents, the design decisions. I came away with a clearer model of what `go.mod` and `go.sum` are defending against, and a better read on why Go made the choices it did.

But I still had a gap: I knew *what* was in my dependency graph. I had no mental model of what happened to it at build time.

Phase 2 is where I filled that in.

---

## I thought `go build` was one thing

My blind answer from Phase 0, on what the linker receives: *"N/A."*

I'd been running `go build` for years without thinking about what it was doing internally. It was a black box with a satisfying output: a binary. The steps in between were invisible to me.

Turns out `go build` isn't one thing. It's a pipeline with a clear seam in the middle:

1. **Compile** — `cmd/compile` turns each `.go` package into an object file
2. **Link** — `cmd/link` takes all those object files and produces the final binary

The seam between them — the handoff — is what this post is about.

You can watch the whole pipeline happen in real time with one flag:

```bash
go build -x -o ./go-handoff-demo .
```

The `-x` flag prints every command the toolchain runs. The output is overwhelming if you have many dependencies — one compile invocation per package, then a single link invocation at the end. When packages are cached (which they often are), the compile steps are invisible and only the link step shows. Run `go build -a -x` to force a full recompile and see the whole thing.

---

## What the compiler actually produces

For each package in your build — yours, and every transitive dependency — the compiler produces a `.a` file: an archive of compiled machine code for the target architecture. One `.a` file per package.

These don't end up in your binary directly. They're handed to the linker first.

The source for this demo is a minimal program — two functions, one import:

```go
package main

import "fmt"

func Add(a, b int) int {
    return a + b
}

func main() {
    fmt.Println(Add(1, 2))
}
```

To see the object files, run:

```bash
cd /Users/mikayla.toffler/testrepodir/go-dependency-play/phase2
WORK=$(go build -a -work -o ./go-handoff-demo . 2>&1 | grep WORK | awk -F= '{print $2}') && find $WORK -name "*.a"
```

The `-work` flag keeps the temporary work directory instead of cleaning it up. The output looks like this (truncated):

```
/var/folders/.../go-build254363032/b054/_pkg_.a
/var/folders/.../go-build254363032/b038/_pkg_.a
/var/folders/.../go-build254363032/b009/_pkg_.a
/var/folders/.../go-build254363032/b001/_pkg_.a
... (59 total)
```

59 object files. One import of `fmt`. There is one folder per package — yours and every package in your transitive graph. In a single-package program, `b001` is your code; in a multi-package module, several slots will be yours. With dd-trace-go as a dependency, the count climbs significantly higher. These folders are the compile step's output — everything the linker is about to receive.

---

## Reading an object file

This is where it gets interesting. Each `_pkg_.a` has a symbol table — a map of every name the package defines or references. You can read it:

```bash
go tool nm $WORK/b001/_pkg_.a
```

The output has two kinds of entries. First, the symbols *defined* in this package — these have a hex offset marking their position within the file, and a type:

```
3b43 T main.Add       ← T = code defined here (your function)
3b53 T main.main      ← another defined function
3bc3 D main..inittask ← D = initialized data (compiler-generated)
```

Second, the symbols this package *needs but doesn't have*:

```
     U fmt..inittask
     U fmt.Fprintln
     U os.(*File).Write
     U os.Stdout
     U sync/atomic.(*Pointer[os.dirInfo]).Load
     ... (and more)
```

`U` means undefined. The compiler left these as placeholders — "I know I need this, I don't know where it lives yet." Filling in every `U` is the linker's primary job.

Notice what's in the `U` list: not just `fmt.Fprintln`, which I explicitly called, but `os.(*File).Write`, `os.Stdout`, and `sync/atomic` methods I never mentioned. Some of these appear because the compiler *inlined* parts of the `fmt.Fprintln` call chain directly into my package — copying that code into my object file rather than generating a call instruction. Once inlined, that code directly references `os.Stdout`, so the reference shows up as a `U` entry in *my* object file. This happens at compile time, before the linker has even started. (Dave Cheney's blog has an accessible treatment of Go inlining if you want to go deeper.)

---

## The importcfg: how the linker knows where to look

Each `b0XX` folder also contains an `importcfg` file:

```bash
cat $WORK/b001/importcfg
```

```
# import config
packagefile fmt=/var/folders/.../go-build1185586193/b002/_pkg_.a
packagefile runtime=/var/folders/.../go-build1185586193/b009/_pkg_.a
```

This maps import paths to the location of their compiled `.a` files. `b001` lists what *your* package imported; `b002` (fmt) lists what *fmt* imported; and so on down the chain.

The linker doesn't traverse these files one at a time in order. It does two passes:

1. **Build the index** — read all `importcfg` files across all packages, producing one global map of `package name → .a file location`
2. **Resolve** — collect every `U` entry across all `_pkg_.a` files, then match each one against the index, find the corresponding `T`, and wire them together with a real memory address

By the time the linker is done, every Go package symbol is resolved. You can verify — filter for Go symbols specifically:

```bash
go tool nm ./go-handoff-demo | grep "^ *U [^_]"
```

Nothing from Go packages. What *does* remain as `U` in the final binary are C library and OS syscall symbols — `_pthread_create`, `_mmap`, `_write`, and others that the Go runtime calls at a lower level. These aren't resolved by the Go linker; they're resolved by the OS dynamic linker at runtime from system libraries. They were never in any `_pkg_.a` file — they live outside Go's build graph entirely.

---

## The thing that surprised me most

The `// indirect` marker in `go.mod` felt like a meaningful distinction to me. Direct dependencies are *my* imports; indirect ones are *their* imports — one step removed, someone else's concern.

The linker doesn't know the difference.

It receives a flat pile of `.a` files. It doesn't consult `go.mod`. It doesn't care how a package ended up in the build graph — whether you explicitly imported it or it arrived ten levels deep through a transitive chain. A deeply indirect dependency with a heavy `init()` function hits your binary just like anything you deliberately chose.

The `// indirect` label is bookkeeping for humans. To the linker, it's invisible.

---

## One more thing: build mode is the linker's concern, not the compiler's

While I was poking at object files, I noticed that the hex offsets in the symbol table — `3b17 T main.Add` — are offset positions *within* the object file, not their final memory addresses. The linker assigns real addresses later — and whether it assigns absolute addresses or relative ones depends on the build mode.

This is why build mode (specifically `-buildmode=pie`) is a linker-time decision, not a compiler-time one. The compiler always produces the same object files regardless of build mode. What changes is what the linker does with the offsets: in non-PIE mode, it assigns absolute memory addresses; in PIE mode, it emits relative offsets and writes a relocation table so the OS loader can patch addresses at runtime. Same input; different output.

On macOS with Apple Silicon, PIE is the default. If you're on a mac, you are already paying for the relocation table by default.

---

## What's next

Now I understand what the linker *receives*. The next question is what it *does* with it — specifically, which symbols from all those `.a` files actually make it into the final binary, and which get dropped.

That's Dead Code Elimination. And it's the heart of my research. More on that in Phase 4.

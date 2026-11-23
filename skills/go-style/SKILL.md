---
name: go-style
description: Write, review or refactor Go code that is easy to read and still fast. Keeps functions small with low cyclomatic and cognitive complexity, puts performance into API design and data layout rather than clever code, and proves changes with benchmarks, allocation counts and escape analysis. Use this skill whenever you write new Go code, design a Go API (especially readers, decoders, parsers, iterators or anything on a hot path), refactor or simplify existing Go, review a Go diff, or are told Go code is too complex, hard to maintain, slow or allocating too much, even if the request does not mention style. DO NOT load it for other languages, or for Go questions with no code to write or review.
---

# Go style: readable and fast

The aim is code a maintainer can read top to bottom, with performance that comes
from the shape of the API and the data rather than from tricks spread through the
code. Readability and speed rarely conflict once the design is right; when they
do, give up readability only on a measured hot path, and say so in a comment.

This follows Dave Cheney's "API Design as a Performance Primitive": beyond a
point, performance cannot be added after the fact, so design for it up front,
then keep everything else plain.

Sources: Cheney's talk (https://dave.cheney.net/paste/gophercon-uk-2026.html),
with code in github.com/pkg/json and github.com/davecheney/bytereader; John
Ousterhout, "Define errors out of existence".

## Order of priorities

1. **Correctness.** Never trade it for speed or brevity.
2. **API shape and big-O.** Who owns memory, what is returned, how much work each
   call does. These decide performance more than any later tuning.
3. **Allocations on the hot path.** Each allocation costs twice: once to allocate,
   once for the garbage collector. The collector may be fast, "but not allocating
   will always be faster."
4. **Work per unit of input.** Touch each byte once and make calls per token or
   record, not per byte.
5. **Micro-optimisations** (manual inlining, bit tricks, avoiding bounds checks),
   only on a path a profile shows is hot, and only with a benchmark proving the gain.

If a change is not justified at the level above, do not make it.

## Separate hot and cold paths

Decide which code runs per event or per request (hot) and which runs at setup,
build, configuration or update time (cold).

- **Cold path:** write the plainest code. Clone instead of sharing, use maps and
  `fmt`, allocate freely. Do not carry hot-path idioms here; they cost readability
  and buy nothing.
- **Hot path:** may use reusable buffers, generation counters, flat arrays and
  similar techniques. Give each technique a name (a small type or helper) and a
  comment explaining why it exists, so a reader is not left to reverse-engineer it.

## Design the API for performance

The API decides how much a caller can ever save. "You're benchmarking the garbage
collector" when the API forces every result onto the heap. Shape it so the fast
path is the natural one; a 2-10x gain is often available "if you're prepared to
accept a different API".

- **Do not return values that must escape.** Returning a primitive through an
  `interface{}`/`any`, a fresh string, a map or a pointer to a new struct per item
  forces an allocation per call (`encoding/json.Decoder.Token` is the classic
  "garbage factory"). Return a concrete type, or a view such as `[]byte`, and let
  the caller convert only what it needs.
- **Return views, not copies,** when the caller only needs to read. Document how
  long the view is valid ("valid until the next call", like `bufio.Scanner.Bytes`).
  A view of raw input (the token's bytes, whose first byte gives its kind) is
  cheaper than a decoded value the caller might ignore.
- **Let the caller own reusable memory.** Accept a scratch or buffer argument, or
  hold it in a struct the caller keeps, instead of allocating per call. Let callers
  supply the initial buffer, so it can live on the stack or come from a pool:
  `var buf [8 << 10]byte; d := NewDecoderBuffer(r, buf[:])`.
- **Hide buffer management behind a few methods.** `io.Reader` copies into a
  caller-managed buffer and leaves refilling to every caller. A small window API
  keeps data in place: `Window() []byte` to look, `Release(n)` to consume,
  `Extend() int` to read more, with "the window is invalidated by Release or
  Extend" in the doc comment.
- **Define errors out of the hot loop.** Use iteration that returns a bool or an
  empty result and reports the error afterwards: `for s.Next() { ... }` then
  `s.Err()`. Use sentinel results on the slow path (`Extend` returns 0, then the
  caller asks for the error) so the inner loop has no error branches. Treat
  `io.EOF` like any other terminal condition.
- **Count work per unit of input.** Process each byte once, make function calls
  per token or record rather than per byte, and remove intermediate copies.
  Moving from per-byte to per-token calls is often the second biggest win after
  removing allocations.
- **Follow the bytes end-to-end.** Don't decode, convert and re-encode data that
  arrives in the format it leaves in and isn't inspected in between; pass it
  through as `[]byte` / `json.RawMessage`. Don't optimise a converter the caller
  shouldn't be calling.
- **Match writer boundary types.** `io.Writer` takes `[]byte`. Passing `[]byte`
  through `fmt.Sprintf` into a `write(string)` helper forces `[]byte` → `string`
  → `[]byte` copies of the whole payload.
- **Layer the API.** Offer a low-level, allocation-free building block (raw
  tokens, views, scratch-owning iterators) and build the convenient API on top of
  it. Do not make the convenient, allocating API the only one.
- **Keep domain packages free of transport types.** Model packages should not
  import proto or HTTP packages; translate wire types at the transport edge.
- **Zero-value safety at the edge, positive semantics inside.** When adding an
  opt-out field to an existing struct whose zero value already means "all" (like
  `http.Transport.DisableCompression`), invert it once at registration so
  internal code works with positive flags instead of double negatives.
- **Write ownership and lifetime into doc comments:** what is borrowed, what is
  retained, what is safe for concurrent use, what the zero value means. Attach
  each guarantee to the exact symbol that enforces it (not to a type if only one
  field or consumer enforces it).
- **Make the zero value useful** where it is natural.
- **Prefer immutable shared values** and publish new snapshots, over locks around
  shared mutable state, when readers dominate.

Measure the API, not just the implementation: compare allocs/op and throughput
against the API you are replacing. [references/examples.md](references/examples.md)
shows these shapes side by side.

### Tuning a hot loop, last

Only after the API and big-O are right, and a profile shows the loop is hot:

- A `switch` over a state compiles to a chain of comparisons. Small methods per
  state, chosen through a `func` field (`d.state = (*Decoder).stateValue`), keep
  each state readable and can remove that dispatch.
- Move a check into the function that uses the value so the compiler can prove
  bounds checks unnecessary, and keep the entry point under the inlining budget
  (80; a call costs about 52). Check with `go build -gcflags=-m=2`.
- Merging two functions to avoid a call can win 5-15% on a per-token path, at the
  cost of deeper nesting. Do it only with a benchmark, and say why in a comment.

As Cheney puts it: do not reach for these tricks "without addressing the big O
effects in your api. It won't move the needle."

## Keep functions small and flat

Aim for each function to do one thing, with at most one or two branches. As a
guide, keep cyclomatic complexity at or below 10 and cognitive complexity at or
below 10 (`gocyclo`, `gocognit`). A flat type switch that hands each case to one
call is fine above that; nesting is not.

- **Name multi-term conditions.** Replace
  `if len(a)*len(b) > maxN || size(a)*len(b)+size(b)*len(a) > maxM` with
  `if !fitsProduct(a, b)`. The name documents the intent and the function can
  carry the explanation.
- **Dispatch, then delegate.** A switch over node or message kinds should call one
  method per case, not contain each case's logic.
- **Return early.** Guard clauses first, the main path unindented.
- **Do not repeat a case.** If two branches differ only in operand order, write
  one helper and call it twice.
- **Split mixed concerns.** "Find it", "build the new value" and "tidy up" are
  three functions, not one.
- **Avoid boolean flag parameters.** If one is the clearest option, give it a
  meaningful name and explain it at the call site.
- **Use lookup tables** (`map[string]Kind`, `[256]bool`) instead of chains of
  string or byte comparisons.

## Shape data for the reader and the CPU

- **Group parallel slices into a slice of structs.** Four slices indexed by the
  same ID are one struct. It reads better and usually improves locality.
- **Give a repeated mechanism its own small type.** If the same counter-wrap,
  bitmap or buffer logic appears twice, it is a type with two or three methods.
- **Prefer explicit fields to bit-packing** unless the packing is measured to
  matter. `[]segment{name, each}` beats a `uint64` of flags with shift arithmetic.
- **Keep representations few.** Fewer states or kinds mean fewer duplicated
  switches. Merge a special case into the general one when it costs nothing measurable.

## Prove performance claims

Never claim a change is faster, slower or allocation-free without measuring it.
Before changing performance-sensitive code:

1. Save a `benchstat` baseline of the relevant benchmarks.
2. Make the change, then compare. Report significant differences with numbers.
3. Diff `go build -gcflags=-m` output against the base for new heap escapes.
4. Keep or add `testing.AllocsPerRun` tests for paths that promise zero allocations.
5. Profile before guessing at the cause of a regression.

If simplifying costs measurable hot-path time and you cannot recover it cleanly,
stop and report the trade-off rather than quietly adding complexity back or
quietly accepting the regression.

Read [references/verify.md](references/verify.md) for the exact commands,
including how to measure complexity and the pitfalls that commonly cause
regressions. Read [references/examples.md](references/examples.md) for
before-and-after refactors that show the style.

## Follow Go conventions

- `gofmt`, `go vet` and the project's linter must pass. Fix code, do not weaken
  or exclude linter rules; suppress a single line only with a named rule and reason.
- Short receiver names, consistent across a type's methods; do not mix pointer and
  value receivers on one type without reason.
- Doc comments start with the identifier's name and state the contract.
- Keep sibling methods symmetric: if `Add` guards before cloning a snapshot,
  `Remove` should too; don't mutate a value copy outside the branch that stores
  it back.
- Comment the why, not the what. Match the surrounding comment density.
- Standard library first (`slices`, `maps`, `strings`, `math/bits`) before
  hand-written helpers.
- Never hand-edit generated code.

## When reviewing or refactoring

1. Identify the hot path and its benchmarks before touching anything. Check that
   the benchmark fixtures actually populate the default production fields.
2. Separate correctness, API and clarity findings from performance suggestions.
   Never suggest micro-optimisations (stack-array buffers, struct field packing,
   shared sentinel slices, scratch reuse) without a profile showing the item is a
   real share of the path's budget—saving 1-3 allocs on a 100+ alloc/op path is
   over-optimisation.
3. Measure complexity to find the worst functions; fix those first.
4. Audit tests for false confidence and redundancy: verify each assertion
   actually triggers the branch it claims to prove (e.g. overlapping inputs when
   testing cache reuse), and drop unit tests of trivial unexported helpers
   already covered through the real entry point.
5. Refactor in small steps, running focused tests after each.
6. Keep behaviour identical: same results, same budgets and fallbacks, same
   allocation guarantees. Change behaviour only as a separate, stated decision.
7. Report what changed, the before/after complexity and benchmark numbers, and
   anything not verified.

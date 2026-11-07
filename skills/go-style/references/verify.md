# Verifying readability and performance

Commands and pitfalls for proving a Go change keeps its performance. Replace
`./pkg/foo` with the package you are changing. Run benchmarks on an otherwise
idle machine, and keep the baseline and comparison runs on the same machine.

## Benchmarks with benchstat

```shell
go install golang.org/x/perf/cmd/benchstat@latest   # once, if missing

# Baseline, before changing code (or from a git worktree of the base branch).
go test -run '^$' -bench . -benchmem -count=6 ./pkg/foo > /tmp/before.txt

# After the change.
go test -run '^$' -bench . -benchmem -count=6 ./pkg/foo > /tmp/after.txt

benchstat /tmp/before.txt /tmp/after.txt
```

- Use `-count=6` or more; benchstat needs several samples to report significance.
  A `~` means no significant difference.
- Shorten long suites with `-benchtime=200ms` rather than fewer counts.
- `-bench` patterns are split on `/` per sub-benchmark level, so an alternation
  such as `A/(x|y)/n=1` may match nothing. Run separate patterns instead.
- Report time, B/op and allocs/op. Allocations often matter more than ns/op.

To get a baseline after you have already edited code, build one from the base
commit without touching your working tree:

```shell
git worktree add /tmp/base HEAD
(cd /tmp/base && go test -run '^$' -bench . -benchmem -count=6 ./pkg/foo) > /tmp/before.txt
git worktree remove --force /tmp/base
```

Do not use `git stash` for this when there are new untracked files: they stay in
the tree and the base build fails or measures the wrong code.

## Escape analysis

```shell
go build -gcflags=-m ./pkg/foo 2>&1 | grep -E 'moved to heap|escapes to heap|func literal escapes'
```

Compare the output for the base and your change. A new `moved to heap` or
`func literal escapes` on a hot path is a new allocation per call. Lines such as
`append escapes to heap` for reusable buffers are expected: they only allocate
when the buffer grows.

Check inlining of small hot helpers:

```shell
go build -gcflags=-m ./pkg/foo 2>&1 | grep -E 'can inline|inlining call to'
go build -gcflags=-m=2 ./pkg/foo 2>&1 | grep 'cannot inline'   # shows the cost and budget
```

## Profiling

```shell
go test -run '^$' -bench 'BenchmarkHot$' -benchtime=3s \
  -cpuprofile /tmp/cpu.out -o /tmp/pkg.test ./pkg/foo
go tool pprof -top -nodecount=25 /tmp/pkg.test /tmp/cpu.out
go tool pprof -list 'pkg\.(\*T)\.method$' /tmp/pkg.test /tmp/cpu.out   # per line
```

When a refactor regresses, profile both versions with the same benchmark and
compare where time moved. Test one hypothesis at a time: change one thing,
measure, keep it or revert it.

## Allocation tests

Lock in zero-allocation promises with a test, so a later refactor cannot silently
break them:

```go
func TestLookupDoesNotAllocate(t *testing.T) {
	var s Scratch
	ix := buildIndex(t)
	ix.Lookup(&s, input) // Warm the scratch buffers.
	if n := testing.AllocsPerRun(100, func() { ix.Lookup(&s, input) }); n != 0 {
		t.Fatalf("Lookup allocated %v times per call", n)
	}
}
```

## Measuring complexity

`gocyclo` and `gocognit` ship inside golangci-lint. Measure with a throwaway
config so the project's config is not changed. Run each linter separately:
golangci-lint keeps one issue per line by default, so two linters reporting the
same function hide each other.

For golangci-lint v1:

```yaml
# /tmp/cyclo.yml
linters:
  disable-all: true
  enable: [gocyclo]          # or [gocognit] in a second file
linters-settings:
  gocyclo:
    min-complexity: 1        # report every function
  gocognit:
    min-complexity: 1
issues:
  max-issues-per-linter: 0
  max-same-issues: 0
```

For golangci-lint v2, use `version: "2"`, `linters: {default: none, enable: [...],
settings: {...}}`. A v2-format file passed to v1 is silently ignored and reports
nothing, so check `golangci-lint version` first, or lower the threshold to 1 and
confirm functions are listed.

```shell
golangci-lint run -c /tmp/cyclo.yml --tests=false ./pkg/foo/... 2>&1 \
  | grep -oE 'complexity [0-9]+ of func `[^`]+`' | awk '{print $2, $5}' | sort -rn | head
```

If the project pins golangci-lint as a Go tool, use `go tool golangci-lint`.
Run the same command against a worktree of the base to report before and after.

## Pitfalls seen in practice

- **Range-over-func iterators** (`iter.Seq`) can escape to the heap when the
  iterator is not inlined, adding an allocation per call. On hot or rebuild-heavy
  paths, use a plain loop or a small method, and check with `-gcflags=-m`.
- **Closures and method values** passed to `slices.ContainsFunc` and similar can
  allocate on a hot path. A plain loop is as readable and cannot escape.
- **Splitting a hot accessor** into "fast check, else slow load" keeps the fast
  part small enough to inline. Merging everything into one function can push it
  over the inlining budget.
- **Extra call levels** in a per-attribute or per-element read add a few
  nanoseconds each. On a tens-of-nanoseconds path that is a 10% regression.
- **`m[string(b)]`** with a `[]byte` key does not allocate for a lookup. Only
  inserting a new key allocates. Reuse the byte buffer for building keys.
- **`clear` on a slice of pointers** calls into the runtime; it is not free for a
  tiny slice, but a hand-written loop was not faster either. Measure before
  replacing idiomatic code.
- **Removing a deduplication mark** that only applied in a rare case, and applying
  it always, costs a store and load per check. Keep rare-case work in the rare case.
- **Cloning on the cold path** is fine; cloning per hot call is not.

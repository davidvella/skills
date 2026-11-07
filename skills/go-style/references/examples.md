# Before-and-after examples

Each example shows a common shape of hard-to-read Go and the refactor that keeps
behaviour and performance. Names are illustrative.

## Contents

1. Name a multi-term condition
2. Dispatch, then delegate
3. Split find, build and tidy
4. Turn parallel slices into a struct
5. Give a repeated mechanism its own type
6. Let the caller own memory
7. Keep the hot accessor inlinable
8. Plain loops over closures on hot paths
9. Return a view, not a boxed value
10. Keep the buffer behind a window, not io.Reader
11. Define errors out of the inner loop
12. Layer a convenient API over a fast one
13. One small method per state

## 1. Name a multi-term condition

Before: the reader has to decode the arithmetic to learn what is being checked.

```go
func and(a, b Expr) Expr {
	if a.Empty() {
		return b
	}
	if b.Empty() {
		return a
	}
	if len(a.terms)*len(b.terms) > maxTerms || a.size()*len(b.terms)+b.size()*len(a.terms) > maxSize {
		if b.size() < a.size() {
			return b
		}
		return a
	}
	// ... expand ...
}
```

After: each branch is one word, and the helper carries the explanation.

```go
func and(a, b Expr) Expr {
	switch {
	case a.Empty():
		return b
	case b.Empty():
		return a
	case !fitsProduct(a, b):
		return smaller(a, b)
	}
	// ... expand ...
}

// fitsProduct reports whether expanding a and b stays within budget. Each term of
// a is combined with each term of b, so every condition of a is copied len(b.terms) times.
func fitsProduct(a, b Expr) bool {
	terms := len(a.terms) * len(b.terms)
	size := a.size()*len(b.terms) + b.size()*len(a.terms)
	return terms <= maxTerms && size <= maxSize
}
```

## 2. Dispatch, then delegate

Before: one function holds every case, nested two or three levels deep, with the
same logic repeated for each operand order. Cyclomatic complexity near 20.

```go
func (e *extractor) constraint(expr ast.Expr) Constraint {
	switch n := expr.(type) {
	case *ast.Binary:
		switch n.Op {
		case ast.Eq:
			if path, ok := e.path(n.LHS); ok {
				if v, ok := literal(n.RHS); ok {
					return equal(path, v)
				}
			}
			if path, ok := e.path(n.RHS); ok {
				if v, ok := literal(n.LHS); ok {
					return equal(path, v)
				}
			}
		case ast.In:
			// ... three more nested branches ...
		}
	}
	return Constraint{}
}
```

After: the switch only routes. Each case is a method with one job, and the
repeated operand-order logic is one helper called twice.

```go
func (e *extractor) constraint(expr ast.Expr) Constraint {
	switch n := expr.(type) {
	case *ast.Group:
		return e.constraint(n.Expr)
	case *ast.Binary:
		return e.binary(n)
	}
	return Constraint{}
}

func (e *extractor) binary(n *ast.Binary) Constraint {
	switch n.Op {
	case ast.Eq:
		if c, ok := e.equals(n.LHS, n.RHS); ok {
			return c
		}
		c, _ := e.equals(n.RHS, n.LHS)
		return c
	case ast.In:
		return e.in(n.LHS, n.RHS)
	}
	return Constraint{}
}

// equals extracts path == "value".
func (e *extractor) equals(lhs, rhs ast.Expr) (Constraint, bool) {
	path, ok := e.path(lhs)
	if !ok {
		return Constraint{}, false
	}
	v, ok := literal(rhs)
	if !ok {
		return Constraint{}, false
	}
	return equal(path, v), true
}
```

## 3. Split find, build and tidy

Before: removal from a copy-on-write store validates, copies, mutates and trims
in one function.

```go
func (s *Store) Without(id ID) *Store {
	b, slot := split(id)
	if id < 0 || b >= len(s.blocks) {
		return s
	}
	old := s.blocks[b]
	if old == nil || old.live&(1<<slot) == 0 {
		return s
	}
	block := *old
	block.live &^= 1 << slot
	next := &Store{blocks: slices.Clone(s.blocks), n: s.n - 1}
	if block.live != 0 {
		block.rebuild()
		next.blocks[b] = &block
		return next
	}
	next.blocks[b] = nil
	for len(next.blocks) > 0 && next.blocks[len(next.blocks)-1] == nil {
		next.blocks = next.blocks[:len(next.blocks)-1]
	}
	return next
}
```

After: the top-level function reads as the steps, and each step is testable.

```go
func (s *Store) Without(id ID) *Store {
	block := s.find(id)
	if block == nil {
		return s
	}
	b, slot := split(id)
	next := &Store{blocks: slices.Clone(s.blocks), n: s.n - 1}
	next.blocks[b] = block.without(slot)
	next.trimEmpty()
	return next
}

// find returns the block holding a live id, or nil.
func (s *Store) find(id ID) *block { /* ... */ }

// without returns a rebuilt copy of b without slot, or nil if it is now empty.
func (b *block) without(slot int) *block { /* ... */ }

// trimEmpty drops trailing empty blocks. Live entries never move, so IDs stay stable.
func (s *Store) trimEmpty() { /* ... */ }
```

## 4. Turn parallel slices into a struct

Before: four slices indexed by the same ID. Every reader has to know they line up.

```go
type Index struct {
	exact      []map[string][]int32
	prefix     []map[string][]int32
	prefixLens [][]int
	all        [][]int32
}
```

After: one slice of one type. The type can own its insertion logic.

```go
type Index struct {
	postings []postings // By attribute ID.
}

type postings struct {
	exact      map[string][]int32
	prefix     map[string][]int32
	prefixLens []int   // Distinct prefix lengths, ascending.
	all        []int32 // Every entry, for values of unknown shape.
}

func (p *postings) add(c *cond, id int32) { /* ... */ }
```

## 5. Give a repeated mechanism its own type

Before: the same "generation counter with wraparound" appears twice, with
different field names.

```go
s.callStamp++
if s.callStamp == 0 {
	clear(s.groupMark)
	s.callStamp = 1
}
// ... elsewhere ...
s.versionStamp++
if s.versionStamp == 0 {
	clear(s.conjMark)
	s.versionStamp = 1
}
```

After: one small type, written and explained once. Its methods inline, so it
costs nothing.

```go
// marks is a set of small integers that empties in O(1) by advancing a
// generation: i is in the set when slot[i] == gen.
type marks struct {
	gen  uint32
	slot []uint32
}

// next empties the set. When the generation wraps, it clears every slot once.
func (m *marks) next() {
	m.gen++
	if m.gen == 0 {
		clear(m.slot)
		m.gen = 1
	}
}

// add adds i, reporting whether it was absent.
func (m *marks) add(i int) bool {
	if m.slot[i] == m.gen {
		return false
	}
	m.slot[i] = m.gen
	return true
}
```

## 6. Let the caller own memory

Before: every call allocates its result and its working state.

```go
func (ix *Index) Candidates(doc Doc) []ID {
	seen := map[ID]bool{}
	var out []ID
	// ...
	return out
}
```

After: the caller keeps a `Scratch` per goroutine. After the first call the
lookup allocates nothing. The doc comment states the lifetime.

```go
// Scratch is reusable working memory. The zero value is ready to use. Do not
// share one between goroutines.
type Scratch struct {
	out  []ID
	seen marks
}

// Candidates returns the IDs that may match doc. The returned slice belongs to s
// and is valid until its next call. After the first call, it does not allocate.
func (ix *Index) Candidates(s *Scratch, doc Doc) []ID { /* ... */ }
```

## 7. Keep the hot accessor inlinable

Before: the cached check and the slow load are one function, too large to inline,
so every cached hit pays for a call.

```go
func (v *reader) value(attr int) docValue {
	if v.cache[attr].state == unread {
		// ... 15 lines reading and recording the value ...
	}
	return v.cache[attr]
}
```

After: the fast path is tiny and inlines; the slow path is its own function.

```go
// value reads an attribute at most once. It is small enough to inline; load
// does the reading.
func (v *reader) value(attr int) docValue {
	if v.cache[attr].state != unread {
		return v.cache[attr]
	}
	return v.load(attr)
}
```

Confirm with `go build -gcflags=-m | grep 'can inline (\*reader).value'`.

## 8. Plain loops over closures on hot paths

Before: an elegant iterator that allocates a closure on every call because it is
not inlined.

```go
func (c *cond) keys() iter.Seq[key] {
	return func(yield func(key) bool) { /* ... */ }
}

for k := range c.keys() { // func literal escapes to heap
	uses[k]++
}
```

After: a plain loop. Just as readable, and nothing escapes.

```go
for _, v := range c.values {
	uses[c.valueKey(v)]++
}
if c.prefix != "" {
	uses[c.prefixKey()]++
}
```

Iterators are fine on cold paths. Check `-gcflags=-m` before using them where
call counts are high.

## 9. Return a view, not a boxed value

Before: each token comes back as an `any`. Strings, numbers and delimiters are
boxed, so every call allocates, whether or not the caller needs the value.

```go
// Token returns the next JSON token: Delim, bool, float64, Number, string or nil.
func (d *Decoder) Token() (any, error)
```

After: return the token's bytes. The first byte identifies the kind, and the
caller converts only the tokens it cares about. In the talk this cut tokenising
from hundreds of thousands of allocations per document to single digits.

```go
// NextToken returns the next token. The slice refers to the decoder's buffer and
// is valid until the next call. Its first byte gives the kind:
//
//	{ } [ ] , :   delimiters
//	"             string
//	t f n         true, false, null
//	- 0-9         number
func (d *Decoder) NextToken() ([]byte, error)
```

## 10. Keep the buffer behind a window, not io.Reader

Before: `io.Reader` copies into a buffer the caller allocates, and every caller
re-implements refilling, partial reads and carrying leftovers.

```go
buf := make([]byte, 4096)
for {
	n, err := r.Read(buf)
	// ... handle a token split across two reads ...
}
```

After: the reader owns the buffer and exposes three small operations. Data stays
where it was read; the caller looks at it, consumes some, and asks for more.

```go
// Window returns the unread bytes. The window is invalidated by Release or Extend.
func (br *ByteReader) Window() []byte

// Release discards the first n bytes of the window.
func (br *ByteReader) Release(n int)

// Extend reads more data into the window, keeping unreleased bytes. It returns the
// number of bytes added, or 0 at the end of input or on error; see Err.
func (br *ByteReader) Extend() int
```

The caller's loop is simple: "if there is data in the window, process it, extend
if you need to. If you can't extend, give up."

## 11. Define errors out of the inner loop

Before: every iteration checks an error, even though errors are rare.

```go
for {
	tok, err := s.Token()
	if err == io.EOF {
		break
	}
	if err != nil {
		return err
	}
	handle(tok)
}
```

After: iteration returns a bool or an empty result, and the error is fetched once,
after the loop. The rare failure moves to the slow path (`Extend` returning 0).

```go
for s.Next() {
	handle(s.Bytes())
}
if err := s.Err(); err != nil {
	return err
}
```

## 12. Layer a convenient API over a fast one

Offer both, built in this order: the low-level API that allocates nothing, then
the convenient API implemented on top of it. Callers who need speed are never
forced through the convenience layer.

```go
// Low level: a view per token, nothing allocated after warm-up.
func (d *Decoder) NextToken() ([]byte, error)

// Convenient: decodes one value. Built on NextToken; allocates only the result.
func (d *Decoder) Decode(v any) error
```

## 13. One small method per state

A parser's `switch d.state` over every state grows into one long function. Give
each state a small method that reads its own input and sets the next state.
Holding the state as a method value replaces the switch with one indirect call,
and lets each method check bounds where it uses the data.

```go
type Decoder struct {
	scanner *Scanner
	state   func(*Decoder) ([]byte, error)
}

// NextToken is small enough to inline into callers.
func (d *Decoder) NextToken() ([]byte, error) { return d.state(d) }

func (d *Decoder) stateObjectColon() ([]byte, error) {
	tok := d.scanner.Next()
	if len(tok) < 1 {
		return nil, io.ErrUnexpectedEOF
	}
	if tok[0] != ':' {
		return nil, fmt.Errorf("expected colon, got %q", tok)
	}
	d.state = (*Decoder).stateObjectValue
	return tok, nil
}
```

This is a hot-loop tuning step: do it after the API and allocations are right,
and keep it only if a benchmark shows the gain.

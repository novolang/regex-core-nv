# regex-core-nv

A regular-expression engine for novo-lang that answers **where** — match
positions and numbered and named capture groups — in time linear in the
subject.

**Status: NOT IMPLEMENTED — interface only.**  Every `pub fn` body is a
`todo()`, so the signatures, the effect rows and the tests are published
and nothing is implemented.  The first implementation is the `0.1.0`
published over this.

## What this is

A port of Rust's [`regex`](https://docs.rs/regex) crate, which is a port
of [RE2](https://github.com/google/re2): a pattern is parsed to a typed
tree, compiled to a program, and run by one of two engines with a linear
time bound that never backtracks.

| module | holds | rows |
| --- | --- | --- |
| `rxerr` | why a pattern did not compile, at which byte, and the limits | `[]` |
| `rxclass` | a class as sorted ranges, the flags, Unicode and ASCII modes | `[]` |
| `rxast` | the typed tree, published so a consumer can ask about a pattern | `[]` |
| `rxprog` | the compiler, and the Pike VM / lazy DFA pair as a named choice | `[]` |
| `rxmatch` | **`RxMatch`** — positions, captures, split, replace | `[]` |
| `rxprofile` | I-Regexp, ASCII-only and `std.regex`-compatible, as named subsets | `[]` |

## The load-bearing interface

```novo norun:pseudo
pub struct RxMatch
    start: Int
    end: Int

pub struct RxCaptures
    slots: [Int]
    group_count: Int
```

**A match is a POSITION, and that is the whole reason this row exists.**
`std.regex.find` answers `?Str` — the matched text, copied — and
jsonpath-nv's `jpregex` answers a `Bool`.  Between them a caller can
learn *that* something matched and *what* it was, and never *where*.
Six of jq's builtins need where: `match` reports the offset and length
of every match, `capture` names the groups, `scan` walks them, `splits`
cuts between them, and `sub`/`gsub` rebuild the subject around them.
None of those can be written over a copied substring, because a copy has
lost its position in the subject it came from.

So every answer here is offsets and the text is one slice away.
`RxCaptures` is flat — slot `2n` is group `n`'s start and `2n+1` its end
— because that is the shape the Pike VM carries per thread, and a group
that did not participate is `-1` rather than an empty span: `(a)|b`
against `"b"` leaves group 1 **absent**, which is a different fact from
group 1 matching the empty string, and jq's `capture` reports `null` for
one and `""` for the other.

## Two engines, one program, and the choice is published

| engine | inner loop | answers | when |
| --- | --- | --- | --- |
| `RxLiteralScan` | a substring search | a position | the pattern is a literal run |
| `RxLazyDfa` | a table lookup per byte | where a match ends | no captures were asked for |
| `RxPikeVm` | every alternative at once, with slots | positions **and** groups | captures were asked for |
| `RxNeverMatches` | nothing | `false` | the pattern can match nothing |

The matcher runs the DFA to find that there is a match and where it
ends, and the Pike VM over that span to find the groups; a caller that
asked only `is_match` never runs the VM.  `rxprog.engine_for` answers
which one a call **would** use, before the call — because "my regex got
slow" is unactionable and "asking for groups turned the DFA off" is a
sentence somebody can act on.

**Linear time is the guarantee, and it is why two things are missing.**
Matching is a Thompson NFA simulation with no backtracking, so `(a+)+b`
against twenty-five `a`s is immediate rather than exponential.
Back-references and lookaround are the two constructs that cannot be
matched under that bound, and `rxerr` refuses both **by name** —
`RxBackreference` and `RxLookaround`, with `is_by_design` saying which
refusals will never be lifted.  They are the price, not an oversight.

## The one example that will work

```novo
use rxmatch
use udata

fn main() [io]
    match rxmatch.compile(udata.data_compact(), "(?<year>[0-9]{4})-(?<month>[0-9]{2})")
        Err(e) => println(e.message())
        Ok(r)  =>
            match rxmatch.captures(r, "shipped 2026-09-12")
                None    => println("no match")
                Some(c) =>
                    match rxmatch.group_named(r, c, "year")
                        None    => println("no year")
                        Some(m) => println(rxmatch.text_of("shipped 2026-09-12", m))
```

## Adding it, and checking it

```console
$ novo pkg add regex-core-nv
$ novo pkg build
$ novo test tests/rxmatch_tests.nv
```

The suites are **red on purpose**: every body is a `todo()`, so every
assertion reaches `not implemented: regex-core-nv.<module>.<fn>`.  That
is what an interface release looks like from the outside, and it is how
the first implementation will know it is finished.

## The layer, and why

`core`, and every row is `[]`.  A regular-expression engine is
arithmetic over text the caller already holds: nothing is read, nothing
is written, and a compiled pattern is a value the caller threads —
which matters, because compiling is the expensive half and matching is
the cheap one.  A jq program compiles its six patterns once and matches
them over a million documents.

The one thing an engine of this shape usually reaches for and this one
does not is a **mutable cache**: the lazy DFA's states.  `RxLimits.
max_dfa_bytes` is a number the *caller* sets, and building the states
inside one call rather than across calls is the implementation lane's
problem rather than a `[mutate]` row on this package's public surface.
If it turns out it cannot be, the row widens and this section says so
rather than the claim quietly growing.

### No device claim, and why

There is no `tests/embedded_probe.nv`, deliberately.  Compiling a
pattern builds an instruction list and a class table, both of which
allocate and neither of which is sized by the caller in advance; and the
classes a device would most want — `\w`, `\d` in their Unicode readings
— are unicode-nv's data tables, which is the largest thing in this
package's closure.

What a device consumer of regular expressions actually wants is a
pattern **compiled on a host and shipped as a program**, matched on the
target by the Pike VM alone over a fixed thread array.  That is a real
row and nobody has asked for it; naming it here is the honest form of
the refusal, and `rxprog.disassemble` and the published `RxInsn` are
what it would be built on.

## What `std.regex` keeps, and how the three consumers switch

`std.regex` is prepended to every program, needs no dependency, and is
the right answer for a script, a tutorial example and a forty-line
`main`.  This package does not replace it.

- **The global, pattern-first surface stays.** `regex.is_match(pattern,
  subject)` with no compile step and no `UniData` argument is what a
  one-off call wants, and nothing here is shorter.
- **`glob_match` stays.** It is not a regular expression and has no
  counterpart here; glob-nv is its neighbour.
- **The total, no-`Result` shape stays.** `std.regex` treats an
  unparsable pattern as "no match" on purpose, because its surface is
  `Bool`- and `?Str`-shaped and has nowhere to put a reason.  This
  package makes the other choice — every compile is a `Result` — because
  its callers hold patterns that came from a jq program, a JSON Schema
  document or a search box, and a `find` that answered `None` both for
  "no match" and for "your bracket is unclosed" would make every one of
  them re-ask.

**The one difference that changes an answer rather than adding a
feature** is the Unicode mode.  `\w` is `[0-9A-Za-z_]` in `std.regex`
and any Unicode letter here, so a pattern validating an identifier
silently starts accepting one written in Greek when it moves.
`rxprofile.gained_from_std` reports it per pattern, and it is the first
entry that function returns.

The three consumers, and what each changes:

| consumer | today | with this package |
| --- | --- | --- |
| **jsonquery-nv** | six jq builtins absent by name (`match`, `capture`, `scan`, `splits`, `sub`, `gsub`, and `test/2`), published as data in `jqbuiltin.absent_named` | the six become present. `absent_named` keeps only the host family (`env`, `input`, the clock), and `absent_for_layer` — which today answers "no layer can close `gsub`" — becomes true for none of the regex family |
| **jsonpath-nv** | `jpregex` is its own I-Regexp parser answering a `Bool` | `jpregex` becomes `rxprofile.compile_in(d, RxIRegexp, …)` plus `rxmatch.is_match`. `is_iregexp`, `needs_categories` and `unsupported_construct` keep their names and their answers — `rxprofile.accepts`, `rxprofile.needs_categories` and `rxprofile.outside` are the same three questions |
| **matchers-nv** | `matchtext.matches_regex` is a `Matcher<Str>` over `std.regex`, with a comment saying it "costs no dependency and no catastrophic backtracking" | half of that sentence stays true and the other half improves: this package has no catastrophic backtracking either, and a matcher over it can report **where** a match was, which is what an assertion failure message wants |

`jqbuiltin.absence_reason` is the place jsonquery-nv says why each name
is missing; when the six land, those rows are the changelog entry.

## What widened, and what did not

- **Nothing widened.**  Every row in the package is `[]`.
- **`UniData` is a parameter on every function that could consult it**,
  rather than a module-level value.  It is the only shape that lets a
  caller in ASCII mode prove — with `rxprog.needs_tables`, before it
  matches anything — that it is not paying for the tables.  The cost is
  an extra argument on eleven functions, which is the right trade for
  the largest dependency in the closure being optional in practice.
- **A recursive enum needs its recursion behind a list.**  `RxNode`'s
  `RxGroup` and `RxRepeatNode` hold `inner: [RxNode]` which is exactly
  one element by construction, and `rxast.node_child` is the accessor
  that says so once instead of at every call site.  That is the
  language's shape rather than the design's, and it is written down
  here so a reader does not read it as a modelling decision.
- **The AST is published and that was not obvious.**  A smaller package
  would parse straight to instructions.  Three consumers need to *ask*
  something about a pattern without matching anything — is this
  I-Regexp, does this pattern have a `.*` that should be `[^/]*`, where
  does this group start — and none of those can be answered from a
  program.  Publishing the tree is what stops each of them writing a
  second parser.
- **A profile refuses and never changes meaning.**  A pattern both
  `RxFull` and `RxIRegexp` accept matches the same strings under both.
  A profile that also changed semantics would reintroduce exactly the
  interoperability failure RFC 9485 exists to prevent, in the package
  meant to fix it.

## The reference implementation

The [`regex`](https://docs.rs/regex) crate for the surface and the
engine split, and [RE2](https://github.com/google/re2) for the
semantics.  [RFC 9485](https://www.rfc-editor.org/rfc/rfc9485) is
I-Regexp, and
[Russ Cox's articles](https://swtch.com/~rsc/regexp/) are where the Pike
VM and the lazy DFA are described.  The test vectors are the `regex`
crate's own suite and RFC 9485's examples; the pathological
`(a+)+b` case is the one every backtracking engine fails.

## Licence

Apache-2.0.

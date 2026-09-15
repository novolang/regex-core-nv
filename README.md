# regex-core-nv

A regular expression is a pattern that describes a set of strings. This
package is a port of Rust's [`regex`](https://docs.rs/regex) crate,
which is itself a port of [RE2](https://github.com/google/re2): a
pattern is parsed into a tree, compiled into a program, and run by an
engine whose running time is linear in the length of the subject. It
also implements
[RFC 9485](https://www.rfc-editor.org/rfc/rfc9485), the I-Regexp
interoperable subset. It is built on
[unicode-nv](https://novo-lang.org/packages/unicode-nv), and
[jsonquery-nv](https://novo-lang.org/packages/jsonquery-nv) and
[jsonpath-nv](https://novo-lang.org/packages/jsonpath-nv) are the
packages waiting for it.

**Status: NOT IMPLEMENTED — interface only.** Every function is declared
with its full signature, but every body is a `todo()` that panics when
called. The package is published so its design can be reviewed and
depended on before it is implemented. Version 0.1.0 will be the first
working release.

## What it is

The **pattern** is the expression. The **subject** is the text it is
matched against. A **match** here is a pair of byte offsets into the
subject, not a copy of the matched text.

A **capture group** is a parenthesised part of the pattern whose own
match is reported separately. Groups are numbered from 1 in the order
their opening parenthesis appears, and a group may also be given a name
with `(?<name>…)`. A group that took no part in the match is **absent**,
which is a different fact from a group that matched the empty string.

A **character class** is a set of codepoints, written `[a-z]` or `\d`.
A **flag** changes how the pattern is read: case-insensitive,
multiline, dot-matches-newline and so on.

The pattern is compiled into a **program**, a list of instructions, and
run by one of four **engines**. A literal scan is a substring search. A
**lazy DFA** builds a state table as it goes and answers where a match
ends, but cannot report groups. A **Pike VM** simulates every
alternative of the pattern at once, carrying the group offsets with
each alternative, and is the engine that reports captures.

Linear time is the guarantee, and it is what makes `(a+)+b` against
twenty-five `a` characters immediate rather than exponential. Two
constructs cannot be matched under that bound, so this package refuses
both: a **back-reference**, written `\1`, which requires the engine to
remember what a group matched, and **lookaround**, written `(?=…)` or
`(?<=…)`, which requires it to try a second match at the same position.

A **profile** is a named subset of the syntax. A profile refuses
patterns; it never changes what an accepted pattern means.

## Install

```
novo pkg add regex-core-nv
```

## Example

```novo
use rxmatch
use udata

fn main() [io]
    let subject = "shipped 2026-09-12"

    // Compile the pattern once. The Unicode tables are an argument, so
    // a caller in ASCII mode can see that it is not paying for them.
    match rxmatch.compile(udata.data_compact(), "(?<year>[0-9]{4})-(?<month>[0-9]{2})")
        // A pattern that does not compile says what was wrong and at
        // which byte of the pattern.
        Err(e) => println(e.message())
        Ok(r)  =>
            // `captures` answers the offsets of the whole match and of
            // every group, or `None` when the pattern did not match.
            match rxmatch.captures(r, subject)
                None    => println("no match")
                Some(c) =>
                    // A named group is looked up by name. `None` means
                    // the group took no part in the match.
                    match rxmatch.group_named(r, c, "year")
                        None    => println("no year")
                        // The offsets are turned into text against the
                        // caller's own subject.
                        Some(m) => println(rxmatch.text_of(subject, m))
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: regex-core-nv.<module>.<fn>` panic. The tests are the
specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `rxerr` | Why a pattern did not compile, at which byte, and the two sets of resource limits. |
| `rxclass` | A character class as sorted codepoint ranges, the set operations over them, the flags, and the named classes. |
| `rxast` | The parsed pattern as a tree, published so a consumer can ask about a pattern without matching anything. |
| `rxprog` | The compiler from tree to program, and which engine a call would use. |
| `rxmatch` | Compiling, matching, captures, splitting and replacement. The module most callers use. |
| `rxprofile` | The four named subsets, what a pattern uses outside one, and what a pattern gains when it moves from `std.regex`. |

## How to choose an entry point

**`rxmatch.compile` is the ordinary way in.** It takes the Unicode
tables and a pattern and answers a `Result`. `rxmatch.regex` is the
same with the flags and the limits given explicitly.

**`rxmatch.is_match` never runs the Pike VM.** Asking only whether
something matched lets the lazy DFA answer alone.

**`rxmatch.find` answers one match; `find_all` answers every
non-overlapping match; `find_first_n` stops after a count.**

**`rxmatch.captures` answers the groups as well, and costs more.** Use
it only when a group is wanted.

**`rxmatch.split` answers the pieces between matches, as offsets.**
`split_n` stops after a count.

**`rxmatch.replace_all` builds a new string; `replace_into` appends to
a buffer you own.** Both take a template, and
`rxmatch.template_fault` checks a template before either is called.

**`rxprofile.compile_in` compiles under a named subset.** Use
`RxIRegexp` when two implementations have to agree, `RxAsciiOnly` when
the Unicode tables must not be linked, and `RxStdCompatible` when
checking a pattern that is moving off `std.regex`.

**`rxast.parse` gives you the tree without compiling anything.** Use it
to ask questions about a pattern: which constructs it uses, whether it
is anchored, whether it can match the empty string, where a group
starts.

**`rxprog.engine_for` says which engine a call would use, before the
call.** "My regular expression got slow" is unactionable. "Asking for
groups turned the lazy DFA off" is something a caller can act on.

## The rules a user needs

1. **A match is a position, not a string.** `RxMatch` is a start and an
   end offset into the subject you passed in.
   `rxmatch.text_of(subject, m)` is the one slice. This is what lets a
   caller report where something matched, which a copied substring
   cannot do.
2. **`RxCaptures` is flat, and slot `-1` means absent.** Slot `2n` is
   group `n`'s start and slot `2n+1` is its end. A group that took no
   part in the match carries `-1` in both, which is a different answer
   from a group that matched the empty string. Matching `(a)|b` against
   `b` leaves group 1 absent.
3. **Back-references and lookaround are refused by name, and always
   will be.** `RxBackreference` and `RxLookaround` are separate error
   variants, and `rxerr.is_by_design` reports which refusals will never
   be lifted. Neither construct can be matched in linear time.
4. **Every compile answers a `Result`.** A pattern may come from a
   query language, a schema document or a search box, so a failure has
   a reason and a byte offset. `rxerr.offset_of` is that offset, and it
   is 0 for a fault about the whole pattern.
5. **An unknown escape is refused.** RE2 refuses `\q` rather than
   reading it as the letter `q`, because the two readings differ
   silently and the wrong one is a pattern that matches nothing.
6. **The Unicode tables are a parameter on every function that could
   consult them.** A caller in ASCII mode can prove, with
   `rxprog.needs_tables` and before matching anything, that it is not
   paying for them. `rxclass.needs_tables` answers the same question
   about one class.
7. **Use `rxerr.untrusted_limits()` for a pattern that came from
   outside.** It allows 4 KiB of pattern, 4000 instructions, a counted
   repetition ceiling of 100 and no lazy-DFA cache at all.
   `default_limits()` is the `regex` crate's own numbers, for a pattern
   the program wrote itself.
8. **`RxLimits.max_dfa_bytes` of 0 turns the lazy DFA off.** What is
   left is the Pike VM, which is what a caller who wants bounded memory
   rather than speed asks for.
9. **A counted repetition is expanded at compile time.**
   `{1000}{1000}{1000}` is linear to match and quadratic to compile,
   which is why `max_repeat` and `max_program` exist and why
   `RxRepeatTooLarge` names the limit it exceeded.
10. **A profile refuses and never changes meaning.** A pattern that
    both `RxFull` and `RxIRegexp` accept matches the same strings under
    both. RFC 9485 exists to prevent implementations disagreeing, and a
    profile that also changed semantics would reintroduce exactly that.
11. **I-Regexp always matches the whole subject.** RFC 9485 section 3
    has no anchors, no back-references, no lookaround, no lazy
    quantifiers, no named groups and no flags.
12. **`\w`, `\d` and `\s` mean more here than in `std.regex`.** They
    are Unicode classes by default rather than ASCII ones, so a pattern
    validating an identifier starts accepting one written in Greek.
    `rxprofile.gained_from_std` reports the difference for a given
    pattern, and this is the first entry it returns.
13. **`RxNode`'s recursion sits behind a list.** `RxGroup` and
    `RxRepeatNode` hold `inner: [RxNode]` with exactly one element, and
    `rxast.node_child` is the accessor that says so. That is the
    language's shape rather than a modelling decision.
14. **Compiling is the expensive half and matching is the cheap one.**
    A compiled `RxRegex` is a value the caller keeps. A program with six
    patterns compiles them once and matches them over a million
    documents.

## What is not included

- **Back-references and lookaround.** See rule 3. They are the price of
  the linear time bound.
- **Reading or writing anything.** The pattern and the subject are both
  strings the caller already holds, and `replace_into` appends to the
  caller's buffer.
- **A mutable cache across calls.** The lazy DFA's state table is built
  inside one call, so nothing in this package's public surface
  declares a mutation effect. `RxLimits.max_dfa_bytes` is the caller's
  bound on it.
- **Running on a microcontroller.** The package makes no such claim and
  carries no device probe. Compiling a pattern builds an instruction
  list and a class table, neither sized by the caller in advance, and
  the Unicode classes a device would want are the largest data in this
  package's dependency closure. Firmware that wants regular expressions
  wants a pattern compiled on a host and shipped as a program, matched
  by the Pike VM over a fixed thread array. `rxprog.disassemble` and
  the published `RxInsn` are what that would be built on.
- **Glob patterns.** A glob is not a regular expression.
  [glob-nv](https://novo-lang.org/packages/glob-nv) is the package for
  those, and `std.regex.glob_match` is the standard library's.

## Related packages

- [unicode-nv](https://novo-lang.org/packages/unicode-nv) supplies the
  character database. The `UniData` handle is an argument rather than a
  global, so a caller can see and avoid the cost.
- [jsonquery-nv](https://novo-lang.org/packages/jsonquery-nv) is
  missing six jq builtins — `match`, `capture`, `scan`, `splits`, `sub`
  and `gsub` — because none of them can be written over a copied
  substring. They become available when this package lands.
- [jsonpath-nv](https://novo-lang.org/packages/jsonpath-nv) has its own
  I-Regexp parser answering a boolean. `rxprofile.compile_in` with
  `RxIRegexp` replaces it, and `accepts`, `needs_categories` and
  `outside` are the same three questions it already asks.
- [matchers-nv](https://novo-lang.org/packages/matchers-nv) has a
  matcher over `std.regex`. One over this package can report where a
  match was, which is what an assertion failure message wants.
- `std.regex` in the standard library is prepended to every program and
  needs no dependency. Its surface is pattern-first with no compile
  step, it treats an unparsable pattern as no match, and it answers the
  matched text rather than its position. It is the right answer for a
  script or a forty-line program, and this package does not replace it.

## Tests

The reference implementations are the
[`regex`](https://docs.rs/regex) crate for the surface and the engine
split, and [RE2](https://github.com/google/re2) for the semantics.
[RFC 9485](https://www.rfc-editor.org/rfc/rfc9485) defines I-Regexp.
[Russ Cox's articles](https://swtch.com/~rsc/regexp/) describe the Pike
VM and the lazy DFA.

The vectors are the `regex` crate's own suite and RFC 9485's examples.
The pathological `(a+)+b` case is the one every backtracking engine
fails.

```bash
novo test tests/rxsyntax_tests.nv    # 14 tests: parsing, classes and the refusals
novo test tests/rxmatch_tests.nv     # 14 tests: positions, captures, split, replace
novo test tests/rxprofile_tests.nv   #  7 tests: the four subsets and what they refuse
```

The suite asserts that a match is reported as offsets, that an absent
group is distinguishable from an empty one, that a back-reference and a
lookaround are each refused by their own error variant, that an unknown
escape is refused rather than read as a letter, that `(a+)+b` finishes,
that a pattern exceeding the untrusted limits is refused before it is
parsed, that an I-Regexp pattern means the same under `RxFull`, and
that `\w` under `RxAsciiOnly` needs no tables.

The tests compile today and fail at run, each on the
`not implemented: regex-core-nv.<module>.<fn>` panic that is its body.
That is the expected state of an interface release. They turn green one
at a time as bodies land.

## Implementation status

Nothing is implemented. The table lists the surface an implementation
has to fill.

| Item | Implemented |
| --- | --- |
| `rxerr.offset_of`, `.is_by_design`, `RxError.message` | no |
| `rxerr.default_limits`, `.untrusted_limits` | no |
| `rxclass.default_flags`, `.ascii_flags`, `.flag_set` | no |
| `rxclass.empty`, `.one`, `.range`, `.union`, `.intersect`, `.difference`, `.negate` | no |
| `rxclass.fold_case`, `.fold_case_ascii`, `.matches`, `.is_empty`, `.cardinality` | no |
| `rxclass.word`, `.digit`, `.space`, `.dot`, `.posix_class`, `.property`, `.property_names` | no |
| `rxclass.needs_tables` | no |
| `rxast.parse`, `.node_child`, `.span_of`, `.to_pattern`, `.escape` | no |
| `rxast.matches_empty`, `.is_anchored_start`, `.group_index`, `.constructs_used` | no |
| `rxprog.compile`, `.build`, `.insn_count`, `.disassemble` | no |
| `rxprog.engine_for`, `.engine_name`, `.literal_prefix`, `.is_anchored`, `.min_length` | no |
| `rxprog.group_count`, `.group_names`, `.needs_tables` | no |
| `rxmatch.regex`, `.compile`, `.pattern_of`, `.group_count` | no |
| `rxmatch.is_match`, `.is_full_match`, `.find`, `.find_at`, `.find_all`, `.find_first_n` | no |
| `rxmatch.captures`, `.captures_at`, `.captures_all`, `.group_span`, `.group_named` | no |
| `rxmatch.text_of`, `.split`, `.split_n` | no |
| `rxmatch.replace_into`, `.replace_all`, `.replace_first`, `.expand_into`, `.template_fault` | no |
| `rxprofile.profile_name`, `.profile_flags`, `.accepts`, `.compile_in` | no |
| `rxprofile.outside`, `.outside_all`, `.needs_tables_in`, `.needs_categories`, `.gained_from_std` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->

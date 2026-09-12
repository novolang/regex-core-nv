# Changelog

All notable changes to regex-core-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.0.1 — 2026-09-12

The **interface**: every signature and every effect row, and no bodies.
`stability = "draft"`, and the release is recorded `implemented = false`.

### Added

- `rxmatch` — `RxMatch` and `RxCaptures`, the load-bearing types: a
  match is a POSITION, captures are positions, and the text is one
  slice away.  `find`, `find_at`, `find_all`, `captures`,
  `captures_all`, `split`, `replace_all` and `replace_into`.
- `rxast` — the typed tree, published because three consumers need to
  ask about a pattern without matching anything.
- `rxclass` — a class as sorted disjoint ranges, the five flags, and
  `RxUnicodeMode` so one build answers both readings of `\w`.
- `rxprog` — the compiler, and the Pike VM / lazy DFA pair as a named
  choice a caller can read before it calls.
- `rxprofile` — I-Regexp (RFC 9485), ASCII-only and
  `std.regex`-compatible as named subsets, with the construct named on
  every refusal.
- `rxerr` — a fault per way a pattern can be wrong, each with the byte
  it is about, and `is_by_design` for the two refusals that are the
  linear-time guarantee.

### Notes

- **Six jq builtins become possible.**  jsonquery-nv names `match`,
  `capture`, `scan`, `splits`, `sub`, `gsub` and `test/2` as absent
  because I-Regexp has no capture semantics; positions and groups are
  what they needed.
- **jsonpath-nv's `jpregex` becomes a profile check.**  `RxIRegexp`
  keeps the same refusals under the same names, so nothing downstream
  renames.
- **No device claim**, and the README argues it: a pattern compiled on
  a host and shipped as a program is the row a device would want, and
  nobody has asked for it.
- **`UniData` is a parameter, not a global**, so an ASCII-mode caller
  can prove with `rxprog.needs_tables` that it is not paying for
  unicode-nv's tables.
- **Five quiet mistakes are tests**: an empty match advances by one or
  the loop never terminates; a group that did not participate is absent
  and not empty; alternation is leftmost-FIRST so the order is
  meaningful; `\b` is evaluated against the whole subject and not
  against a tail; and `\w` means two different things in the two
  Unicode modes, which is the one difference a migration from
  `std.regex` can lose data to.

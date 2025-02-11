regex-automata
==============
A low level regular expression library that uses deterministic finite automata.
It supports a rich syntax with Unicode support, has extensive options for
configuring the best space vs time trade off for your use case and provides
support for cheap deserialization of automata for use in `no_std` environments.

[![Build status](https://github.com/BurntSushi/regex-automata/workflows/ci/badge.svg)](https://github.com/BurntSushi/regex-automata/actions)
[![](https://meritbadge.herokuapp.com/regex-automata)](https://crates.io/crates/regex-automata)

Dual-licensed under MIT or the [UNLICENSE](https://unlicense.org/).


### Documentation

https://docs.rs/regex-automata


### Usage

Add this to your `Cargo.toml`:

```toml
[dependencies]
regex-automata = "0.1"
```

and this to your crate root (if you're using Rust 2015):

```rust
extern crate regex_automata;
```


### Example: basic regex searching

This example shows how to compile a regex using the default configuration
and then use it to find matches in a byte string:

```rust
use regex_automata::Regex;

let re = Regex::new(r"[0-9]{4}-[0-9]{2}-[0-9]{2}").unwrap();
let text = b"2018-12-24 2016-10-08";
let matches: Vec<(usize, usize)> = re.find_iter(text).collect();
assert_eq!(matches, vec![(0, 10), (11, 21)]);
```

For more examples and information about the various knobs that can be turned,
please see the [docs](https://docs.rs/regex-automata).


### Support for `no_std`

This crate comes with a `std` feature that is enabled by default. When the
`std` feature is enabled, the API of this crate will include the facilities
necessary for compiling, serializing, deserializing and searching with regular
expressions. When the `std` feature is disabled, the API of this crate will
shrink such that it only includes the facilities necessary for deserializing
and searching with regular expressions.

The intended workflow for `no_std` environments is thus as follows:

* Write a program with the `std` feature that compiles and serializes a
  regular expression. Serialization should only happen after first converting
  the DFAs to use a fixed size state identifier instead of the default `usize`.
  You may also need to serialize both little and big endian versions of each
  DFA. (So that's 4 DFAs in total for each regex.)
* In your `no_std` environment, follow the examples above for deserializing
  your previously serialized DFAs into regexes. You can then search with them
  as you would any regex.

Deserialization can happen anywhere. For example, with bytes embedded into a
binary or with a file memory mapped at runtime.

Note that the
[`ucd-generate`](https://github.com/BurntSushi/ucd-generate)
tool will do the first step for you with its `dfa` or `regex` sub-commands.


### Cargo features

* `std` - **Enabled** by default. This enables the ability to compile finite
  automata. This requires the `regex-syntax` dependency. Without this feature
  enabled, finite automata can only be used for searching (using the approach
  described above).
* `transducer` - **Disabled** by default. This provides implementations of the
  `Automaton` trait found in the `fst` crate. This permits using finite
  automata generated by this crate to search finite state transducers. This
  requires the `fst` dependency.


### Differences with the regex crate

The main goal of the [`regex`](https://docs.rs/regex) crate is to serve as a
general purpose regular expression engine. It aims to automatically balance low
compile times, fast search times and low memory usage, while also providing
a convenient API for users. In contrast, this crate provides a lower level
regular expression interface that is a bit less convenient while providing more
explicit control over memory usage and search times.

Here are some specific negative differences:

* **Compilation can take an exponential amount of time and space** in the size
  of the regex pattern. While most patterns do not exhibit worst case
  exponential time, such patterns do exist. For example, `[01]*1[01]{N}` will
  build a DFA with `2^(N+1)` states. For this reason, untrusted patterns should
  not be compiled with this library. (In the future, the API may expose an
  option to return an error if the DFA gets too big.)
* This crate does not support sub-match extraction, which can be achieved with
  the regex crate's "captures" API. This may be added in the future, but is
  unlikely.
* While the regex crate doesn't necessarily sport fast compilation times, the
  regexes in this crate are almost universally slow to compile, especially when
  they contain large Unicode character classes. For example, on my system,
  compiling `\w{3}` with byte classes enabled takes just over 1 second and
  almost 5MB of memory! (Compiling a sparse regex takes about the same time
  but only uses about 500KB of memory.) Conversly, compiling the same regex
  without Unicode support, e.g., `(?-u)\w{3}`, takes under 1 millisecond and
  less than 5KB of memory. For this reason, you should only use Unicode
  character classes if you absolutely need them!
* This crate does not support regex sets.
* This crate does not support zero-width assertions such as `^`, `$`, `\b` or
  `\B`.
* As a lower level crate, this library does not do literal optimizations. In
  exchange, you get predictable performance regardless of input. The
  philosophy here is that literal optimizations should be applied at a higher
  level, although there is no easy support for this in the ecosystem yet.
* There is no `&str` API like in the regex crate. In this crate, all APIs
  operate on `&[u8]`. By default, match indices are guaranteed to fall on
  UTF-8 boundaries, unless `RegexBuilder::allow_invalid_utf8` is enabled.

With some of the downsides out of the way, here are some positive differences:

* Both dense and sparse DFAs can be serialized to raw bytes, and then cheaply
  deserialized. Deserialization always takes constant time since searching can
  be performed directly on the raw serialized bytes of a DFA.
* This crate was specifically designed so that the searching phase of a DFA has
  minimal runtime requirements, and can therefore be used in `no_std`
  environments. While `no_std` environments cannot compile regexes, they can
  deserialize pre-compiled regexes.
* Since this crate builds DFAs ahead of time, it will generally out-perform
  the `regex` crate on equivalent tasks. The performance difference is likely
  not large. However, because of a complex set of optimizations in the regex
  crate (like literal optimizations), an accurate performance comparison may be
  difficult to do.
* Sparse DFAs provide a way to build a DFA ahead of time that sacrifices search
  performance a small amount, but uses much less storage space. Potentially
  even less than what the regex crate uses.
* This crate exposes DFAs directly, such as `DenseDFA` and `SparseDFA`,
  which enables one to do less work in some cases. For example, if you only
  need the end of a match and not the start of a match, then you can use a DFA
  directly without building a `Regex`, which always requires a second DFA to
  find the start of a match.
* Aside from choosing between dense and sparse DFAs, there are several options
  for configuring the space usage vs search time trade off. These include
  things like choosing a smaller state identifier representation, to
  premultiplying state identifiers and splitting a DFA's alphabet into
  equivalence classes. Finally, DFA minimization is also provided, but can
  increase compilation times dramatically.


### Future work

* Look into being smarter about generating NFA states for large Unicode
  character classes. These can create a lot of additional work for both the
  determinizer and the minimizer, and I suspect this is the key thing we'll
  want to improve if we want to make DFA compile times faster. I *believe*
  it's possible to potentially build minimal or nearly minimal NFAs for the
  special case of Unicode character classes by leveraging Daciuk's algorithms
  for building minimal automata in linear time for sets of strings. See
  https://blog.burntsushi.net/transducers/#construction for more details. The
  key adaptation I think we need to make is to modify the algorithm to operate
  on byte ranges instead of enumerating every codepoint in the set. Otherwise,
  it might not be worth doing.
* Add support for regex sets. It should be possible to do this by "simply"
  introducing more match states. I think we can also report the positions at
  each match, similar to how Aho-Corasick works. I think the long pole in the
  tent here is probably the API design work and arranging it so that we don't
  introduce extra overhead into the non-regex-set case without duplicating a
  lot of code. It seems doable.
* Stretch goal: support capturing groups by implementing "tagged" DFA
  (transducers). Laurikari's paper is the usual reference here, but Trofimovich
  has a much more thorough treatment here:
  https://re2c.org/2017_trofimovich_tagged_deterministic_finite_automata_with_lookahead.pdf
  I've only read the paper once. I suspect it will require at least a few more
  read throughs before I understand it.
  See also: https://re2c.org
* Possibly less ambitious goal: can we select a portion of Trofimovich's work
  to make small fixed length look-around work? It would be really nice to
  support ^, $ and \b, especially the Unicode variant of \b and CRLF aware $.
* Experiment with code generating Rust code. There is an early experiment in
  src/codegen.rs that is thoroughly bit-rotted. At the time, I was
  experimenting with whether or not codegen would significant decrease the size
  of a DFA, since if you squint hard enough, it's kind of like a sparse
  representation. However, it didn't shrink as much as I thought it would, so
  I gave up. The other problem is that Rust doesn't support gotos, so I don't
  even know whether the "match on each state" in a loop thing will be fast
  enough. Either way, it's probably a good option to have. For one thing, it
  would be endian independent where as the serialization format of the DFAs in
  this crate are endian dependent (so you need two versions of every DFA, but
  you only need to compile one of them for any given arch).
* Experiment with unrolling the match loops and fill out the benchmarks.
* Add some kind of streaming API. I believe users of the library can already
  implement something for this outside of the crate, but it would be good to
  provide an official API. The key thing here is figuring out the API. I
  suspect we might want to support several variants.
* Make a decision on whether or not there is room for literal optimizations
  in this crate. My original intent was to not let this crate sink down into
  that very very very deep rabbit hole. But instead, we might want to provide
  some way for literal optimizations to hook into the match routines. The right
  path forward here is to probably build something outside of the crate and
  then see about integrating it. After all, users can implement their own
  match routines just as efficiently as what the crate provides.
* A key downside of DFAs is that they can take up a lot of memory and can be
  quite costly to build. Their worst case compilation time is O(2^n), where
  n is the number of NFA states. A paper by Yang and Prasanna (2011) actually
  seems to provide a way to character state blow up such that it is detectable.
  If we could know whether a regex will exhibit state explosion or not, then
  we could make an intelligent decision about whether to ahead-of-time compile
  a DFA.
  See: https://www.researchgate.net/profile/Xu-Shutu/publication/229032602_Characterization_of_a_global_germplasm_collection_and_its_potential_utilization_for_analysis_of_complex_quantitative_traits_in_maize/links/02bfe50f914d04c837000000/Characterization-of-a-global-germplasm-collection-and-its-potential-utilization-for-analysis-of-complex-quantitative-traits-in-maize.pdf

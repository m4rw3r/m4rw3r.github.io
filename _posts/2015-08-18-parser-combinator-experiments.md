---
layout: post
title: Parser Combinator Experiments in Rust
permalink: parser-combinator-experiments-rust
---

For the last week I have been working a bit on parser-combinator experiments using the
programming-language Rust. I have tried stacking structs, manually threading state and
boxed closures, with the last two seeming to be the most promising.

I am writing this as I would like feedback on my approach, as well as to announce that I have
something in the works which looks pretty promising.

## The code

The code can be found in my [rust_parser_experiments][] repo, the ``master`` branch currently
containing the manual state-threading and the ``fourth`` branch containing the boxed closure
version.

* [Manually threading state][]
* [Boxed closures][]

## First, some numbers

I used the attoparsec's http-header [examples][] as a simple benchmark to
compare the two approaches, both with each other as well as the other provided examples to see
how well it holds up. I also included the parser combinator [Nom](https://github.com/Geal/nom)
in this comparison.

The data used is the file ``http-requests.txt`` file as well as two copies of this file which
contains the same data copied 100 times and 10,000 times, resulting two files, 2 MB and 204 MB,
in size.

The tests were run on a MacBook Pro (Retina, 15-inch, Late 2013) with a 2.3 GHz Intel Core i7 and
16 GB RAM. All optimizations were turned on (``-O2`` for Haskell, ``-O3`` for C and ``--release``
for Rust).

Parser         | Time, 21 kB | Time, 2 MB  | Time, 204 MB
---------------|------------:|------------:|------------:
C http-parser  | 0.003 s     |     0.009 s |  0.62 s
Attoparsec     | 0.004 s     |     0.021 s |  1.45 s
Parsec         | 0.009 s     |     0.490 s | 47.75 s
[Nom][]        | 0.003 s     |     0.018 s |  1.42 s
[Manual][]     | 0.003 s     |     0.015 s |  1.19 s
[Boxed][]      | 0.004 s     |     0.041 s |  3.75 s

Making some quick profiling using Instruments (bundled with XCode) it seems like the Boxed version
spends about a second in ``je_malloc`` to allocate all the boxed closures which are used. In
comparison the Manual version spends most of its time (~600 ms) in the ``message_header``
function.

## Usage

The basic monad laws apply to both of the approaches, the difference being that manual
state threading requires the first parameter to always be the monad-state.

Haskell                | Manual                             | Boxed
-----------------------|------------------------------------|-------------------------------
``m :: Parser a``      | ``m: Parser<A>``                   | ``m: Parser<A>``
``f :: a -> Parser b`` | ``f: Fn(Empty, A) -> Parser<B>``   | ``f: Fn(A) -> Parser<B>``
``g :: Parser b``      | ``g: Fn(Empty) -> Parser<B>``      | ``g: Fn() -> Parser<B>``
``m >>= f``            | ``bind(m, f)``                     | ``bind(m, f)``
``m >> g``             | <code>bind(m, &#124;m, _&#124; g(m))</code> | <code>bind(f, &#124;_&#124; g())</code>
``return a``           | ``ret(m, a)``                      | ``ret(a)``
``fail a``             | ``err(m, a)``                      | ``err(a)``

For the manual version above, ``Empty`` is an alias for ``Parser<()>`` to indicate state but no
wrapped value (all the parsers require an ``Empty`` instance to act upon, to prevent accidental
loss of data).

The usage itself does not differ much, all parsers just needs to accomodate for that ``Empty``
parameter.

Expanded version of ``do``-syntax:

### Manually threading state

```rust
fn request_line<'a>(p: Empty<'a, u8>) -> Parser<'a, u8, Request<'a>, Error<u8>> {
    bind(take_while1(p, is_token), |p, method|
         bind(take_while1(p, is_space), |p, _|
              bind(take_while1(p, is_not_space), |p, uri|
                   bind(take_while1(p, is_space), |p, _|
                        bind(http_version(p), |p, version|
                             ret(p, Request{method: method, uri: uri, version: version,}))))))
}
```

### Boxed

```rust
fn request_line<'a>() -> Parser<'a, 'a, u8, Request<'a>, Error<u8>> {
    bind(take_while1(is_token), move |method|
         bind(take_while1(is_space), move |_|
              bind(take_while1(is_not_space), move |uri|
                   bind(take_while1(is_space), move |_|
                        bind(http_version(), move |version|
                             ret(Request{method: method, uri: uri, version: version,}))))))
}
```

And here is the difference in the http-example once the ``mdo!`` macro is used:
[Diff](https://gist.github.com/m4rw3r/6d1ca498f8e4abc24dbd)

## Operation

The difference really becomes apparent once you look at how the parser is run. The manual
threading of state eagerly parses the data as it moves through the defined parser functions,
eg. in ``request_line`` above the resulting data contains the actual ``Request`` struct, fully
populated and all, as well as the remaining data to be parsed. This is pretty straightforward and
inlining makes the resulting code appear similar to how a hand-written parser would look.

The boxed-closure version on the other hand does nothing when a defined parser like ``request_line``
is run. Instead this allocates a bunch of structures on the heap (boxed ``FnOnce``s inside of each
other) which are then executed once you want to parse something. Due to requiremens from ``ret``
the boxed closures cannot be ``Fn`` or ``FnMut`` or it would otherwise enforce ``Clone`` on the
value passed to ``ret``.

The ``FnOnce`` requirement also poses a problem when combinators like ``many`` and ``manyTill``
are used, as the parser can only be used once. This means that ``many`` and the like require
a newly allocated parser for every iteration, taking a ``Fn() -> Parser<A>`` as a parameter, which
causes a lot of heap-allocation during parsing.

## My opinion

I am currently on the fence here, but leaning towards using the manual threading-of-state for the
time being since it optimizes much better and seems to correspond better to Rust's current
feature-set. Once [abstract return types](https://github.com/rust-lang/rfcs/issues/518) lands, and
it works with functions, it should hopefully improve the performance of the closure-returning
version and make that a clear winner.

**EDIT 2018-08-19:** Updated performance numbers for Nom. The ``nom::Stepper`` introduced lots of memory
operations and caused cycles to be spent on moving data which should not have needed to be moved.

This increased Nom's performance from 0.004 s and 6.902 s for the first two benchmarks, and the
last one couldn't complete it in one hour when ``nom::Stepper`` was used. The ``nom::MemProducer``
was also tried along with ``nom::Stepper`` but that did not make any noticeable difference.

[rust_parser_experiments]: https://github.com/m4rw3r/rust_parser_experiments
[Manually threading state]: https://github.com/m4rw3r/rust_parser_experiments/tree/f64d0cc317c5d850987b83f206191eeed1e9bb68
[Boxed closures]: https://github.com/m4rw3r/rust_parser_experiments/tree/b36a60a79cf38bb9e1c39a2d382b737b0f6aeb22
[examples]: https://github.com/bos/attoparsec/tree/master/examples
[Nom]: https://gist.github.com/m4rw3r/0dd154d232abd0f3d4cf
[Manual]: https://github.com/m4rw3r/rust_parser_experiments/blob/master/examples/http_parser.rs
[Boxed]: https://github.com/m4rw3r/rust_parser_experiments/blob/fourth/examples/http_parser.rs

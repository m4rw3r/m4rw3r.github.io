---
layout: post
title: "Parser Combinator Experiments in Rust - Part 3: Performance and impl Trait"
permalink: parser-combinator-experiments-part-3
---

# Performance update

[Geoffroy Couprie](https://github.com/Geal), the creator of [Nom](https://github.com/Geal/nom),
suggested to me earlier that I should replace the ``is_token`` implementation from the previous
performance tests since it is pretty slow. The previous version was a pretty simple one-liner with
some significant overhead since it was written to be easy to read and to be comparable to the
``notInClass`` function from the Haskell parsers in terms of readability:

```rust
fn is_token(c: u8) -> bool {
    c < 128 && c > 31 && b"()<>@,;:\\\"/[]?={} \t".iter()
                           .position(|&i| i == c).is_none()
}
```

The version above is performing a naive linear search, which is far from the most efficient method
of determining if a character is in the given set. Attoparsec actually creates a binary search
tree, which is much faster. Here is the optimized version Geoffroy suggested:

```rust
fn is_token(c: u8) -> bool {
    // roughly follows the order of ascii chars: "\"(),/:;<=>?@[\\]{} \t"
    c < 128 && c > 32 && c != b'\t' && c != b'"' && c != b'(' && c != b')' &&
        c != b',' && c != b'/' && !(c > 57 && c < 65) && !(c > 90 && c < 94) &&
        c != b'{' && c != b'}'
}
```

This is of course much harder to read but gives a decent performance boost. Since it affects all
Rust parser-combinators I have replaced their ``is_token`` implementations with the optimized
version from above:

Parser                   | Time, 21 kB | Time, 2 MB | Time, 204 MB
-------------------------|------------:|-----------:|------------:
[C http-parser]          | 0.003 s     |    0.009 s |  0.62 s
[Experiment]<sup>1</sup> | 0.003 s     |    0.009 s |  0.71 s
[Nom]                    | 0.003 s     |    0.013 s |  1.01 s
[Attoparsec]             | 0.004 s     |    0.021 s |  1.45 s
[Combine]<sup>2</sup>    | 0.004 s     |    0.067 s |  6.10 s
Parsec<sup>3</sup>       | 0.009 s     |    0.490 s | 47.75 s

1: [Library code from last post](https://github.com/m4rw3r/rust_parser_experiments/tree/fifth),
   with verbose errors enabled.

2: Combine is now stable. Sadly it does not include the [Ranged Stream](https://github.com/Marwes/combine/pull/42)
   pull-request which was used in the Combine-pre benchmark in the last post, this means that the
   parser tested here is not a zero-copy parser and should be compared with the Combine-beta from
   the last post.

3: Uses the exact same parser-code as the Attoparsec benchmark.

# Experimental ``impl Trait``

There is an [experimental branch maintained by eddyb](https://github.com/eddyb/rust/commits/calendar-driven-development)
which implements the now closed [RFC PR #105](https://github.com/rust-lang/rfcs/pull/105). This
implementation allows for unboxed abstract return types through the use of the ``impl`` keyword
in a function return type.

The interesting thing with this in the context of monadic parser-combinator is that it is suddenly
no longer required to box closures whenever any are returned. This avoids a lot of allocations
as well as enables for more optimizations at the same time it makes it somewhat easier to manage
the types.

## Monadic parser combinator

The basics of a monadic parser-combinator is to use the combinator functions to create new
functions which are to be executed later once a parsing context exists (ie. some input to parse).
This means that the monadic type ``Parser<T>`` in simplified terms is ``Fn(&[u8]) -> Option<(T, &[u8])>``,
where ``u8`` is the input type, and the combinators themselves as well as the functions which create
parsers in the ``Parser<T>`` monad are of the type ``Fn(...) -> (Fn(&[u8]) -> Option<(T, &[u8])>)``.

In the manual threading-of-state the input is present from the start and the functions can
immediately operate on the input: ``Fn(&[u8], ...) -> Option<(T, &[u8])>``. This is not as
composable as the purely monadic parser-combinator since the state always needs to be considered
during combination and not only when building the combinators and primitive parsers.

## The unboxed ``Parser<T>``

The boxed example from [the first article of this series]({% post_url 2015-08-18-parser-combinator-experiments %})
is using the (simplified) type ``Box<FnBox(&[u8]) -> Option<(T, &[u8])>>`` which is heap-allocated.
But with the experimental branch we can use the ``impl Trait`` notation to make this closure
stack-allocated: ``impl Fn(&[u8]) -> Option<(T, &[u8])>``, of course this is not something we want
to copy all over the place, so we implement a trait which is implemented for ``Fn(&[u8]) -> Option<(T, &[u8])>``:

```rust
pub trait Parser<'a, T> {
    fn parse(self, &'a [u8]) -> Option<(T, &'a [u8])>;
}

impl<'a, T> Parser<'a, T> for F
  where F: FnOnce(&'a [u8]) -> Option<(T, &'a [u8])> {
    fn parse(self, i: &'a [u8]) -> Option<(T, &'a [u8])> {
        self(i)
    }
}
```

This is a very simplified signature which does not support any error handling or incomplete input,
or different input types. The ``'a`` lifetime is exposed through the ``Parser<'a, T>`` type to
allow zero-copy parsers to return ``Parser<'a, &'a [u8]>``, typically this ``'a`` will be free
unless constructions like this are needed.

The above allows us to write:

```rust
struct MyData<'a> {
    name:      &'a [u8],
    last_name: &'a [u8],
}

fn my_parser<'a>() -> Parser<'a, MyStruct<'a>> {
    bind(take_till(|c| c == b' '), |name|
        bind(take_till(|c| == b'\n'), |last_name|
            MyData{
                name:      name,
                last_name: last_name,
            }))
}

fn main() {
    let buf = b"Martin Wernstål\n";

    let parser = my_parser();

    println!("{:?}", parser.parse(buf));
    // MyData{name: &b"Martin", last_name: &b"Wernstål"}
}
```

## Performance

After making some very suspect (and test breaking) changes to eddyb's code I finally got
[the code](https://github.com/m4rw3r/rust_parser_experiments/tree/ebedd36f2f7e19171c65e38fdee3822d5daa4090)
to compile. This also includes avoiding any kind of linking since metadata is broken for
``impl Trait`` at the moment of writing. ([Diff](https://gist.github.com/m4rw3r/9128819a56db444ba402):
Fixed a probable typo, added monomorphize at some spots and finally removed the check for
``has_escaping_regions`` specifically for ``impl Trait`` syntax.)

The same files were used as in the [last test]({% post_url 2015-08-18-parser-combinator-experiments %}):

Parser               | Time, 21 kB | Time, 2 MB | Time, 204 MB
---------------------|------------:|-----------:|------------:
[slow ``is_token``]  | 0.003 s     | 0.014 s    | 1.16 s
fast ``is_token``    | 0.003 s     | 0.010 s    | 0.80 s

This is actually a very competitive result, seeing as the hand-written C-parser clocks in at 0.6
seconds, and neither the parser combinator nor the specific modifications to ``rustc`` have been
optimized. The code is used as is and only ``--release`` was supplied to cargo when compiling
the binary. To put it in perspective; it is only 30% slower than the hand-written C-parser which uses
switch-case-goto and 15% slower than the manual-threading of state variant. But it is  25% faster
than Nom and 80% faster compared to Attoparsec.

These are really promising results and I am hoping that Rust can land this feature in the near
future, unboxed abstract returns and higher kinded types would be amazing tools to already have in
a great language.

[C http-parser]: https://github.com/bos/attoparsec/blob/4f137347be02106765f6897059b88219c79bb86c/examples/rfc2616.c
[Attoparsec]: https://github.com/bos/attoparsec/blob/4f137347be02106765f6897059b88219c79bb86c/examples/RFC2616.hs
[Experiment]: https://gist.github.com/m4rw3r/cda66a9308ecb91f7147
[Combine]: https://gist.github.com/m4rw3r/4e82c4ee10deb1e141fc
[Nom]: https://gist.github.com/m4rw3r/54f7d80a3a5232c85d79
[slow ``is_token``]: https://github.com/m4rw3r/rust_parser_experiments/blob/ebedd36f2f7e19171c65e38fdee3822d5daa4090/src/main.rs#L208

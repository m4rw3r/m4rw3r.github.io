---
layout: post
title: Chomp and impl Trait, revisited
permalink: chomp-impl-trait--revisited
---

Now that [conservative `impl Trait`](https://github.com/rust-lang/rust/pull/35091) has landed in nightly I decided to finally take the time to reimplement all parsers of [Chomp](http://github.com/m4rw3r/chomp) using `impl Trait` to create a proper parser-monad.

Before doing this I finished up most of the work required for version 0.3 of chomp; [abstracting over the input type](https://github.com/m4rw3r/chomp/pull/45) which will enable users to plug the appropriate input-type. Hopefully this will open up for more specialized input-types in the future, like rope-based structures which are filled and dropped as the parsing proceeds as well as [implementations wrapping iterators](https://github.com/m4rw3r/chomp/pull/49).

But before finishing up 0.3 I just had to give `impl Trait` a try, I do not want to wait for too long before I release the results. The branch containing the code can be found in the [Chomp repo](https://github.com/m4rw3r/chomp/tree/experiment/impl_trait).

I am not yet totally sure if this will actually be the 1.0 of Chomp, or if it will be a separate crate called `chomp2` (or something else) but I am leaning towards creating a new crate. `chomp` still works on stable while the `impl Trait` version does not (and will probably not do so in the forseeable future since it uses `conservative_impl_trait`, `fn_traits` and `unboxed_closures`).

## Chomp and monad-like parsers

The monad-like state of Chomp makes it possible to easily compose parser-actions by passing an input state followed by any parameters the parser requires and in return a `ParseResult` is obtained, containing the remainder of the input-state as well as any success or failure value.

Parsers themselves do not implement any specific trait, instead they all follow about the same function-signature:

~~~rust
Fn*<I: Input>(I, ...) -> ParseResult<I, T, E>
~~~

where `Fn*` is the appropriate function-trait (depending on internal implementation of the parser, `many` requires a `FnMut`parser for example to be able to repeat) and `T` and `E` are success and error respectively.

There are two issues with this way of structuring parsers; first all input-state needs to be threaded through the parsers, making usage slightly more complicated, and parsers are not unified under one type which can make it slightly harder to compose them.

## A proper parser monad

A proper monad on the other hand does no actual parsing in the parser-functions themselves (like `any`, `string`, `take_while`, `many` and even user-defined parsers). The `ParseResult` type no longer exists and instead all parser-functions produce types implementing the `Parser` trait which are later invoked with a parser input:

~~~rust
trait Parser<I: Input> {
    type Output;
    type Error;

    fn parse(self, I) -> (I, Result<Self::Output, Self::Error>);
}
~~~

This trait is analogous to the following closure-signature:

~~~rust
Fn(input) -> (input-remainder, Result<success, error>)
~~~

which will create a Parser monad when used with the appropriate `bind` and `return` implementations:

~~~rust
fn return(T)  -> FnOnce(I) -> (I, Result<T, E>);
fn bind(A, B) -> FnOnce(I) -> (I, Result<U, E>)
  where A: FnOnce(I) -> (I, Result<T, E>)
        B: FnOnce(T) -> FnOnce(I) -> (I, Result<U, E>);
~~~

Preferably `impl Trait` should be used as much as possible since it prevents heavyweight syntax, but trait methods cannot return anomymized types (yet). This means that the basic combinators (like `bind`, `then` and `map`) provided on the `Parser` trait will, just like in [`Future` in the `futures` crate](http://alexcrichton.com/futures-rs/futures/trait.Future.html), have concrete types for those combinators.

## Converting Chomp to a proper monad

The basic parsers and combinators were no issues to convert whatsoever, mainly the function signature had to change to return an `impl Parser` and the function body had to be wrapped in a closure.  Here is an example of the `or` combinator difference:

~~~rust
pub fn or<I: Input, F, G>(f: F, g: G) -> impl Parser<I, Output=F::Output, Error=F::Error>
  where F: Parser<I>,
        G: Parser<I, Output=F::Output, Error=F::Error> {
    move |i: I| {
        let m = i.mark();

        match f.parse(i) {
            (b, Ok(d))  => (b, Ok(d)),
            (b, Err(_)) => g.parse(b.restore(m)),
        }
    }
}
~~~

It is almost identical to the monad-like definition:

~~~rust
pub fn or<I: Input, T, E, F, G>(i: I, f: F, g: G) -> ParseResult<I, T, E>
  where F: FnOnce(I) -> ParseResult<I, T, E>,
        G: FnOnce(I) -> ParseResult<I, T, E> {
    let m = i.mark();

    match f(i).into_inner() {
        (b, Ok(d))  => b.ret(d),
        (b, Err(_)) => g(b.restore(m)),
    }
}
~~~

Same goes for the bounded combinators; `many`, `many1`, `skip_many` and `many_till`, with one difference; the parameter is not the parser itself but instead a parser-constructor (ie. an `FnMut() -> Parser`, to be able to reuse parsers without requiring them to be `Clone`). The parser constructor and inner iterator-type will be a part of the closure being returned, the iterator-instance will be constructed and used in `FromIterator` when the parser is run.

`sep_by` on the other hand was a bit problematic due to the lifetime of the supplied closures since they need to be tied to the return value for as long as it has not yet been used. But by adding a parser for `Option<Parser>` the parser could be unified as a single type and be used with the appropriate `many` combinator.

Sadly for the bounded combinators I had to do the same thing as with the `Parser` methods, implement concrete structures for each combinator as well as split them up into different traits due to different requirements of the generics. This also made for some annoyance to make a generic `sep_by` taking any type implementing `BoundedMany` since the closure used in `sep_by` had to be converted to a struct implementing `FnMut` so that the types could be fully described.

This made the `sep_by` function kinda ugly, but the alternative was to have 5 copies of `sep_by` for the different types of ranges:

~~~rust
pub fn sep_by<I, T, F, G, P, Q, R>(r: R, f: F, sep: G) -> R::ManyParser
  where I: Input,
        T: FromIterator<P::Output>,
        F: FnMut() -> P,
        G: FnMut() -> Q,
        P: Parser<I>,
        Q: Parser<I, Error=P::Error>,
        R: BoundedMany<I, SepByInnerParserCtor<I, F, G>, T, P::Error> {
    BoundedMany::many(r, SepByInnerParserCtor {
        item: false,
        f:    f,
        sep:  sep,
        _i:   PhantomData,
    })
}
~~~

## Performance

Now, for the interesting part. How well does the conservative `impl Trait` version of Chomp perform?

Using the benchmarks present in the [`benches` directory](https://github.com/m4rw3r/chomp/tree/a8fe651dbf19c9cf1a53bfd36c48f150f01859e2/benches) for both the [input trait branch](https://github.com/m4rw3r/chomp/tree/a8fe651dbf19c9cf1a53bfd36c48f150f01859e2) and the [`impl Trait` branch](https://github.com/m4rw3r/chomp/tree/68fa6941cf01b13280f1c692817516a022891c45/benches) as well as using the `http_parser.rs` example used in some of the [previous](http://m4rw3r.github.io/parser-combinators-road-chomp-0-1/) [posts](http://m4rw3r.github.io/parser-combinator-experiments-rust/) we obtain the following numbers:


Test                           | `impl Trait`  |              | PR: input trait | &nbsp;
-------------------------------|--------------:|:-------------|----------------:|:------------
count_vec_10k                  | 6,802 ns/iter | (+/- 2,478)  | 6,856 ns/iter   | (+/- 694)
count_vec_10k_maybe_incomplete | 6,094 ns/iter | (+/- 2,092)  | 5,887 ns/iter   | (+/- 1,828)
count_vec_1k                   |   725 ns/iter | (+/- 265)    |   724 ns/iter   | (+/- 106)
many1_vec_10k                  | 6,576 ns/iter | (+/- 1,741)  | 6,574 ns/iter   | (+/- 1,518)
many1_vec_10k_maybe_incomplete | 6,590 ns/iter | (+/- 2,868)  | 6,698 ns/iter   | (+/- 1,254)
many1_vec_1k                   |   958 ns/iter | (+/- 128)    |   961 ns/iter   | (+/- 164)
many_vec_10k                   | 6,560 ns/iter | (+/- 918)    | 6,609 ns/iter   | (+/- 1,104)
many_vec_10k_maybe_incomplete  | 6,641 ns/iter | (+/- 2,140)  | 6,657 ns/iter   | (+/- 1,750)
many_vec_1k                    |   948 ns/iter | (+/- 108)    |   957 ns/iter   | (+/- 36)
multiple_requests              |44,617 ns/iter | (+/- 1,953)  | 44,681 ns/iter  | (+/- 5,344)
single_request                 |   606 ns/iter | (+/- 96)     |    610 ns/iter  | (+/- 175)
single_request_large           |   980 ns/iter | (+/- 329)    |    979 ns/iter  | (+/- 52)
single_request_minimal         |   107 ns/iter | (+/- 9)      |    114 ns/iter  | (+/- 7)
http_parser.rs, 204 MB         | ~0.548 s      |              | ~0.559 s        |

All these numbers seem to be very promising, especially the "real-world" usage in the HTTP-parser reading from a file on disk.

Kudos to Eddyb and everyone involved who managed to make `impl Trait` happen!

---
layout: post
title: "Parser Combinators: The road to Chomp 0.1"
permalink: parser-combinators-road-chomp-0-1
---

A few months ago I wrote some [articles](http://m4rw3r.github.io/parser-combinator-experiments-rust) [about](http://m4rw3r.github.io/parser-combinator-experiments-errors) [parser-combinators](http://m4rw3r.github.io/parser-combinator-experiments-part-3). Now I finally caved in and decided to settle with "good enough".

I have now released the initial version of [Chomp](https://github.com/m4rw3r/chomp), a monadic-style parser-combinator compatible with stable Rust.

# Monadic-style?

I call it monadic-style since it is not strictly monadic. Instead of implementing a real monad, I decided to go with the idea behind the third version of my [parser combinator experiments](https://github.com/m4rw3r/rust_parser_experiments/tree/third) and explicitly thread the state through the parsers. To write a real monad which can carry state besides what is present in its type (eg. `Option`, `Result`) boxing the values and/or actions is almost mandatory.

In the case of a parser combinator like Parsec and Attoparsec and my own [boxed implementation](https://github.com/m4rw3r/rust_parser_experiments/blob/fourth/src/lib.rs) it requires stacking of closures which means that we either have to settle for severely limited expressiveness in our parsers or that we have to box every single parser (in some cases both apply, see `Fn` vs `FnOnce` vs `FnMut`).

I selected manual threading of state instead of any of the other versions since it produced a parser with good performance, simple parsers, short function declarations (compare to stacking structs) for the parsers themselves as well as being straightforward to use.

# Why decide to write Chomp?

The goals for Chomp are to make a parser combinator which…

* …does not rely on lots of macros.
* …does not allocate anything unless absolutely necessary.
* …allows for monadic-style computation and composition.
* …lends itself well to writing lots of small composable functions.
* …hides the parser state from the user unless asked for.

And then a few personal reasons:

* It is fun and a challenge.
* Exploring what rust is capable of.
* I needed something to replace the hand-written parser in one of my own projects.

# Features

I am going to describe two of the more important features of Chomp: The approximation of linear types used to make sure the parser state is threaded through all the parsers and the macro which makes monadic composition so much more pleasant to write and read.

## `Input` and `ParseResult` are opaque linear types

The idea of explicitly passing a state parameter through all functions is definitely not a new idea, but languages like [Clean](https://en.wikipedia.org/wiki/Clean_%28programming_language%29) have taken this a step further and use something called [linear types](https://en.wikipedia.org/wiki/Substructural_type_system#Linear_type_systems). A linear type is a type which cannot be cloned or destroyed and has to be used exactly once. In a lazy language this will serialize all the side-effects but in a language like Rust we can use it to make sure that we do not let the state get misused nor forgotten.

Rust does not have support for linear types but its ownership model allows us to get close. The first thing we do is to limit the creation and destruction of values of the type to a library function which will pass it on to a closure and then expect the linear type back. The second thing is to prevent clones and copies, this is the default in Rust so we just neglect to derive `Clone` or `Copy`. The third part we cannot strictly enforce, but we can get close by using the `must_use` annotation.

The parser type is actually a pair of linear types, `Input<'a, I>` and `ParseResult<'a, I, T, E>`. `Input` is a completely separate type to make sure that they are not confused and to prevent accidental data loss (eg. passing a `ParseResult` to a function which continues to parse without using the value), and to carry some extra state which `ParseResult` does not carry in all its variants.

Example of it in action with comments and full type annotations (skipping input and error type):

```rust
// `Stream::parse` accepts a closure taking an `Input` and expects a
// `ParseResult` as a return value.
let r: Result<MyData, _> = buffer.parse(|i: Input<_>| -> ParseResult<_, MyData, _> {
    // We call a parser, this consumes `i` and moves the state into `d`.
    let d: ParseResult<_, Value, _> = some_parser(i, "some param");

    // if we forget to use `d` here we get a warning, and probably also a
    // compile error since `buffer.parse` expects a `ParseResult` back with
    // the same lifetime as the initial `i`.
    d.bind(|i2: Input<_>, t: Value| -> ParseResult<_, MyData, _> {
        // bind extracts the value of the `ParseResult` and also provides the
        // state we need to continue parsing, same rules apply as above.
        // Here we pass on `i2` to a parser and then `map` over the result:
        decimal(i2).map(|n| MyData(t, n))
    })
    // And here we hand the `ParseResult` back to `parse` which destructures it
    // for the user and also manages any input state.
});
```

And to prevent the state from being observed in an uncontrolled manner there are no public properties or methods which allow the state to be seen. This will make it hard to read data out-of-band and disrupt the parsing, which makes the high-level parsers to be very composable.

Of course you cannot write parsers without being able to observe the data which you are parsing. For this Chomp has the [primitives](http://m4rw3r.github.io/chomp/chomp/primitives/index.html) module which contains traits to do just this. The `combinators` and `parsers` modules use this module to observe and modify the parser state in a controlled manner.

And it is a public module so writing these fundamental parsers is not exclusive to the Chomp crate.

## The `parse!` macro

*One macro to rule them all.*

This feature was a must, I started detailing the syntax I wanted for it months ago but just recently got to implementing the macro itself. Flexibility and simplicity was key for the syntax yet i needed to be powerful enough that people would not avoid using it whenever they needed to use some more powerful features.

Normally you would have to keep on chaining calls to `bind` and wrap your code in closure after closure like this (this is typical of monadic code):

```rust
take_while1(i, is_token).bind(|i, method|
    take_while1(i, is_space).then(|i|
        take_while1(i, is_not_space).bind(|i, uri|
            take_while1(is_space).then(|i|
                http_version(i).bind(|i, version|
                    i.ret(Request {
                        method:  method,
                        uri:     uri,
                        version: version,
                    }))))))
```

This is not easy to parse a human, it is way too noisy and the assignment to values through `bind` is not easy to spot. If you read it line by line it might be simple, but not as a whole.

Using the [parse!](https://github.com/m4rw3r/chomp/blob/6bb50e22513c6b670dd1c22ba144be2b6884c8ab/src/macros.rs#L76) macro we can instead write the above code like this:

```rust
parse!{i;
    let method  = take_while1(is_token);
                  take_while1(is_space);
    let uri     = take_while1(is_not_space);
                  take_while1(is_space);
    let version = http_version();

    ret Request {
        method:  method,
        uri:     uri,
        version: version,
    }
}
```

The result is the same, the performance is the same and the compile time is not affected much at all, but it is much easier to read and understand what is happening.

# Performance

The performance is pretty good, especially when using the specific buffering available in the [buffer](http://m4rw3r.github.io/chomp/chomp/buffer/index.html) module of Chomp.

The [bigbuckbunny.mp4](https://raw.githubusercontent.com/Geal/nom/95c228c75c2964b20f0e1e42ee11a3877c1725ef/assets/bigbuckbunny.mp4) and [small.mp4](https://raw.githubusercontent.com/Geal/nom/95c228c75c2964b20f0e1e42ee11a3877c1725ef/assets/small.mp4) are parsed using [nom_benchmarks](https://github.com/Geal/nom_benchmarks/tree/f12814b075145795fb2cc26c279ca5a1d7d17e34). The code for Nom, Attoparsec and Chomp can be found in the repository.

The HTTP file tests are the same as I used in my previous blog posts, and they invlove more than just parsing since they are reading from a file on disk and parsing it as they are reading.

Parser                | bigbuckbunny.mp4 bench | small.mp4 bench | HTTP file, 2 MB | HTTP file, 204 MB
----------------------|-----------------------:|----------------:|----------------:|-----------------:
[Chomp]<sup>1</sup>   | 307 ns                 | 355 ns          |  8 ms           | 486 ms
[Chomp]<sup>2</sup>   | 384 ns                 | 436 ns          |  9 ms           | 512 ms
[Nom]<sup>3</sup>     | 379 ns                 | 395 ns          |  9 ms           | 572 ms
[Joyent http-parser]  |         --             |      --         |  9 ms           | 626 ms
[Attoparsec]          | 829 ns                 | 833 ns          | 20 ms           | 1,382 ms
[Combine]<sup>4</sup> |         --             |      --         | 26 ms           | 2,151 ms

1: Chomp was compiled without `verbose_error` (`--no-default-features`).

2: Chomp was compiled with `verbose_error` feature, this is the default.

3: Nom seems to have issues supporting buffered reading, if an incomplete state is encountered inside of a `many0!` or `many1` it will cause the parser to error. By modifying the `many1` macro [accordingly](https://gist.github.com/m4rw3r/e7018d0b9689530c18f7) (NOTE: Breaks inference for some code) I could use the [Nom adapter](https://github.com/m4rw3r/chomp/issues/14) in Chomp to drive Nom using the buffers provided by Chomp.

4: Compiled with `range_stream` feature to enable zero-copy parsing. Combine does not have support for incomplete parsing so the whole file was loaded into memory before parsing started.

Note that the `verbose_error` version is much slower for the mp4-tests. It seems like the culprit is the `string` parser and its verbose error message, since it copies the string it was trying to match on error. This causes a lot of small allocations when trying to match the mp4 box tags trough an alteration. [An issue](https://github.com/m4rw3r/chomp/issues/15) has been created for it and the mp4-parser currently works around it by not using `string` in a hot-path.

I might want to overhaul the error handling later, but the core should still be fine since it does not actually care about the type of the error itself (it is local to the `parsers` module).

Parser     | Version
-----------|--------
Attoparsec | 0.13.0.1
Chomp      | 0.1.1
Combine    | 1.1.0
Nom        | 1.0.1

[Chomp]: https://github.com/m4rw3r/chomp/blob/6bb50e22513c6b670dd1c22ba144be2b6884c8ab/examples/http_parser.rs
[Nom]: https://gist.github.com/m4rw3r/df10573763ff838988a8
[Combine]: https://gist.github.com/m4rw3r/a1f8d17e120828963a19
[Joyent http-parser]: https://github.com/bos/attoparsec/blob/4f137347be02106765f6897059b88219c79bb86c/examples/rfc2616.c
[Attoparsec]: https://github.com/bos/attoparsec/blob/4f137347be02106765f6897059b88219c79bb86c/examples/RFC2616.hs

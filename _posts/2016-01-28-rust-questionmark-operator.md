---
layout: post
title: "Rust: The `?` operator"
permalink: rust-questionmark-operator
---

For people who are not familiar with Haskell or Scala, Rust's `Option` and `Result` types might
feel a bit cumbersome and verbose to work with. To make it easier and less verbose to use them
the RFC [PR #243: Trait-based exception handling](https://github.com/rust-lang/rfcs/pull/243) has
been proposed.

In this blog post I will go through some basics of the RFC and then compare with a hypothetical
`do`-notation.

The RFC proposes a `?` operator which is a compiler-assisted rewrite of expressions around `?`
characters.  It is a unary suffix operator which can be placed on an expression to unwrap the value
on the left hand side of `?` while propagating any error through an early return:

~~~rust
File::create("foo.txt")?.write_all(b"Hello world!")
~~~

Would be transformed to:

~~~rust
match File::create("foo.txt") {
    Ok(t)  => t.write_all(b"Hello world!"),
    Err(e) => return Err(e.into()),
}
~~~

On its own `?` is just syntactic sugar for the `try!` macro, making it easier to write code
chaining expressions which can fail:

~~~rust
try!(File::create("foo.txt")).write_all(b"Hello world!")
~~~

# `try` and `catch`

The RFC also details a `try`-`catch` expression which would "catch" any early returns performed by
the `?` operator. Essentially the early returns would jump to the `catch` block and the whole
`try`-`catch` expression would assume that value. If no `catch` block is provided the `try` block
will return a wrapped result:

~~~rust
try {
    let mut f = File::create("foo.txt")?;
    f.write_all(b"Hello world!")?
}
// can also be written as
try { File::create("foo.txt")?.write_all(b"Hello world")? }
~~~

Note that the `?` is required at the last line since we want a `Result<(), io::Error>`, not a
`Result<Result<(), io::Error>, io::Error>`. The `Result` type will automatically re-wrap the
return value of the block if there is no `catch` block, so that the whole expression assumes
a `Result<T, E>` without the need to wrap the return value yourself.

Adding the `catch` would be equivalent to using `Result::or_else` with `try` and `match`:

~~~rust
try {
    let mut f = File::create("foo.txt")?;
    f.write_all(b"Hello world!")?
}
catch {
    // we only have one type to match on
    e => {
        println!("{}", e)
    }
}
~~~

Is equivalent to:

~~~rust
try {
    let mut f = File::create("foo.txt")?;
    f.write_all(b"Hello world!")?
}.or_else(|e|
    println!("{}", e)
)
~~~

The difference here is that any `return` inside of `Result::or_else` cannot immediately result in
an early return.

The `?` also allows us to use it at an arbitrary nesting within the `try` block (and in code in
general):

~~~rust
fn logging_on()                  -> Result<bool, io::Error>     { ... }
fn read_values()                 -> Result<SomeData, io::Error> { ... }
fn log_values(values: &SomeData) -> Result<(), io::Error>       { ... }

try {
    let data = read_values()?;

    if logging_on()? {
        log_values(data)?;
    }

    data
}
~~~

# Do-notation

So called `do`-notation is a syntactic-sugar which allows us to write statements and expressions
dealing with the computation within a context. For example values of types like `Option` and
`Result` enable us to perform operations without having to worry about the failure-state of the
same inside of the `do`-expression:

~~~rust
do {
    mut f <- File::create("foo.txt");
    f.write_all(b"Hello world!")
}
~~~

The first line inside of the `do`-block is a so called monadic bind: it will bind the value
contained inside of the type on the right of `<-` to the identifier on the left for the rest of
the block. The result of the rest of the block will be merged with the context from the value on
the right of `<-`. In the case of `Result` and `Option` this is very simple and would just not
evaluate the rest of the block if the right hand side is of an error-variant.

The second line is an expression which is evaluated within the context of the first line: `f`
is available and can be mutated and the result is another `Result` which will be returned to the
monadic-bind method of the first `Result` value for merging (in `Result` this would be a no-op
since there is no state to merge).

The result of the `do`-block expression is `Result<(), io::Error>`, since that is the return value
of `write_all`. The two expressions are compatible since they both return a value of type
`Result<T, io::Error>`.

The above code desugars to:

~~~rust
File::create("foo.txt").and_then(|mut f|
    f.write_all(b"Hello world!"))
~~~

Each expression in the `do`-notation above evaluates to some `Result<T, io::Error>` (for any `T`) which
means that when adding expressions not resulting in the `Result<T, io::Error>` type they need to be
wrapped (this is called "lifting" in Haskell terminology) to produce a `Result` which then fits into the
`do`-block:

~~~rust
let h = "Hello".to_owned();

do {
    mut f <- File::create("foo.txt");
    s     <- Ok(h + " world!");

    f.write_all(s)
}
~~~

In the `try`-block we could just add it as usual:

~~~rust
let h = "Hello".to_owned();

try {
    let mut f = File::create("foo.txt")?;
    let s     = h + " world!";

    f.write_all(s)?
}
~~~

Though in the code examples above would be more suitable to just move the expression assigned to
`s` into the call to `write_all`. It would also be desirable to allow the use of normal
`let`-binds inside of `do`-blocks to allow the declarations of variables without having to use
monadic bind.

Another difference is that `do`-notation only works on the statement-level whereas `?` works at
any nesting inside of the `try`-block. A direct translation of the nesting-example of the
`try`-block would look like this:

~~~rust
fn logging_on()                  -> Result<bool, io::Error>     { ... }
fn read_values()                 -> Result<SomeData, io::Error> { ... }
fn log_values(values: &SomeData) -> Result<(), io::Error>       { ... }

do {
    data <- read_values();
    log  <- logging_on();

    if log {
        log_values(data)
    } else {
        Ok(())
    };

    Ok(data)
}
~~~

Note that we cannot use `logging_on` directly as the condition of the `if`-expression and that the
`if`-expression needs to return a `Result` from both branches.

Utility functions can easily alleviate some of this, but Higher Kinded Types are required to make
many of them generic enough.

# Monad with state

The `Result` and `Option` monads only carry state in the type, eg.  `Option` has `Some(T)` and
`None` but there is no extra value describing any state. But there are monads which carry state
as another data-item, like the State, Iterator and Parser monads (the State monad would probably
not be very interesting for Rust, but list-comprehension and parser combinators certainly are).

The signature of the proposed `?` is "`M<T, E> -> T`", essentially the same as `unwrap` but with
the invisible addition that an early return or jump will be performed if the type decides that it
is in an "error" state.

Monadic bind on the other hand has the signature `M<T> -> (Fn*(T) -> M<U>) -> M<U>`; we have an
initial context `M<T>` which is then unwrapped to let the closure `Fn*(T) -> M<U>` act on it to
produce another wrapped value and then the returned `M<U>` is merged with the remaining state of 
the original `M<T>` (this is a simplification). Both the unwrapping and merging of the value is
under the control which implements the bind-operator and the original state is still available,
which means that context can be carried through in an appropriate way.

The closure denoted `Fn*` is one of the three closure types `FnOnce`, `FnMut` and `Fn` since
different types of bind-implementations have different requirements (eg. `Result` would use
`FnOnce` since it is just run once, whereas an iterator would need `FnMut` since the closure would
be executed once for each item). To let a trait-signature be generic over closures in this way is
something which is not yet possible in Rust at the moment, but once that is possible `do`-notation
should not be far off.

The proposal also mentions that the signature of `?` inside of `try` could be written as
`M<T, E> -> (FnOnce(T) -> R, FnOnce(E) -> R) -> R` if the compiler rewrites it to the trait-part of
the proposal. This `R` would then be wrapped using the static method `M<R, E>::normal` once it is
returned from the `try`-block, which has the signature `R -> M<R, E>`.

This is very similar to monadic bind, but with some key differences in its use and the return value
`R` is not actually merged in the monadic context `M`. By adding a `= M<U>` constraint to `R` we
can allow the wrapping method to investigate and update the state of any returned `M<U>` and
actually provide a proper monadic bind for the type. Though the proposal is lacking one very
important piece which is needed to make it at all possible, and that is how the state should be
carried over from the original `M<T, E>` to the new `M<R, E>`.

The trait-version of the proposal for `?` and `try`-blocks is essentially a do-notation for a
bi-monad (ie. a monad carrying two values) but without the needed restrictions on the types or
any way of carrying state from the left hand side to the return value. This makes it impossible
to actually use for anything but the most simple monad types.

# `try` vs `do`

Differences:

* `do` requires users to lift expressions into the used type whereas `try` requires users to unwrap
  values *out* of the type.
* `do` only works on one level, requiring nested expressions to use their own `do`-blocks when
  necessary. `try` allows the `?` to short-circuit the whole thing whenever needed.
* `try` automatically wraps the resulting value from the block whereas `do` requires the block to
  return a wrapped value.
* `do` allows the type to control the state-management between statements, `try` explicitly
  disallows the carrying of state between the original type and the resulting type.

Similarities:

* Both result in a wrapped value

# Alternatives

The `try`-blocks and `?`-operator are intrinsically tied to the execution around a "failure-state"
and does not consider any other type of state. There are alternatives which are more general and
would open up for different types of contextual-execution.

Personally I do not see any gain by the `catch` itself, since it can easily be constructed using
existing constructions in the language. The `try`-blocks or `do`-notation is another matter,
if implemented properly this would be a nice composable way of dealing with compuations in a
context.

## Method-position macros

Method-position macros could make the `?` operator without `try`-blocks superflous, sine we would
be able to write the following:

~~~rust
File::create("foo.txt").try!.write_all(b"Hello world!").try!;
~~~

It does not replace the `?` + `try`-blocks functionality but could serve as a good complement to
some kind of `do`-notation.

## `do`-notation

As detailed above it would be much more composable with different types compared to the
`try`-blocks which would be limited to just short-circuiting types with simple control-flow.

If we compare a few of the examples from the comments in the RFC-comments and how they look if
we use `do`-notation we can see that they are not so bad:

~~~rust
self.type_variables.borrow()
                   .probe(v)
                   .map(|t| self.shallow_resolve(t))
                   .unwrap_or(typ)
// is equivalent to:
try {
    let t = self.type_variables.borrow().probe(v)?;
    self.shallow_resolve(t)
} catch {
    _ => typ
}    
// which is equivalent to:
do {
    t <- self.type_variables.borrow().probe(v);
    Ok(self.shallow_resolve(t))
}.unwrap_or(typ)
~~~

Making a `Option<(A, B)>` from two `Option`:

~~~rust
// current:
a.and_then(|x| b.map(|y| (x, y)))
// try + ?:
try { (a?, b?) }
// do and map:
do { x <- a; b.map(|y| (x, y)) }
// only do:
do { x <- a; y <- b; Some((x, y)) }
~~~

Multiple `try!` macros in a row would also be easier to deal with, especially if their
success-value is not needed:

~~~rust
// from libsyntax, printing code fragments:
try!(self.space_if_not_bol());
try!(self.ibox(indent_unit));
try!(self.word_nbsp("let"));
// can be written as:
do {
    self.space_if_not_bol();
    self.ibox(indent_unit);
    self.word_nbsp("let")
}.try!
~~~

Personally I am a fan of `do`-notation since it much more general than just a control-flow-specific
language-construction and allows much more advanced ways of composing operations.

**EDIT:** Posted on reddit: [/r/rust](https://www.reddit.com/r/rust/comments/435572/blog_the_operator_and_try_vs_do/)

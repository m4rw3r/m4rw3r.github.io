---
layout: post
title: Rust and the Monad trait - Not just higher kinded types
permalink: rust-and-monad-trait
---

Higher kinded types is something which has been discussed a lot related to Rust in the past year,
both as a feature people want, but also as a feature people do not really know what to do with.
I am going to focus on the ``Monad`` trait in this post, since that is one of the things which
makes higher kinded types so appealing.

Rust is strict in trait definitions, both types and lifetimes need to match the definition exactly
and there is very little leeway except for using more generics. This is both good and bad, it
guarantees a lot of invariants for the trait but for higher kinded types like ``Monad`` and
``Functor`` it is maybe a bit too restrictive in its current form.

Of course this is not a full proposal &mdash; and most of this does not actually make sense as a
proper feature or syntax in a programming language &mdash; but it is more of an exploration of
what would be needed to properly implement a generic ``Monad`` trait. Hopefully this article can
serve as some kind of start for a discussion about a real RFC.

# Simple ``Monad<M<T>>`` definition

This is a definiton of ``Monad<M<T>>`` which is similar to the definition Scala uses, with the
difference that the ``Applicative`` trait is not involved:

```rust
trait Monad<M<T>> {
    fn bind<F, U>(self, F) -> M<U>
      where F: FnOnce(T) -> M<U>;

    fn unit(T) -> M<T>;
}
```

This looks pretty neat and works nicely for ``Option<T>``:

```rust
impl<T> Monad<Option<T>> for Option<T> {
    fn bind<F, U>(self, f: F) -> Option<U>
      where F: FnOnce(T) -> Option<U> {
        match m {
            Some(t) => f(t),
            None    => None,
        }
    }

    fn unit(t: T) -> Option<T> {
        Some(t)
    }
}
```

Though once we try to apply this on ``Result<T, E>`` we run into a problem: what do we do with
``E``?  ``impl<T, E> Monad<Result<T>> for Result<T, E>``? That will not work since ``Monad<M<T>>``
expects a type with only one type-parameter and ``Result`` has two. That leaves us with two
options, either denote the higher kinded parameter with a separate syntax, or allow for partial
application of type-constructors:

```rust
impl<T, E> Monad<R<T>> for R<T>
  where R<O> = Result<O, E> {
    fn bind<F, U>(self, f: F) -> R<U>
      where F: FnOnce(T) -> R<U> {
        match m {
          Ok(t)  => f(t),
          Err(e) => Err(e),
        }
    }

    fn unit(t: T) -> R<T> {
        Ok(t)
    }
}
```

The purpose of the syntax above would be to create a type alias ``R<O>`` as ``Result<O, E>`` for
any fixed ``E``, this would allow us to define ``Monad<M<T>>`` on ``R<O>`` since it only requires a
single type-parameter.

## ``Fn``, ``FnMut`` vs ``FnOnce``

Another problematic issue with ``Monad<M<T>>`` is the type of the function ``F`` passed to ``bind``;
it will require either ``Fn``, ``FnMut`` or ``FnOnce`` depending on how it is used. Some monads,
like ``Option<T>``, ``Result<T, E>`` and my own ``Parser`` monad are happily accepting ``FnOnce``
which is the most permissive generic for the user of ``bind`` since they are allowed to do
almost anything inside of ``F``.

On the other hand, implementing ``Monad<M<T>>`` for ``Iterator<Item=T>`` requires an ``FnMut`` bound
on ``F``, since ``F`` will be executed once for each item in the iterator and that disqualifies
``FnOnce``. And for some kind of ``Future<T>`` ``F`` will probably need to be something like
``FnOnce + Send + 'static``, and some parallel monad would want ``Fn + Send + 'static``.

``FnOnce`` is desirable since it allows the user of the monad to do simple sequencing without
putting any requirements on the code used, like this:

```rust
// Note: Not copy, and not clone
#[derive(Debug)]
struct Foo;

fn makeFoo()   -> Result<Foo, Error> { Ok(Foo) }
fn theAnswer() -> Result<i32, Error> { Ok(42) }

let r = makeFoo()
          .bind(|foo| theAnswer()
                        .bind(move |data| (foo, data)));

match r {
    Ok((foo, data)) => println!("Foo: {:?}, the answer is: {}", foo, data),
    Err(e)          => println!("Something went wrong: {:?}", e),
}
```

This typechecks nicely with a ``bind(M<T>, FnOnce(T) -> M<U>)``, but would fail if ``bind``
required ``F`` to be a ``FnMut`` or ``Fn`` since ``Foo`` in the code above is not ``Clone`` or
``Copy``. This kind of sequencing is necessary for any real-world usage of ``std::io`` or
``std::fs`` since most structures in those modules do not implement ``Clone`` (this includes
``File``, ``Metadata``, ``DirEntry`` and the list goes on).

Another reason for allowing ``FnOnce`` for ``bind`` is to avoid unnecessary allocations and copies
which both degrade performance and increases the risk of users being confused and frustrated by
accidentally operating on copies of the data or having to jump through hoops to satisfy the
type-checker.

This means that the ``Monad<M<T>>`` trait needs to also somehow be generic over the function-type
used by the concrete implementation while still enforcing the general signature of
``Fn*(T) -> M<U>``. This will require [impl specialization]
since ``FnOnce`` implements ``FnMut`` and ``FnMut`` implements ``Fn`` which means that any attempt
to unify all three ``Fn``* types under one generic will fail due to conflicting impl-blocks.

Another possibility is to have separate implementations of ``Monad``, ``MonadMut`` and
``MonadOnce`` where they all have a different ``Fn*`` type. In addition to this we provide default
implementations for the other compatible variants so that a ``MonadOnce`` can be used as a ``Monad``:

```rust
trait Unit<M<T>> {
    fn unit(T) -> M<T>;
}

trait Monad<M<T>>: Unit<M<T>> {
    fn bind<F, U>(self, F) -> M<U>
      where F: Fn(T) -> M<U>;
}

trait MonadMut<M<T>>: Unit<M<T>> {
    fn bind<F, U>(self, F) -> M<U>
      where F: FnMut(T) -> M<U>;
}

trait MonadOnce<M<T>>: Unit<M<T>> {
    fn bind<F, U>(self, F) -> M<U>
      where F: FnOnce(T) -> M<U>;
}

mod impls {
    use super::{Monad, MonadMut, MonadOnce};

    impl<T, N: MonadOnce<M<T>>> MonadMut<M<T>> for N {
        fn bind<F, U>(self, f: F) -> M<U>
          where F: FnMut(T) -> M<U> {
            MonadOnce::bind(self, f)
        }
    }

    impl<T, N: MonadMut<M<T>>> Monad<M<T>> for N {
        fn bind<F, U>(self, f: F) -> M<U>
          where F: Fn(T) -> M<U> {
            MonadMut::bind(self, f)
        }
    }
}
```

Example without higher kinded types: [playground](https://play.rust-lang.org/?gist=4480a38393cd27097bb2&version=stable).

The drawback of this is that the trait needs to be specified for each use, and the traits
themselves are incompatible in the reverse direction of the inheritance chain. This is not so
ergonomic for the user, what monad-trait should he/she use?

# The ``Iterator`` monad

Attempting to define the ``Monad`` implementation for an arbitrary ``I: Iterator`` is problematic
due to the type-parameters for the return types of ``bind`` and ``F`` since neither <code>I<&#85;></code> nor
``Iterator<Item=U>`` can be used because the source iterator (ie. the type after the ``for``
keyword)  is not of the same concrete type as the iterator returned by either ``F`` or ``bind``,
and plain traits are not allowed as a return-type.

My own ``Parser`` monad also has this issue, and it is also shared by the ``State``&ndash;,
``Future``&ndash;, and possibly some async-IO&ndash;monad. The base-type is generic but ``F``,
``bind`` and ``unit`` do not return the same concrete type (since they generally are all closures,
which means that their actual type is a compiler implementation-detail and cannot be described and
is always unique).

This would require us to be able to implement a trait for another trait, and not just a generic,
since the generic type would be the same ``Self`` in the return from ``F``, ``bind`` and ``unit``.
If this constraint remains in place it would be impossible to make a generic and extensible
monad implementing the ``Monad`` trait without having to box everything on every ``bind`` and
``unit`` operation which will cause horrible performance issues.

Here I assume that we can use ``impl Trait`` to denote that it is some concrete type implementing
``Trait`` (but not necessarily the same concrete type), which would allow us to use
``impl I<T> where I<X> = Iterator<Item=X>`` (we need to somehow alias the associated type into a
normal generic, or otherwise be able to use an associated type in HKT):

```rust
impl<T> Monad<impl I<T>> for impl I<T>
  where I<X> = Iterator<Item=X> {
    fn bind<U>(self, f: F) -> impl I<U>
      where F: FnMut(T) -> impl I<U> {
        // Create new iterator wrapping self and F yielding items from
        // the iterator f(self.next()). Essentially flat_map but
        // without the need for allocations.
    }

    fn unit(T) -> impl I<T> {
        // Iterator yielding the item once
    }
}
```

This monad would allow us to build lazy list-comprehensions and similar constructions:

```rust
let r: Vec<(u32, u32)> = 0..10.iter()
                              .bind(|i| i..10.iter()
                                             .bind(|j| Monad::unit((i, j))))
                              .collect();
// r = [(0, 0), (0, 1), ..., (1, 1), (1, 2), ..., (8, 8), (8, 9), (9, 9)]
```

Another possibility to solve this would be to use a newtype allowing ``impl Trait`` inside it. But
it might be problematic and confusing for the the user since suddenly you have a type which acts as
a ``Sized`` type in most aspects (can be stack allocated, can be passed by value, can be put inside
of structs without ``Box``es) but does not have the same size as any other instance of the same
type (eg. this will prevent you from putting multiple instances inside of a ``Vec`` without
``Box``ing the items first).

The existing closures behave just like that, so there is precedent for allowing types which behave
like this. But the downside of this approach is that the user would be required to do an explicit
cast the value by wrapping it in its newtype before being able to treat it as a ``Monad``.

## Lifetimes

So, let's say we got all the above working, now we have our ``Monad`` trait and we are done with this,
right? No, we still have another class of types which exists in Rust: lifetimes.

Lifetimes which are a part of the monad can be moved out by using partial application of
type-constructors and treating the lifetime as just another type parameter:

```rust
impl<'a, T> Monad<M<T>> for M<T>
  where M<X> = MyType<'a, X> {
    fn bind<U>(self, f: F) -> M<U>
      where F: FnOnce(T) -> M<U> { ... }

    fn unit(t: T) -> M<T> { ... }
}
```

On the other hand, a monad which is lazy (eg. the ``Iterator`` monad) will require ``Self`` and
``F`` to have the same lifetime as the returned <code>M<&#85;></code> (It will also require the same
``impl Trait`` or similar feature to allow different concrete types):

```rust
impl<T> Monad<I<T>> for impl I<T>
  where I<X> = Iterator<Item=X> {
    fn bind<'a, U>(self, f: F) -> impl I<U> + 'a
      where Self: 'a,
            F:    FnMut(T) -> impl I<U> + 'a {
        BindIter { source: self, fun: f, current: UnitIter(None) }
    }

    fn unit(t: T) -> I<T> {
        UnitIter(Some(t))
    }
}
```

Note the ``'a`` constraint on ``Self``, ``F`` and the return of ``bind``. This is necessary
because lifetime elision will restrict the lifetimes specified on ``bind`` to be (using
``+ 'lifetime`` to detail the restriction)
<code>fn bind(self + 'a, f: F + 'b) -> I<&#85;> + 'a where F: FnMut(T + 'c) -> I<&#85;> + 'c</code> and ``'c``
shorter than ``'b`` which is shorter than ``'a``. So either ``F`` (elide all lifetimes) or
``Self`` (when ``'a`` is added to ``F`` and the return of ``bind``) will not live long enough
without these extra annotations.

The ``Option``, ``Result`` &mdash; and other types which do not need ``F`` and/or ``Self`` to live as
long as the return from ``bind`` &mdash; will not use this constraint at all.

This means that ``Monad<M<T>>`` not only needs to be generic over the type it is implemented on
and the ``Fn*`` type it uses, it also needs to be generic over the lifetimes in the signature
of ``bind``. Another way to accomplish this would be to let traits be used as associated types,
then the desired function-trait can be used as an associated type by using the ``unboxed_closure``
feature.

So now we have a ``Monad<M<T>>`` trait which looks more like this:

```rust
// Need to be separate due to inference issues
trait Unit<M<T>> {
    fn unit(T) -> M<T>;
}

// Separate trait just for bind since F is intended to be used as a free
// generic specified by the trait-implementation.
trait Monad<M<T>, F>: Unit<M<T>> {
    fn bind<U>(self, F) -> M<U>
      // Let's assume we can use the trait Func and its associated types
      // Input and Output to enforce a function-signature
      where F: Func,
            F::Input  = (T,),
            F::Output = M<U>;
}
```

This enables us to add lifetimes to ``F``, and with the help of ``impl Trait`` in type-position
we can also add the same lifetime to ``Self`` and the return of ``bind``. It is a bit cumbersome,
but looks promising:

```rust
// Option monad
impl<T> Unit<Option<T>> for Option<T> {
    fn unit(t: T) -> Option<T> { Some(t) }
}

impl<T, F> Monad<Option<T>, F> for Option<T>
  // We leave out the return type definition to bind using the feature
  // unboxed_closure to be able to use the <> notation. If we are not
  // allowed to use this feature we also need to move U to the Monad trait
  where F: FnMut<(T,)> {
    fn bind<U>(self, f: F) -> Option<U>
      where F: Func,
            F::Input  = (T,),
            F::Output = Option<U> {
        match self {
            Some(t) => f(t),
            None    => None,
        }
    }
}

// Iterator monad
impl<T> Unit<I<T>> for impl I<T>
  where I<X> = Iterator<Item=X> {
    fn unit(T) -> impl I<T> { ... }
}

impl<'a, T, F> Monad<impl I<T>, F> for impl I<T>
  // Use the lifetime 'a here to restrict all uses of I<_>
  where I<X> = Iterator<Item=X> + 'a,
        // Here we can add that F needs to live at least as long as
        // the return from bind (and Self):
        F: FnMut<(T,)> + 'a {
    fn bind<U>(self, f: F) -> impl I<U>
      where F: Func,
            F::Input  = (T,),
            F::Output = impl I<U> {
        // Now we can put both self and f inside of something and
        // return it safely
        ...
    }
}
```

And now we actually have a ``Monad`` trait which allows for different ``Fn*`` types, different
lifetime requirements on the function passed to ``bind`` and it can also be implemented for
N-ary types as well as traits themselves.

## Receiver types

An additional, and slightly smaller, problem is receiver types. All parameters to ``bind`` have
so far been taken by value, which means that unless the type implementing the monad trait is
``Copy`` the original value would be invalidated once it is passed to ``bind``. This is a bit
problematic when you want to reuse the same piece of data multiple times without having to build
the monad computation anew every time it has to be used.

Haskell's GHC does some limited memoization to avoid this issue, eg. when performing repeated
calls to the same function with the same value it will only calculate the value one, which solves
this issue. Rust cannot do that since there is no runtime, and the value itself would have to be
``Clone``.

I am not sure I see too much use of ``&self``, and ``&mut self`` seems to be somewhat redundant
since you cannot use ``&mut self`` for anything else until the specific monad instance has been
computed.

The solution to these two would probably be to implement specific monad-traits for the
receiver-types. Hopefully this will not have to be combined with different traits for ``Fn*`` types
since that would result in a lot of different, somewhat-incompatible, monad-traits.

# Attempting to use the ``Monad`` trait in a generic way

The code above is all ok as long as we do not try to actually be generic over the ``Monad<M<T>>``
trait and let the compiler infer all the types. Then the ``Monad::bind`` and ``Monad::unit``
functions work pretty well. But what if we try to define functions generic over ANY ``Monad``?

```rust
fn liftM2<M, MT, MU, F, H, T, U>(f: F, m1: MT, m2: MU)
    -> impl Monad<F::Output, H>
  where M:  Monad<M<_>, _>,
        MT: M<T>,
        MU: M<U>,
        F:  Fn<(T, U)>,
        H:  Fn<(U,)> {
    Monad::bind(m1, move |x1| Monad::bind(m2, move |x2| Unit::unit(f(x1, x2))))
}
```

The above will actually not work. Firstly, the generic ``H`` will not be possible to infer.
Secondly, all the different ``Fn*`` parameters are creating lots of unnecessary noise as well as
restricts the user to the most basic use of ``bind``.  By using "associated traits" we can
simplify it a tiny bit:

```rust
fn liftM2<M, MT, MU, F, T, U>(f: F, m1: MT, m2: MU) -> impl Monad<F::Output>
  where M:  Monad<_>,
        MT: Monad<M<T>>,
        MU: Monad<M<U>>,
        F:  Fn<(T, U)> {
    Monad::bind(m1, move |x1| Monad::bind(m2, move |x2| Unit::unit(f(x1, x2))))
}
```

It is still very far from being ergonomic to use. I suspect that we will need to be able to
implement traits for types without having to specify their type-parameters, as well as using
traits and types which are partially applied as types and type-parameters. This would enable us
to use both the "associated traits" and the types implenting traits as a type-constructors if
their type-parameters are not fully specified:

```rust
fn liftM2<M, F, T, U>(f: F, m1: M<T>, m2: M<U>) -> M<F::Output>
  where M: Monad,
        F: M::BindFn<(T, U)> {
    Monad::bind(m1, move |x1| Monad::bind(m2, move |x2| Unit::unit(f(x1, x2))))
}
```

The above code requires the use of traits in place of concrete types, having the compiler
monomorphize them to concrete types without having to be explicitly generic over them. This is
starting to look a lot more like how ``Monad`` is declared  and used in Haskell, more so than how
it is in Scala, and it cuts down on a lot of noise which the extra generics otherwise would have
added.

# What is needed to be able to implement a generic ``Monad`` trait?

* Proper higher kinded types. Currently this can be emulated using associated types and
  trait-inheritance, but this has the downside that it cannot enforce the signature of the types
  used by ``F`` or returned from ``bind``.
* Partial application of type-constructors, to allow for implementing ``Monad`` for N-arity
  type-constructors as well as associated types.
* [impl specialization] or otherwise some unification for ``Fn*`` traits to enforce the function
  signature of ``F`` passed to ``bind``, since the ``Monad`` trait needs to be generic over the
  ``Fn*`` type itself.
* Type-equality constraints in ``where``-clauses: [#20041](https://github.com/rust-lang/rust/issues/20041)
  This will allow the ``F`` passed to ``bind`` to be a generic specified by the implementor and
  be enforced to have the correct function signature.
* Implementing a trait for another trait needs to have some kind of mechanism which allows the
  concrete type used to be different within the same ``impl``-block without failing to typecheck.
  Eg. all ``impl Trait`` inside of the same ``impl``-block could be considered to be equivalent to
  the generic ``T: Trait`` for typechecking of traits. Otherwise the <code>M<&#85;></code> return
  of ``F`` and ``bind`` will not typecheck for monads more complex than ``Option`` or ``Result``.
  Another solution would be to allow traits to be used in type-position and then let the compiler
  monomorphize.

The above also holds true for the ``Functor`` trait, since ``Option`` and ``Result`` can take
``F = FnOnce`` while ``Iterator`` can at most take ``F = FnMut``, ``Iterator``'s ``map`` also
returns a different concrete type compared to the original type.

There are a few additional features which can be added to this list if ease of use is considered,
and it is not just limited to these:

* Allowing traits to be implemented on type-constructors.
* Allowing types and function signatures to be generic over type-constructors.
* Allowing traits to be used in type-position, letting the compiler monomorphize the type.
* Allowing restrictions defined in traits on associated types.

This would allow us to hopefully write ``Monad`` as something more like this:

```rust
trait Monad
  // The type monad is implemented on must be a 1-arity type-constructor
  where Type: <_> -> {
    // Trait-parameter of 1-arity, inheriting from the Func trait
    type BindFn<A>: Func;

    // Here we assume we can use the syntax Type<...> to parameterize
    // the self type, same for BindFn:
    fn bind<'a, F, T, U>(self, F) -> Type<U>
      where Self      = Type<T> + 'a,
            F:          Type::BindFn<(T,)> + 'a,
            F::Output = Type<U>;

    fn unit<T>(T) -> Type<T>;
}

impl Monad for Option {
    trait BindFn<A> = FnOnce<A>;

    fn bind<'a, F, U>(self, f: F) -> Option<U>
      where Self      = Option<T> + 'a,
            F:          Type::BindFn<(T,)> + 'a,
            F::Output = Option<U> {
        match self {
            Some(t) => f(t),
            None    => None,
        }
    }

    fn unit<T>(t: T) -> Option<T> { Some(t) }
}
```

[impl specialization]: https://github.com/rust-lang/rfcs/pull/1210

**EDIT:** Posted on reddit: [/r/rust](https://www.reddit.com/r/rust/comments/3li3by/rust_and_the_monad_trait_not_just_higher_kinded/)

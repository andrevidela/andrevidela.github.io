---
layout: post
title: Binding Application in Idris
date: 2025-07-10 20:00:00
description: "The new binding application in Idris helps\n
  write programs with dependent pairs and other structures with\n
  lambda as the trailing argument. This post is a small collection\n
  of uses I have for it."
tags: Dependent-types, idris, programming
featured: true
---

I've recently implemented _binding_ application as a language feature in Idris. This feature allows writing types such as Dependent pairs in a more ergonomic way without relying on special compiler magic. Or rather, the compiler magic is made available to everyone. This post is a collection of uses for this feature.

This feature is not publicly available yet, but I intend to make it available in the near future.

## What is it?

Binding syntax and binding-application is an idea I had a couple of years ago and presented at
[WITS 2024](https://popl24.sigplan.org/details/wits-2024-papers/1/Binding-Syntax-for-Dependently-Typed-Programs).
It's a syntactic feature piggybacking on the function-space of a dependent programming language that
offers some customisation to the notion of _binding_. The main use case is to facilitate writing types such as
$$\Pi$$ or $$\Sigma$$ where the second argument is a function dependent on the type given in the first
argument. In Idris, one can mark such functions as `typebind` because they are meant
to bind a type argument, and any such function written `f (x : t) | g x` desugars to
`f t (\x : t => g x)` This pattern can be generalised to any function with a trailing lambda where
the second argument does not necessarily depend on the first. This syntax looks like this
`f (x <- e) | g x` and desugars to `f e (\x : ? => g x)`, those functions are `autobind` since the type is automatically inferred.

Let's move onto the use cases.

### Sigma

Dependent pairs, or sigma-types, are ubiquitous in programming with dependent types.
If you define your own sigma-type, you have to use a lambda to describe the type of the second projection.

```idris
record Sigma (a : Type) (b : a -> Type) where
  constructor (&&)
  first : a
  second : b first

RichList : Type -> Type
RichList a = Sigma Nat (\n => Vect n a) -- lambda here
```

By marking the type as binding, we can use a much more natural syntactic form.

```idris
typebind
record Sigma (a : Type) (b : a -> Type) where
  constructor (&&)
  first : a
  second : b first

RichList : Type -> Type
RichList a = Sigma (n : Nat) | Vect n a
```

### Exists

The Idris `base` library has a type called `Exists : (a : Type) -> (b : a -> Type) -> Type` which is exactly the same as the dependent pair, except that its first argument is erased. Marking the type as `typebind` allows writing `Exists (a : Type) | p a`, that syntax enables the natural mathematical reading "There exists a type `a` _such that_ `p` of `a` holds".

```idris
typebind
record Exists (a : Type) (b : a -> Type) where
  constructor (-!)
  0 index : a
  pred : b index
```
### Subset

A similar type exists in base except that the predicate is erased. This type represents values which are constrained by a predicate, but we don't need to carry the predicate at runtime.

```idris
typebind
record Subset (a : Type) (pred : a -> Type) where
  constructor (!-)
  value : a
  0 constraint : pred value
```

Such a type can be written `Subset (x : Nat) | LTE x 9` and can be read "a value `x` of type `Nat` _such that_ `x` is smaller or equal to `9`"

### Ornaments

Ornaments are type-descriptors, they contain all the necessary to build types within a dependently-typed programming language. Here is the most basic definitions:

```idris
  data Orn : (j : Type) -> (j -> i) -> Desc i -> Type where

    Say : Inverse e i -> Orn j e (Say i)

    Ask : Inverse e i -> Orn j e d -> Orn j e (Ask i d)

    Sig : (s : Type) -> {d : s -> Desc i} -> ((sv : s) -> Orn j e (d sv)) -> Orn j e (Sig s d)

    Del : (s : Type) -> (s -> Orn j e d) -> Orn j e d
```

We notice that both `Sig` and `Del` have the perfect shape for making use of binding application.  Therefore, we can rename them and mark them as binding:

```haskell
  data Orn : (j : Type) -> (j -> i) -> Desc i -> Type where

    Say : Inverse e i -> Orn j e (Say i)

	Ask : Inverse e i -> Orn j e d -> Orn j e (Ask i d)

    typebind
    Σ : (s : Type) -> {d : s -> Desc i} -> ((sv : s) -> Orn j e (d sv)) -> Orn j e (Sig s d)

    typebind
    Δ  : (s : Type) -> (s -> Orn j e d) -> Orn j e d
```

Every time we want to use `Σ` or `Δ`, we can use binding syntax, for example, the ornament decorating the `Nat` description into `Fin n` makes use of `Δ`:

```idris
NatD : Desc ()
NatD = Sig (#["Z", "S"]) $ \case
    "Z" => Say ()
    "S" => Ask () (Say ())

FinO : Orn NatT (const ()) NatD
FinO = Sig (#["Z", "S"]) $ \case
    "Z" => Δ (xn : NatT) | Say (Ok (S xn))
    "S" => Δ (xn : NatT) | Ask (Ok xn) (Say (Ok (S xn)))
```
### ForAll

Another type in `base` is the predicate transformer `All : (p : a -> Type) -> (xs : List a) -> Type`. It takes a predicate `p` and ensures it holds for every element of the list `xs`. There are some cases where writing the predicate is a bit awkward because it cannot be written point-free and requires a lambda. For this, one can use a binding alias:

```idris
autobind
ForAll : List a -> (a -> Type) -> Type
ForAll xs p = All p xs
```

Because the first argument is not a type, we use `autobind` instead of `typebind`. The syntax also makes clear that we are "iterating" over the list rather than using the list itself as the argument to the predicate.

This enables the syntax:

```idris
ForAll (x <- xs) | LTE x 9
```

Which reads "for all `x` in `xs` the predicate `LTE x 9` holds"

Without the binding syntax, we would have to write the following, which doesn't read as nicely:

```idris
All (\x => LTE x 9) xs
```
### ForSome

In addition to `All` there is `Any` a predicate transformer that ensures exactly one of the elements in the list satisfies the predicate. The alias can be written in the same way:

```idris
autobind
ForSome : List a -> (a -> Type) -> Type
ForSome xs p = Any p xs
```

And similarly we can write

```idris
ForSome (x <- xs) | LTE x 9
```

With the benefit that it reads nicely as "For some `x` in `xs`, the predicate `LTE x 9` holds"
### For-Loops

The most exciting application for me is the fact that we can have idiomatic-looking for-loops in the language.

A for-loop in its C interpretation is a loop defined by three components: An initial state, a break condition, and a state update. This is a very low-level representation of loops, so much so that subsequent languages have abstracted over the notion of loops by relying on _iterators_. While the details differ from language to language, the general idea remains: An iterator allows iterating over each element once using `for`. Here are a few examples where the iterator is `xs`:

```
// python
for x in xs:
  body

//scala
for (x <- xs) body

// zig
for (xs) |x| {
  body
}

// java
for (x : xs) {
  body
}

//rust
for x in xs {
  body
}
```

Back to Idris, it has a function `traverse` with the type `traverse : Traversable t => Applicative f => (a -> f b) -> t a -> f (t b)`, this type captures the notion that for every " iterator" `t`, we can perform a side-effect `f` and return a new list.
With binding application we can define `for` in Idris as an alias for `traverse`.

```haskell
autobind
for : Applicative f => Traversable t => t a -> (a -> f b) -> f (t b)
for = flip traverse
```

It enables the following syntax:

```idris
xs : List String
xs = ["hello",  "world", "!"]

main : IO ()
main = ignore $ for (x <- xs) |
  putStrLn x
```

Which mimics the syntax used in other programming languages to iterate on a list. `ignore` discards the result of the iteration.

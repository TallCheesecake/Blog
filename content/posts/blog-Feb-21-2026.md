+++
title = "Subtyping and Variance"
date = "2026-02-21"
description = "A guide on subtying and variance in rust!"
[taxonomies]
tags=["rust"]
+++

# Intro

*NOTE: A lot of these examples are taken from the
<a href="https://doc.rust-lang.org/nomicon/" target="_blank"
rel="noopener noreferrer">rustinomicon</a>*


## Preface / Prereqs

Be comfortable with Rust. When we say *co* or *contravariant* over `T`,
we are not referring to `T`'s memory layout. Instead, we are talking
about its **type-theory meaning**.

The theory behind all of this can feel a little dry, but we need some of
it in order to talk about variance precisely. A type constructor `F<T>`
(read: “F defined over T” or “F generic over T”) is something that takes
a type and produces a new type. For example, `Box<T>` is not the same as
`T`. Their guarantees and invariants are different. And as such we can
say that Box is a new type.

A subtype `T1` of a supertype `T2` is written as: `T1 <: T2`. This means
that anywhere a `T2` is required, we may substitute a `T1`. However, the
reverse is not necessarily true.

Intuitively, you can think of `T1` as being a strictly more useful type.
For example, consider *Dog* and *Animal*. If we only rely on behaviors
guaranteed by *Animal*, then a *Dog* is acceptable — and even more
specific — because every Dog is an Animal. The guarantees we can make
about a Dog (its invariants) are strictly stronger than those of Animal
in this context. So we say: `Dog <: Animal`. Please note that this is a
very intuitive analogy but can get very misleading and out of hand, try
and stick to the type theory and not animals.

# Example:

``` rust, linenos
let mut hello = "hello";
{
    let world = String::from("world");
    hello = &world;
}
println!("{hello}");
```

The intuitive problem is that `world` gets dropped in the block, and
then we try to read from it through `hello` after the drop.

The reason this is relavent is that the reason we can't do this is
exactly due to the rust compilers rules on subtyping and variance. (your
reading a niche blog on subtying and variance I'm pretty sure your good
on motivation)

# Variance

There are three main variances in Rust: Contravariant, Covariant, and
Invariant.

## Functions (Contravariance)

T1 and T2 are types. For example, Option is a type, and i64 is a type.
We define types very specifically by their invariants. In the case of
Option, one invariant is that it must always be either None or Some(T).
If it were instead None or Maybe(T), then by definition it could not be
called an Option.

```


T1 = Dog
T2 = Animal

Fn(T1) = functions that take Dog
Fn(T2) = functions that take Animal

Fn(T2) <: Fn(T1)

          
```

A function that can take an **Animal** can also take a **Dog**. However,
a function that only takes a **Dog** cannot accept all **Animals**.

Therefore, we can say that the type `Fn(T2)` is a subtype of `Fn(T1)`.
Everywhere we might want a function that takes a `T1` (Dog), we can
instead provide a function that takes a `T2` (Animal). The reverse is
not true, because not all Animals are Dogs.

This is called **contravariance**, because the usual subtyping
relationship has been flipped:

` T1 <: T2 ` becomes ` Fn(T2) <: Fn(T1) `.

That reversal of the subtype relationship is exactly what contravariance
means.

An easier way to think about this is in terms of lifetimes. As explained
in Jon Gjengset’s
<a href="https://www.youtube.com/watch?v=iVYWDIW71jk" target="_blank"
rel="noopener noreferrer">Crust of Rust: Subtyping and Variance</a> , we
can consider two function types:

```
let x: Fn(&'a str);
let y: Fn(&'static str);

&'static str <: &'a str
```

From category/type theory we know that a subtype T1 of a super type T2
is (loosely a type that can, in all places where a T2 is accepted a T1
can be used instead. But the opposite is not neccecarily true. We can
think of T1 being strictly more usefull than T2

Everywhere where we want to pass a Fn that takes a 'a str we can NOT
pass a 'static the invariants are not the same. Static can only live for
static where as 'a can live for any amount. From this we can conclude
that: that a function that takes the more general type is the more
usefull type. This indicates that the FUNCTION types x and y defined
above are contravariant over the type T.

```
Fn('a str) <: Fn(&'static str) 
```

See how the subtyping relationship got fliped? before we had static
subtying 'a but now its different. This is effectivly (in my
understanding) the deffinition of a change of variance (co to conta).

## Covariance

```

let x: &'static str;
let y: &'a str;
&'static str <: &'a str

          
```

References are **covariant** over their lifetime. A longer lifetime can
be safely downgraded to a shorter one.

What this means (as explained in the functions section) is that the
subtype is the “more specific” or more flexible one. Consider a
reference to a `T` with lifetime `'a` versus `'static`. Suppose our
reference is of lifetime `'b`, and `'b` may outlive `'a`. In that case,
using it could leave us with an **invalid pointer**. Therefore, a
reference to `'static` is always more flexible, because anywhere a
shorter-lived reference is needed, we can safely use the longer-lived
one and downgrade it.

Therefore, the type `&'a T` is covariant over `T`. This means that the
subtyping relationship between any `T1` and `T2` is preserved when
taking a reference. Unlike the function example discussed earlier, where
the subtyping relationship could change due to contravariance in the
argument type, a reference does not alter the original type
relationship.

## Invariance

I found this one the hardest to reason about, so let’s motivate it
naturally. Suppose we have:

```rust, linenos
fn assign(input: &mut T, val: T) {
    *input = val;
}

fn main() {
    let mut hello: &'static str = "hello";
    let world = String::from("world");
    assign(&mut hello, &world);
    println!("{hello}");
}
```

Now let’s try to apply the tools of covariance and contravariance to
reason about what happens to the type `&mut T`.

**CASE 1**: Contravariance (we can provide a more general T)

1.  We assign `hello` to a `'static str`.
2.  We mutate `hello` to hold a `'a str`, where `'a` is shorter than
    `'static`.
3.  We later use `hello`, expecting it to still be `'static`, but it now
    refers to a shorter-lived value.

This would allow a shorter lifetime to be written into a location that
originally required `'static`, leading to a potential use-after-free.

**CASE 2**: Covariance (we can provide a more specific T)

1.  We assign `hello` to a `'a str`.
2.  We mutate `hello` to hold a `'static str`.
3.  The lifetime relationships become inconsistent, breaking the
    guarantees the type system is supposed to enforce.

In both directions, mutation allows us to violate lifetime guarantees.
Because `&mut T` permits writing, neither covariance nor contravariance
is sound.

Therefore, we need a new tool: **invariance**. A type constructor `F<T>`
is invariant over `T` if there is *no* subtyping relationship between
`F<T1>` and `F<T2>`, even if `T1` and `T2` are in a subtype
relationship.

This means that for `&mut T`, the `T` must match exactly. If the types
differ in any way — including their lifetimes the assignment is not
allowed.

## Conclusion

In conclusion this isint really something you're ever gonna run into.
But if you do then now you know :). Also for a more endepth discution of
all the type and their variance check out:
<a href="https://doc.rust-lang.org/nomicon/" target="_blank"
rel="noopener noreferrer">rustinomicon</a>


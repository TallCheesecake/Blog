+++
title = "Lox Update"
date = "2026-03-12"
description = "Updates on the lox project"

[taxonomies]
tags=["rust", "lox"]
+++
# Intro

This blog is simply an update on the front end of my interpreter. I
anticipate being finished with the resolver by the 3rd week of March.

## Current Structure

The current structure is as follows: Scanner → Parser → Resolver →
Codegen

### Resolver

By far the most interesting section of the system is the resolver. I
have chosen to (for now, to keep things simple and manageable) use a
stack-based scoping structure.

``` rust, linenos
pub struct Resolver {
    pub ast: parser::Parser,
    pub current_id: usize,
    // Variable scopes
    pub vscope: Vec<Scope>,
    // Functions as declared in scope
    pub fscope: Vec<FunctionInfo>,
    // Function calls
    pub func_table: HashMap<String, usize>,
    // Functions as declared in scope
    pub cscope: Vec<ClassInfo>,
    // Class calls
    pub class_table: HashMap<String, usize>,
}
```


That is the current state (pun intended) of the resolver. Using a vector
of structs and a DFS search of the AST allows for effectively infinite
scoping while being (relatively) easy to implement. Simply popping
scopes off when you leave the scope. Scopes are implemented as an
operator applied to a \`Tree\` enum.


``` rust, linenos
pub enum Tree {
    Nil,
    ExprStatement(Vec<Tree>),
    Var(String, Rc<Tree>),
    Class {
        name: Atom,
        fields: Vec<Rc<Atom>>,
        members: Vec<Tree>,
    },
    Call {
        callee: Rc<Tree>,
        arguments: Vec<Tree>,
    },
    Atom(Atom),
    NonTerm(Op, Vec<Tree>),
    Op(Op),
    Fun {
        name: Rc<Tree>,
        parameters: Vec<Rc<Tree>>,
        body: Rc<Tree>,
    },
}
```


You end up with an S-expression-like syntax of: \`(group NonTerm\[x, y,
z\])\`. This naturally leads to a non-homogeneous AST, and since I
deemed mutating that AST a bad idea, that leads us into the visitor
pattern.

The visitor pattern is a design pattern where you split the data from
the algorithm. This is incredibly useful since, in Rust, shared mutable
state is not allowed, and doing so correctly with
\`Option\<Rc\<RefCell\>\>\` is a nightmare, especially since \`Tree\`
does not implement \`Copy\`.

The pattern allows me to qualify and analyze the user input (scoping and
arity resolution) without touching the AST. This not only preserves the
AST for later potential use but also means we can use a 2-pass
architecture without having to handle 2 different tree structures.

# Conclusion & Future

The IR (Intermediate Representation) I will be using needs to do two
things:

1.  It must be easy to turn into bytecode.
2.  It must permit me to perform simple optimizations in the future.

When I set out to make an interpreter, an optimizing IR was one of the
most interesting things about the project. However, since I don't have a
degree in CS and am a full-time student, I need to pick something
relatively easy to implement. Given that and the above two requirements,
I will be using an LLVM-style Linear SSA.

I've frankly not yet fully wrapped my mind around phi nodes and other
nuances, so I won't be able to give a great explanation, but the gist
is:

``` rust, linenos
let x = 2 + 1;
let x = x + 1;
let z = 2 + 1;
```

Here, if we convert this to SSA, we get:

``` rust, linenos
let x_1 = 2 + 1;
let x_2 = x_1 + 1;
let z_1 = 2 + 1;
```

This leads to the natural optimization that:

``` rust, linenos
z_1 = x_1
```


Saving us a math operation.

Anyways, that's it. Thanks!


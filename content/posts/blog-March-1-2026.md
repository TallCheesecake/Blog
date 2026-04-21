+++
title = "Unsafe LL"
date = "2026-03-01"
description = "A walkthrough of unsafe linked lists in rust" 

[taxonomies]
tags=["rust"]
+++
# Intro

This is a wrapper/commentary/walkthrough of the incredible: <a
href="https://rust-unofficial.github.io/too-many-lists/fourth-building.html"
target="_blank" rel="noopener noreferrer">Learning Rust With Entirely
Too Many Linked Lists</a>

## Linked Queue

A linked queue (a type of linked list) can be represented like this:

```
[front] -> [] -> [] -> [back] 
```

```
POP: Poped Off([front]) [new front] -> [] -> [back] 
```

```
PUSH: [front] -> [] -> [] -> [back] -> Pushed On([new_back]) 
```

So how do we do this? well we can start with

```rust, linenos
pub struct List<T> {
    head: Link<T>,
    tail: *mut Node<T>,
}

type Link<T> = Option<Box<Node<T>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
} 
```

Now for the operations, Push and Pop. 

```rust, linenos
pub fn push(&mut self, elem: T) {
    let mut new_tail = Box::new(Node {
        elem: elem,
        next: None,
    });
    let raw_tail: *mut _ = &mut *new_tail;
    if !self.tail.is_null() {
        unsafe {
            (*self.tail).next = Some(new_tail);
        }
    } else {
        self.head = Some(new_tail);
    }
    self.tail = raw_tail;
}
```

But there is a problem, and it's hidden from the normal borrow checker.
We need Miri, a nightly Rust interpreter, which interprets unoptimized
IR output from the Rust compiler at runtime. In Miri, we have a concept
of stacked borrows, which will be the meat of the remainder of the
article.

## Miri and Stacked Borrows

If we try and do this:

``` rust, linenos
let mut x = 1;
let y = &mut x;
let z = &mut x;
*y += 1;
*z += 1;
println!("{x}");
```

In normal safe Rust, this will error out with something like:

```
[ERROR]: Cannot borrow x as mutable twice
```

And the borrow checker is completely correct here. One of the main
asserts of the borrow checker is that you can only ever have one
(exclusive) mutable borrow out for a variable. Let's extend this to be a
little more general. If you have something like:

``` rust, linenos
fn do_something(input: &i32) {
    *input = 2
}
fn main() {
    let mut y = 1;
    let x = &y;
    do_something(x);
}
```

\`x\` can't, no matter what, be dereferenced to mutate \`y\`, because
this is an immutable reference. This allows for strong pointer reasoning
and compiler optimizations. Where things go wrong is:


The following code section is adapted from Jung et al's
paper<sup>1</sup>.


```rust, linenos
unsafe {
    let mut data = 10;
    let ALIAS = &mut data as *mut i32; 
    let ref1 = &mut data; 
    *ref1 += 1;
    *ALIAS = 2;
    println!("we expect to see: 11 and got: {}", ref1);
}
```

Here, this will output:

```
we expect to see: 11 and got: 2
```

But why? We have \`ALIAS\` and \`ref1\` BOTH pointing to \`data\`, but
the borrow checker has no knowledge of the raw mutable pointer. This
means that it doesn't know that \`ALIAS\` exists and naturally can't
stop us from mutating data, even though this should be illegal.

This is where Miri and stacked borrows come in. The concept is as
follows: each mutable reference goes on a stack. If you need to use a
reference that is not on the top of that stack, we pop everything above
it off and keep going. If something is not on the stack, you can't use
it as a reference.

The notation I will use is as follows: the stack will start at 0 and
grow higher. \`U\` stands for an exclusive or unique reference, and
\`ShRW\` means a Shared Read-Write pointer.

```rust, linenos
unsafe {
    let mut data = 10; // 0 U
    let ALIASE = &mut data as *mut i32; // ShRW
    let ref1 = &mut data; //1 U THIS COUNTS AS A USE OF DATA
    //Since this very action demands that
    //we have exclusive access to data but we dont
    //since this is a use
    //we can pop the bottom parts of the stack
    //excluding data since that would make no sense
    //
    //Now here this is ok since 1 U and 0 U are still on the stack
    *ref1 += 1;
    //this is not ok since ALIASE is not on the borrow stack and we
    //cant use things not on the stack
    *ALIASE = 2;
    //Here the borrow checker cant see 
    //that ALIASE derefs, this is a problem since
    //we can mutate data through ALIASE and 
    //since ALIASE aliases data alongside with
    //ref1 ref1s expectation of non 
    //shared unique refrence is invalid.
    println!("we expect to see: 11 and got: {}", ref1); 
    } 
```

Above we can see that we have a pointer alias and that data is illegally
modified. Without the stack model, Miri would not know that this is UB.
As commented above, the reference to data is considered a "use" in the
loosest sense of the word. This means that when we pop the
SharedReadWrite pointer alias off the borrow stack to allow for \`ref1\`
to exist, we can't then use the alias later to cause UB.

## UB and linked lists

The reason we even care about this is because of the code snippet from
the \`push\` method a little bit ago.

```rust, linenos
 let mut new_tail = Box::new(Node {
        elem: elem,
        next: None,
    });
    let raw_tail: *mut _ = &mut *new_tail;
    if !self.tail.is_null() {
        unsafe {
            (*self.tail).next = Some(new_tail);
        }
    } else {
        self.head = Some(new_tail);
    }
    self.tail = raw_tail; 
```

We actually have this occur, and Miri catches it.

```
 help: <122538> was created by a SharedReadWrite retag at offsets [0x0..0x10]
   --> src/lib.rs:30:32
    |
 30 |         let raw_tail: *mut _ = &mut *new_tail;
    |                                ^^^^^^^^^^^^^^
help: <122538> was later invalidated at offsets [0x0..0x10] by a Unique retag
   --> src/lib.rs:42:30
    |
 42 |             self.head = Some(new_tail);
    |                              ^^^^^^^^ 
```

What this means is that right now two things point to \`new_tail\`: the
raw pointer \`raw_tail\` and the list struct's head pointer. This means
that someone can, through the \`raw_tail\`, mutate the \`new_tail\`
pointer to point to something undefined :(

Something like:

```rust, linenos
self.head = Some(new_tail);
*raw_tail = evil_memory;
//Now self.head points to evil_memory :( 
```

The underlying problem here is that we are trying to mix safe and unsafe
code in a way that conflicts with guarantees. \`Box\`, which we are
currently using to make links in our linked list, owns its data and
effectively acts as a \`&mut\`. It would make our lives a lot easier if
we could mutate this more easily. We are already using unsafe code, so
having the \`Box\` there isn't really doing anything for us. Let's
instead use:

```rust, linenos
type Link = *mut Node;
```

and

```rust, linenos
pub fn push(&mut self, elem: T) {
    unsafe {
        // Immediately convert the Box into a raw pointer
        let new_tail = Box::into_raw(Box::new(Node {
            elem: elem,
            next: ptr::null_mut(),
        }));

        if !self.tail.is_null() {
            (*self.tail).next = new_tail;
        } else {
            self.head = new_tail;
        }

        self.tail = new_tail;
    }
}
```

Here’s what we’ve achieved: \`new_tail\` will no longer be optimized to
assume exclusive ownership. And everything else works just fine!!!

And frankly, that's it - those are the important parts. Everything else
is just filling out the rest of the methods.

## Refrences

\[1\] Ralf Jung, Hoang-Hai Dang, Jeehoon Kang, and Derek Dreyer. 2020.
Stacked Borrows: An Aliasing Model for Rust. Proc. ACM Program. Lang. 4,
POPL, Article 41 (January 2020), 32 pages.
https://doi.org/10.1145/3371109

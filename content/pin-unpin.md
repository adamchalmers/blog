+++
title = "What is Pin? What is Unpin?"
date = 2021-08-23

[taxonomies]
tags = ["rust", "programming", "async", "pin"]
+++

Using async Rust libraries is really easy. It's just like using normal Rust code, with a little `async` or `.await` here and there. But sometimes, if you're writing your own async Rust library, you need to actually go beneath the abstractions of async/await syntax and actually learn how Rust Futures work. The first time I tried this, I got really confused by arcane, esoteric syntax like `T: ?Unpin` and `Pin<&mut Self>`. I had never seen these types before, and I couldn't understand what they were doing. This is my attempt to write the explainer I wish I could have read back then. In this post, we're gonna learn Pin and Unpin mean, why they're necessary, and how to use them for writing async libraries.

<!-- more -->

# Motivation: let's write a nested Future!

A few years ago, I needed to write some code which would take some async function, run it and collect some metrics about it, e.g. how long it took to resolve. I wanted to write a type `TimedWrapper` that would work like this:

```rust
// Some async function, e.g. polling a URL with [https://docs.rs/reqwest]
// Remember, Rust functions do nothing until you .await them, so this isn't
// actually making a HTTP request yet.
let async_fn = reqwest::get("http://adamchalmers.com");

// Wrap the async function in my hypothetical wrapper.
let timed_async_fn = TimedWrapper::new(async_fn);

// Call the async function, which will send a HTTP request and time it.
let (resp, time) = timed_async_fn.await;
println!("Got a HTTP {} in {}ms", resp.unwrap().status(), time.as_millis())
```

I like this interface, it's simple and should be easy for the other programmers on my team to use. OK, let's implement it! I know that, under the hood, Rust's async functions are just regular functions that return a [`Future`](https://doc.rust-lang.org/stable/std/future/trait.Future.html). The `Future` trait is pretty simple. It just means a type which:

 * Can be polled
 * When it's polled, it might return "Pending" or "Ready"
 * If it's pending, you should poll it again later
 * If it's ready, it responds with a value. We call this "resolving".

Here's a really easy example of implementing a Future. Let's make a Future that returns a random `u16`.

```rust
use std::{future::Future, pin::Pin, task::Context}

/// A future which returns a random number when it resolves.
#[derive(Default)]
struct RandFuture;

impl Future for RandFuture {
    // Every future has to specify what type of value it returns when it resolves.
    // This particular future will return a u16.
    type Output = u16;

    // The `Future` trait has only one method, named "poll".
    fn poll(self: Pin<&mut Self>, _cx: &mut Context) -> Poll<Self::Output> {
        Poll::ready(rand::random())
    }
}
```

Not too hard! I think we're ready to implement `TimedWrapper`. Let's start by defining the type.

```rust
pub struct TimedWrapper<Fut: Future> {
    start: Option<Instant>,
    future: Fut,
}
```

OK, so a `TimedWrapper` is generic over a type `Fut`, which must be a `Future`. And it will store a future of that type as a field. It'll also have a `start` field which will record when the first was first polled. Let's write a constructor:

```rust
impl<Fut: Future> TimedWrapper<Fut> {
    pub fn new(future: Fut) -> Self {
        Self { future, start: None }
    }
}
```

Nothing too complicated here. The `new` function takes a future and wraps it in the `TimedWrapper`. Of course, we have to set start to None, because it hasn't been polled yet. So, let's implement the `poll` method, which is the only thing we need to implement `Future` and make it `.await`able.

```rust
impl<Fut: Future> Future for TimedWrapper<Fut> {
    // This future will output a pair of values:
    // 1. The value from the inner future
    // 2. How long it took for the inner future to resolve
    type Output = (Fut::Output, Duration);

    fn poll(self: Pin<&mut Self>, cx: &mut Context) -> Poll<Self::Output> {
        // Call the inner poll, measuring how long it took.
        let start = self.start.get_or_insert_with(Instant::now);
        let inner_poll = self.future.poll(cx);
        let elapsed = self.elapsed();

        match inner_poll {
            // The inner future needs more time, so this future needs more time too
            Poll::Pending => Poll::Pending,
            // Success!
            Poll::Ready(output) => Poll::Ready((output, elapsed)),
        }
    }
}
```

OK, that wasn't too hard. There's just one problem: this doesn't work.

![Rustc compiler error](/pin-unpin/poll_err.png)

So, the Rust compiler reports an error on `self.future.poll(cx)`, which is "no method named `poll` found for type parameter `Fut` in the current scope". This is confusing, because we know `Fut` is a `Future`, so surely it has a poll method? OK, but Rust continues: `Fut` doesn't have a poll method, but `Pin<&mut Fut>` has one. What is this weird type?

Well, we know that methods have a "receiver", which is some way it can access `self`. The receiver might be `self`, `&self` or `&mut self`, which mean "take ownership of self," "borrow self," and "mutably borrow self" respectively. So this is just a new, unfamiliar kind of receiver. Rust is complaining because we have `Fut` and we really need a `Pin<&mut Fut>`. At this point I have two questions:

1. What is `Pin`?
2. If I have a `T` value, how do I get a `Pin<&mut T>`?

The rest of this post is going to be answering those questions. I'll explain a problem you might face when writing Rust data structures, and how the `Pin` type solves it.

# Why does Pin exist?

Pin exists to solve a very specific problem: self-referential datatypes, i.e. data structures which have pointers into themselves. For example, a binary search tree is has self-referential pointers, because nodes have references to other nodes in the same structure.

Self-referential types can be really useful, but they're also hard to make memory-safe. To see why, let's use this example type with two fields, an i32 called `val` and a pointer to an i32 called `pointer`.

![A self-referential struct where all pointers are valid](/pin-unpin/node_diagram.jpg)

At first, all is well. The `pointer` field points to the `val` field in memory address A, which contains a valid i32. So far all the pointers are _valid_, i.e. they point to memory that has the right type (in this case, an i32). But what happens if we move it?

![Moving invalidates the pointer](/pin-unpin/node_diagram2.jpg)

When we move it, the `pointer` field doesn't change its value. It's still pointing at address A, which now doesn't have a valid i32. The data that was there was moved to address B, but the pointer wasn't updated! So now the pointer is invalid. This is bad -- at best, invalid pointers cause crashes, at worst they cause hackable vulnerabilities. We only want to allow memory-unsafe behaviour in `unsafe` blocks, and we should be very careful to document this type and tell users to update the pointers after moves.

To recap, all Rust types fall into two categories.

1. Types that are safe to move around in memory. This is the default, the norm. For example, this includes primitives like numbers, strings, bools, as well as structs or enums entirely made of them. Most types fall into this category!
2. Self-referential types, which are _not_ safe to move around in memory. These are pretty rare. An example is the [intrusive linked list inside some Tokio internals](https://docs.rs/tokio/1.10.0/src/tokio/util/linked_list.rs.html). Another example is most types which implement Future and also borrow data, for reasons [explained in the Rust async book](https://rust-lang.github.io/async-book/04_pinning/01_chapter.html).

Types in category (1) are totally safe to move around in memory. You won't invalidate any pointers by moving them around. But if you move a type in (2), then you invalidate pointers and can get undefined behaviour, as we saw before. In earlier versions of Rust, you had to be really careful using these types to not move them, or if you moved them, to use `unsafe` and update all the pointers. But since Rust 1.33, the compiler can automatically figure out which category any type is in, and make sure you only use it safely.

Any type in (1) implements a special auto trait called [`Unpin`](https://doc.rust-lang.org/stable/std/marker/trait.Unpin.html). Weird name, but its meaning will become clear soon. Again, most "normal" types implement `Unpin`, and because an auto trait (like `Send` or `Sync` or `Sized`[^1]), so you don't have to worry about implementing it yourself. If you're unsure if a type can be safely moved, just check it on [docs.rs](https://docs.rs) and see if it impls `Unpin`!

To use types in (2) safely, we don't use regular pointers for self-reference. Instead, we use special pointers that "pin" their values into place, ensuring they can't be moved. This is exactly what the [`Pin`](https://doc.rust-lang.org/stable/std/pin/struct.Pin.html) type does.

![Pin wraps some pointer, stopping its value from moving](/pin-unpin/pin_diagram.png)

Pin wraps a pointer and stops its value from moving. The only exception is if the value impls `Unpin` -- then we know it's safe to move. Voila! Now we can write self-referential structs safely! This is really important, because as discussed above, many Futures are self-referential, and we need them for async/await.

# Using Pin

So now we understand why Pin exists, and why our Future poll method has a pinned `&mut self` to self instead of a regular `&mut self`. So let's get back to the problem we had before: I need a pinned reference to the inner future. More generally: given a pinned struct, how do we access its fields?

The solution is to write helper functions which give you references to the fields. These references might be normal Rust references like `&mut`, or they might _also_ be pinned. You can choose whichever one you need. This is called _projection_: if you have a pinned struct, you can write a projection method that gives you access to all its fields.

Projecting is really just getting data into and out of Pins. For example, we get the `start: Option<Duration>` field from the `Pin<&mut self>`, and we need to put the `future: Fut` into a Pin so we can call its `poll` method). If you read the [`Pin` methods](https://doc.rust-lang.org/stable/std/pin/struct.Pin.html) you'll see this is always safe if it points to an `Unpin` value, but requires `unsafe` otherwise.

```rust
// Putting data into Pin
pub        fn new          <P: Deref<Target:Unpin>>(pointer: P) -> Pin<P>;
pub unsafe fn new_unchecked<P>                     (pointer: P) -> Pin<P>;

// Getting data from Pin
pub        fn into_inner          <P: Deref<Target: Unpin>>(pin: Pin<P>) -> P;
pub unsafe fn into_inner_unchecked<P>                      (pin: Pin<P>) -> P;
```

I know `unsafe` can be a bit scary, but it's OK to write unsafe code! I think of unsafe as the compiler saying "hey, I can't tell if this code follows the rules here, so I'm going to rely on you to check for me." The Rust compiler does so much work for us, it's only fair that we do some of the work every now and then. If you want to learn how to write your own projection methods, I can highly recommend [this fasterthanli.me blog post](https://fasterthanli.me/articles/pin-and-suffering) on the topic. But we're going to take a little shortcut.

# Using pin-project instead

So, OK, look, it's time for a confession: I don't using `unsafe`. I know I just explained why it's OK, but still, given the option,

![I would prefer not to](/pin-unpin/zizek_no.png)

I didn't start writing Rust because I wanted to carefully think about the consequences of my actions, damnit, I just want to go fast and not break things. Luckily, someone sympathized with me and made a crate which generates totally safe projections! It's called [pin-project](https://docs.rs/pin-project) and it's _awesome_. All we need to do is change our definition:

```rust
#[pin_project::pin_project] // This generates a `project` method
pub struct TimedWrapper<Fut: Future> {
    // For each field, we need to choose whether `project` returns an
    // unpinned (&mut T) or pinned (Pin<&mut T>) reference to the field.
    // By default, it assumes unpinned:
    start: Option<Instant>,
    // Opt into pinned references with this attribute:
    #[pin]
    future: Fut,
}
```

For each field, you have to choose whether its projection should be pinned or not. By default, you should use a normal reference, just because they're easier and simpler. But if you know you need a pinned reference -- for example, because you want to call `.poll()`, whose receiver is `Pin<&mut Self>` -- then you can do that with `#[pin]`.

Now we can finally poll the inner future!

```rust
fn poll(self: Pin<&mut Self>, cx: &mut Context) -> Poll<Self::Output> {
    // This returns a type with all the same fields, with all the same types,
    // except that the fields defined with #[pin] will be pinned.
    let mut this = self.project();

    // Call the inner poll, measuring how long it took.
    let start = this.start.get_or_insert_with(Instant::now);
    let inner_poll = this.future.as_mut().poll(cx);
    let elapsed = start.elapsed();

    match inner_poll {
        // The inner future needs more time, so this future needs more time too
        Poll::Pending => Poll::Pending,
        // Success!
        Poll::Ready(output) => Poll::Ready((output, elapsed)),
    }
}
```

Finally, our goal is complete -- and we did it all without any unsafe code.

# Summary

If a Rust type has self-referential pointers, it can't be moved safely. After all, moving doesn't update the pointers, so they'll still be pointing at the old memory address, so they're now invalid. Rust can automatically tell which types are safe to move (and will auto impl the `Unpin` trait for them). If you have a `Pin`-ned pointer to some data, Rust can guarantee that nothing unsafe will happen (if it's safe to move, you can move it, if it's unsafe to move, then you can't). This is important because many Future types are self-referential, so we need `Pin` to safely poll a Future. You probably won't have to poll a future yourself (just use async/await instead), but if you do, use the [pin-project](https://docs.rs/pin-project) crate to simplify things.

I hope this helped -- if you have any questions, please [ask me on Twitter](https://twitter.com/adam_chal). And if you want to get paid to talk to me about Rust and distributed systems, my team at Cloudflare is hiring, so get in touch at npunyzref@pybhqsyner.pbz (apply ROT13).

---

[^1] OK _technically_ `Sized` isn't an auto trait, but it basically is, because the compiler has a special case for it. See [this discussion](https://github.com/pretzelhammer/rust-blog/issues/28) for why!

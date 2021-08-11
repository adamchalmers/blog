+++
title = "What is Pin? What is Unpin?"
date = 2021-08-10

[taxonomies]
tags = ["rust", "programming", "async"]
+++

Using async Rust libraries is really easy. It's just like writing synchronous Rust code. You just occasionally sprinkle async on your functions, or .await on your function calls, and then you're done. But sometimes, if you're writing your own async Rust library, you need to actually go beneath the abstractions of async/await syntax and actually learn how Rust Futures work. This post is for anyone who's comfortable using async/await, maybe even methods from the [futures crate](https://docs.rs/futures), but who doesn't really know how to implement their own Futures. I'll explain what Pin and Unpin are, and how you can use them to make your own Futures.

# Motivation: let's write a nested

<!-- more -->
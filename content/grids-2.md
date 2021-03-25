+++
title = "Grids in Rust, part 2: const generics"
date = 2021-03-24

[taxonomies]
tags = ["rust", "programming"]
+++

In [part one](/grids-1), we built two different implementations of a Grid datatype. Benchmarks revealed that a single/1D vector was a better choice than a nested/2D vector. In this post, we'll write a new implementation that uses arrays instead of Vec. This should be faster!

<!-- more -->

# Stack, Heap and Arrays

When Rust allocates memory for a value, it can allocate on either the stack or the heap. [This page](http://web.mit.edu/rust-lang_v1.25/arch/amd64_ubuntu1404/share/doc/rust/html/book/first-edition/the-stack-and-the-heap.html#runtime-efficiency) explains the difference between the two, but here's my attempt to summarize: _stack fast, heap slow_. But, also, _stack restrictive, heap flexible_. Data structures that require following pointers go to the heap. Data structures whose layout can be figured out by the compiler at compiletime can go on the stack.

This "stack vs. heap" binary is perfectly represented by "array vs. Vec".The [docs](https://doc.rust-lang.org/std/vec/struct.Vec.html#guarantees) state that for a Vec, "the memory it points to is on the heap... and its pointer points to `len` initialized, contiguous elements in order." OK, so Vecs inherently use the heap and pointers. This makes them slower than stack-allocated data.

On the other hand, because [arrays](https://doc.rust-lang.org/stable/std/primitive.array.html) have a fixed size, the compiler knows exactly how much RAM they require, and can store them on the stack. The downside is that an array type `[T; N]` has exactly `N` elements, no more, no less, and you have to know the value of `N` at compile-time. You can't determine `N` using user input, or a command-line flag.

Well, it just so happens that for my use-case (a [ray tracer](https://github.com/adamchalmers/raytracer))




Thanks for reading! If you have any questions or suggestions, please let me know on [twitter](https://twitter.com/adam_chal) or via [email](mailto:adam.s.chalmers@gmail.com). The code is [on GitHub](https://github.com/adamchalmers/const_generic_grid) and if you have suggestions for improving it, feel free to open a PR. Thanks!

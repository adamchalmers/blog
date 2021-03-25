+++
title = "Grids in Rust, part 2: const generics"
date = 2021-03-24

[taxonomies]
tags = ["rust", "programming"]
+++

In [part one](/grids-1), we defined a Grid trait and implemented it using 1D and 2D vectors. Benchmarks revealed that a 1D vector was a better choice than a nested 2D vector. In this post, we'll write a new implementation that uses arrays instead of Vec. This should be faster!

<!-- more -->

# Stack and heap

When Rust allocates memory for a value, it can allocate on either the stack or the heap. [This page](http://web.mit.edu/rust-lang_v1.25/arch/amd64_ubuntu1404/share/doc/rust/html/book/first-edition/the-stack-and-the-heap.html#runtime-efficiency) explains the difference between the two, but here's my attempt to summarize

   ? | Stack  | Heap
--------|------|------
    Speed of allocation | Fast | Slow
  Restrictions | No pointers, fixed size | None

Data structures that require following pointers go to the heap. Data structures whose layout can be figured out by the compiler at compile-time can go on the stack.

This "stack vs. heap" binary is perfectly represented by "array vs. Vec". The [docs](https://doc.rust-lang.org/std/vec/struct.Vec.html#guarantees) state that for a Vec, "the memory it points to is on the heap... and its pointer points to `len` initialized, contiguous elements in order." OK, so Vecs inherently use the heap and pointers. This makes them slower than stack-allocated data.

On the other hand, because [arrays](https://doc.rust-lang.org/stable/std/primitive.array.html) have a fixed size, the compiler knows exactly how much RAM they require, and can store them on the stack. The downside is that an array type `[T; N]` has exactly `N` elements, no more, no less, and you have to know the value of `N` at compile-time. You can't determine `N` using user input, or a command-line flag.

Well, it just so happens that for my use-case (a [ray tracer](https://github.com/adamchalmers/raytracer)), I know exactly the exact width and height of the grid at compile-time. So I can make a new implementation of Gridlike backed by arrays, not Vecs. There's one little problem, though.

In my binary, I know the exact width and height of the grid at compile-time. But I'd like to have `Gridlike` trait and its implementations stay in their own reusable library. This means the grids should be able to store any size. After all, there's no point publishing a library with one type for a 2x3 grid, another type for a 2x4 grid, etc. Instead, the Array-based grids should support any width and height, just like the Vec-based grids did. And yet, arrays have to have sizes known at compile time. It seems like these two requirements contradict each other, and until recently, it would have been impossible to satisfy them both. But luckily, Rust just shipped a new feature which enables exactly this sort of requirement.

# Const generics

Rust now supports [const generics](https://blog.rust-lang.org/2021/02/26/const-generics-mvp-beta). You should really read the article to understand what they are, but here's my summary. Traditional generic uses, like `Vec<T>`, are generic over _types_. The generic parameter `T` will always get replaced by a concrete type like `i32` or `String` during compilation. When your program is compiled, there's no more `T`, just `Vec<i32>` or `Vec<String>`. For example, this tiny program defines a function named `last` which is generic over a type `T`.

```rust
// T could be any type
fn last<T>(v: Vec<T>) -> Option<T> {
    v.pop()
}

fn main() {
    // T is replaced with the concrete type &str
    assert_eq!(Some("world"), last(vec!["hello", "world"]));
    // T is replaced with the concrete type bool
    assert_eq!(Some(false), last(vec![true, false]));
}
```

At compile-time, rustc examines every time `last` is called, and figures out the right type to substitute for `T`. The first time it's called, `T = &'static str`. The next time, `T = bool`.

Const generics are very similar. But instead of being generic over a _type_, they're generic over a _constant value_, like `1` or `33` or `false`. For example, the type `[bool; N]` is an array of `N` booleans. When the code is compiled, `N` will be replaced with some constant value, and the compiler will generate an array with a specific length like `[bool: 4]`.

Just like how the concrete type `T` in `last<T>` had to be known at compile-time, the concrete value for `N` has to be known at compile-time. In Rust, we call any value that can be calculated at compile-time a _constant_. Hence, _const generics_. Let's look at an example:

```rust
// T could be any type
// W and H could be any usize value
pub struct Grid<T, const W: usize, const H: usize>
{
    array: [[T; W]; H],
}
```

Here we define a type `Grid` with three generic parameters. `T` is a normal generic over types. `W` and `H` are generic over `usize` values. This means that at compile-time, the grid will have a specific `W` and `H` value, and allocate a 2D array with that width and height. `T`, `W` and `T` are generic in the definition of `Grid`, but will have to be replaced with concrete values at compile-time, just like always with generics.

# 2D array grids with const generics

Now that we know how to make a type which is generic over array lengths, let's use it to make a new `Gridlike` implementation. The constructor for this array-based grid is almost identical to the constructor for the Vec-based grid:

```rust

impl<T, const W: usize, const H: usize> Default for Grid<T, W, H>
where
    T: Default + Copy,
{
    fn default() -> Self {
        Self {
            array: [[T::default(); W]; H],
        }
    }
}
```

How do we actually instantiate this type? It's pretty easy! We just have to ensure the Rust compiler knows what specific values of W and H the binary needs. For example:

```rust
let g: Grid<usize, 3, 4> = Default::default();
```

Or perhaps:

```rust
const WIDTH: usize = 300;
const HEIGHT: usize = 200;
let g: Grid<usize, WIDTH, HEIGHT> = Default::default();
```

The `Default` implementation will be the only constructor we need for this post. You can easily add more constructors if you'd like, maybe `fn new(t: T) -> Self` that returns a `Grid` where all cells have the same value `t`. The `Gridlike` implementation is also pretty similar:

```rust
impl<T, const W: usize, const H: usize> Gridlike<T> for Grid<T, W, H> {
    fn width(&self) -> usize {
        W
    }

    fn height(&self) -> usize {
        H
    }

    fn get(&self, p: Point) -> &T {
        &self.array[p.y][p.x]
    }

    fn set_all_parallel<F>(&mut self, setter: F)
    where
        F: Send + Sync + Fn(Point) -> T,
        T: Send,
    {
        use rayon::prelude::*;
        self.array.par_iter_mut().enumerate().for_each(|(y, row)| {
            for (x, item) in row.iter_mut().enumerate() {
                *item = setter(Point { x, y });
            }
        });
    }
}
```

Note that you can return `W` in a method (e.g. `fn width`) as though it were any old `usize` value. Pretty _freakin' sweet_ in my opinion.

Now, this grid is 2D, and in [part one](/grids-1) we found that 2D Vec grids were slower than 1D Vec grids. But they're slow because a 2D vec requires following one pointer per row, which lowers the CPU's cache hit rate. A 2D array doesn't have that problem, because all `W*H` elements are laid out in memory contiguously. But to get some fair benchmarks, we should try implementing a 1D array-based grid too. This is gonna require some const generics stuff that isn't actually stabilized yet. This means, for the first time in my life, I'm going to have to use... Nightly Rust.

# 1d array grids with const generic arithmetic

To get a 1D array which can store all the elements in the grid, we'll need an array of length `N` where `N = W * H`. We need to do arithmetic with those const generic values, something like this:

```rust
pub struct Grid<T, const W: usize, const H: usize>
{
    array: [T; W * H],
}
```

In theory, this should be totally fine. After all, rustc will know the values of `W` and `H` at compile-time, and it can certainly multiply two numbers at compile-time. In practice, though, implementing this compiler feature properly (for all types, not just `usize`) is pretty complicated, as the const generics team explains [here](https://blog.rust-lang.org/2021/02/26/const-generics-mvp-beta#const-generics-with-complex-expressions). So the feature is still under development. Luckily, we can install an in-development version of the compiler, called "Nightly", and enable this in-progress feature.

If you haven't installed the Nightly compiler before, don't worry, neither had I until implementing this type! It's easy:

```bash
$ rustup install nightly
```
And to make sure your current Cargo project uses the Nightly compiler, just run this from inside your project:
```bash
$ rustup override set nightly
```

Now that we're using the Nightly compiler, we can enable in-progress features (also known as "unstable" features), by adding this to `lib.rs`:

```rust
#![feature(const_evaluatable_checked)]
#![feature(const_generics)]
```

OK, now we can start doing arithmetic with const values. The implementation of the 1D array grid is pretty similar to the 1D vec grid and the 2D array grid. The one strange part is the `where` clause, but I'll explain that in just a second.

```rust
use crate::{Gridlike, Point};

// Define the data structure
pub struct Grid<T, const W: usize, const H: usize>
where
    [(); W * H]: Sized,
{
    array: [T; W * H],
}

// Constructor
impl<T, const W: usize, const H: usize> Default for Grid<T, W, H>
where
    [(); W * H]: Sized,
    T: Default + Copy,
{
    fn default() -> Self {
        Self {
            array: [T::default(); W * H],
        }
    }
}

// Implement Gridlike
impl<T, const W: usize, const H: usize> Gridlike<T> for Grid<T, W, H>
where
    [(); W * H]: Sized,
{
    // Just like the 2D Array
    fn width(&self) -> usize {
        W
    }

    // Just like the 2D Array
    fn height(&self) -> usize {
        H
    }

    // Just like the 1D Vec
    fn get(&self, p: Point) -> &T {
        &self.array[p.y * W + p.x]
    }

    // Just like the 1D Vec
    fn set_all_parallel<F>(&mut self, setter: F)
    where
        F: Send + Sync + Fn(Point) -> T,
        T: Send,
    {
        use rayon::prelude::*;
        self.array.par_iter_mut().enumerate().for_each(|(i, item)| {
            *item = setter(Point { x: i % W, y: i / W });
        });
    }
}
```

The one weird thing about this is the where clause, `where [(); W * H]: Sized`. Initially I didn't include that, but then the compiler complained about an "unconstrained generic constant". I [talked to a Rust dev on twitter](https://twitter.com/ekuber/status/1372017934910943237), and he said that the where clause won't be necessary when this const generic arithmetic feature is finished. It's a bit weird, but hey, the feature is still under development. I'm impressed that it works without panicking, I really don't mind putting an unnecessary where clause in while the Rust team polishes this feature.

> Aside: the Rust core team is so helpful. If I tweet a question to the [@rustlang](twitter.com/rustlang) account, they often retweet it so that it gets seen by a dev who can answer. The [Rust discord](https://discord.com/invite/rust-lang) is also really helpful. Don't be afraid to ask for help! Folks there are friendly and will point you in the right direction.

Now that we have 1D and 2D array-based implementations, let's benchmark them.

# Benchmarks: 1d vs. 2d, Vec vs. array

Before running the benchmarks, I took a second to make a prediction. My hypothesis was that the array implementations would be faster than the Vec implementations, and that the 1D array would be faster than the 2D array.

The benchmarks themselves are basically identical to the benchmarks from the first part, just with our new array-based implementations added. We're benchmarking each of the 4 Gridlike implementations in three different scenarios (setting all elements, getting elements in order, getting random elements). You can view the [full benchmark code on GitHub](https://github.com/adamchalmers/const_generic_grid/blob/master/benches/my_benchmark.rs). Remember to use [`criterion::black_box`](https://docs.rs/criterion/0.3.4/criterion/fn.black_box.html) to avoid the compiler skipping all your code because it (correctly, but unhelpfully) infers that it doesn't actually make any difference to the program output.

The results are _pretty close_ for the Get benchmarks:
![Benchmark GetRandom](/grids-2/getrandom.png)
![Benchmark GetOrder](/grids-2/getorder.png)

As for the Set benchmark, well, 2D Vec is so slow that it makes the results basically unreadable.
![Benchmark Set](/grids-2/set_all.png)

If we recalculate the graph, excluding 2D Vec, it's much easier to read:

![Benchmark Set](/grids-2/set_without_2d_vec.png)

_Very_ interesting! 1D array is consistently slower than 2D array. I suspect this is because the 1D array `fn set_all_parallel` implementation incurs some overhead from translating between 1D and 2D representations. This requires some arithmetic with `%` and `/` (the modulo and division operators). On the other hand, it seems the compiler is smart enough to index the 2D array efficiently. And using a 1D array doesn't get the improved performance of cache-hit rates that we saw from 1D vs 2D Vec. Probably because once you're using arrays, everything is stack-allocated.

It's also entirely possible that I've missed some subtlety of benchmarking, and the compiler is optimizing (or failing to optimize) something away. That's why artificial benchmarks are only part of the solution. Real-world results may vary! In fact, in the ray tracer where I use these grids, the only significant difference was that the 2D Vec implementation was slower than the rest. In my specific use-case, the release binary didn't show any improvements between 1D Array, 2D Array and 1D Vec. I ended up using 2D Array in my ray tracer, because the 1D-to-2D translation math was an unnecessary complication.


Thanks for reading! I hope you enjoyed learning about const generics, Nightly Rust and Criterion—I know I did. If you have any questions or suggestions, please let me know on [twitter](https://twitter.com/adam_chal) or via [email](mailto:adam.s.chalmers@gmail.com). The code is [on GitHub](https://github.com/adamchalmers/const_generic_grid) and if you have suggestions for improving it, feel free to open a PR. Thanks!

_Thanks to Nick Vollmar for feedback_
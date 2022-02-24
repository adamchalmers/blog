+++
title = "Parsing bitstreams with Nom"
date = 2022-01-05
description = "How to use Nom to parse binary protocols at the level of individual bits"

[taxonomies]
tags = ["rust", "programming", "nom", "parsing"]
+++

In our previous Nom adventure we [parsed text files](/nom-chars.md). But Nom can easily parse binary too! Programming languages generally only manipulate bytes (groups of 8 bits). It can be a bit tricky to manipulate single bits. But luckily, Nom has parsing primitives for reading individual bits. Then, using the same Nom parser combinators from the last post, you can combine those simple bitwise parsers into a more complex parser than parses some binary protocol.

<!-- more -->

Note that all code examples use Nom 7 -- I'll try to update this if Nom 8 makes breaking changes, but I can't guarantee anything :)

## Representing bitstream inputs

In the previous post, we saw that Nom parsers are generic over three types:

- `I`, the input
- `O`, the output
- `E`, the error type

In the last post, `I` was always `&str`, which lets you parse a stream of text. The input type `I = &[u8]` lets you parse a stream of bytes. But how can we represent a stream of _bits_? Rust doesn't have any type to represent a bit! 

This question is actually really important, and it's going to come up several times in this blog post.

Now, my first answer was "just use bools". After all, bits can be 0 or 1, and bools can be false or true. This works, you can definitely represent bits as bools. But it's a bit wasteful. Rust bools actually take up an entire byte. So you could represent a bitstream as `&[bool]`, but it would take 8x as many bytes as necessary! So we'd like to find a more efficient solution.

The solution to representing a bitstream is pretty clever: we use two values. Firstly, a list of bytes, and secondly, a number representing which exact bit of those bytes is next.

```rust
I = (&[u8], usize)
```
This pair represents a bitstream. The first element is a stream of bits, stored one byte at a time (so, in groups of 8 bits). The second element is an offset, showing which bit (not byte!) should be read next.

For example, let's say we want to parse a sequence of 16 bits like `1111000011001100`. We'll start by reading from the very first element. This is how I visualize the bitstream and offset:
```
1111000011001100
^
```
Nom represents this using `(&[u8], usize)` as:
```
([0b11110000, 0b11001100], 0)
    ^
```
Lets say you parsed 3 bits from there. After that, the bitstream would be

```
([0b11110000, 0b11001100], 3)
       ^
```
After reading another six bits, the input would have advanced past the first byte:

```
([0b11110000, 0b11001100], 9)
                 ^
```
Because the first byte will never be used again, Nom optimizes by dropping the first byte, adjusting the offset to account for that:

```
 ([0b11001100], 1)
      ^
```

Because this tuple type appears so often, I usually add a type alias for it:

```rust
type BitInput<'a> = (&'a [u8], usize);
```

## Parsing bitstreams with "take"

We just learned how Nom represents input types that can address individual bits. Now we can actually run parsers on that bitstream input. There's two main bit-level parsers: `take` and `tag`. 

The [`nom::bits::complete::take`][bittake] parser is similar to the [`nom::bytes::complete::take`][bytetake] parser from the previous post. It has a parameter, `count`, and it takes that number of bits from the input, then returns those `count` bits as the output. If there's not enough bits left in the input (i.e. the input's length in bits is < `count`) then it panics.

It's easy to understand what this parser does, but invoking it is a litle harder, because it has a lot of generics [^1] and Rust usually can't infer the types. So I've always just used helper functions that provide specific, concrete types. So, let's build a parser that parses four bits. A four-bit number is called a _nibble_ (because it's half a byte... get it... by engineering standards this qualifies as a "joke"). 

Obviously its `I` (generic input type) will be the standard `BitInput` alias. But what will the `O` (generic output type) be? After all, Rust doesn't have a u4 type. How do we store a 4 bit number? 

This question is really just the earlier question of "how can we represent bits in Rust" again. I told you it would come up several times in this blog post.

Well, Nom's `take` parser solves this by rounding up to the nearest 8 bits, and storing them in some uint type like `u8`, `u16` or whichever one you, the programmer, chooses. We'll parse the first 4 bits into a number between 0 and 15, and then just store that in a u8. This means the 4 most significant bits will always be zero. You can use any uint type, but a `u8` will do just fine, and wastes less RAM than using a `u16` or any larger type.

```rust
use nom::{bits::complete::take, IResult};
type BitInput<'a> = (&'a [u8], usize);

/// Take 4 bits from the BitInput.
/// Store the output in a u8, because there's no u4 type, and u8 is the closest-available size.
pub fn take_nibble(i: BitInput) -> IResult<BitInput, u8> {
    take(4usize)(i)
}

let input = ([0b1010_1111].as_ref(), 0);
let (_input, actual_nibble) = take_nibble(input).unwrap();
let expected_nibble = 0b1010;
assert_eq!(actual_nibble, expected_nibble);
```

## Parsing bitstreams with "tag"

The bitwise [`tag`][bittag] parser matches a specific pattern of bits, like "0110", from the input. It's a simple idea with just one tricky question: how do we represent a pattern of bits? As we discussed above, Rust's standard library doesn't have a type that can store an arbitrary number of bits efficiently. We could supply the bit pattern as `Vec<bool>` but that uses 8x the bits we need.

Yes, this is the _third_ time I've asked "how do we represent bits in Rust". I told you, it's a pretty fundamental question! And again, it uses a slightly different solution.

Nom represents the bit pattern using two parameters:
- **count**: how many bits long the pattern is
- **pattern**: The pattern itself, padded with leading zeroes to fit into some uint type.

For example:
- The pattern `101` is represented as `(pattern: 00000_101, count: 3)`. 
- The pattern `111000111` is represented as `(pattern: 0000000_111000111, count: 9)`. 

> Note that Rust number literals let you put underscores wherever you'd like, to enhance readability. E.g. you can help separate commas, by writing 1000000 as 1_000_000. I've used them here to separate the pattern part from the padded zeroes.

You, the programmer, will choose which uint types to use for `pattern` and `count` -- the parser is generic over various uint types. I personally would just use the smallest uint that fits the value. So, example 1's pattern fits in a `u8`, example 2's fits in a `u16`, and in both the count fits in a `u8`. So I'd just use those. Maybe there's some performance implications to this, I don't know. I trust the compiler to generate fast code, but I guess if this was on the hot path of my program, I would benchmark different choices of uint types.

OK, now that we know how to represent a pattern of bits, the `tag` parser is easy. You supply a pattern of bits, and Nom compares it, bit-by-bit, with the input bits. Like all Nom parsers, `tag` returns a Result, where the OK branch is pair (remaining input, matched). So, for example, parsing the pattern `101` on the bitstream `10100` will return (`00`, `101`). 

We're now ready to look at actual code:

```rust
use nom::{bits::complete::tag, IResult};

type BitInput<'a> = (&'a [u8], usize);

// This is just a simple wrapper around the `tag` parser, but it makes the parameter's types
// concrete instead of generic, so now Rust knows how to actually store the pattern.
fn parser(pattern: u8, count: u8, input: BitInput) -> IResult<BitInput, u8> {
    tag(pattern, num_bits_to_compare)(input)
}

// The pattern 1111 matches the stream 1111_1111
assert!(parser(0x0f, 4, (&[0xff], 0)).is_ok());
// The pattern 1 matches the stream too
assert!(parser(0x01, 1, (&[0xff], 0)).is_ok());
// The pattern 01 does _not_ match the stream
assert!(parser(0x01, 2, (&[0xff], 0)).is_err());
// The pattern 1111_1110 doesn't match the stream either
assert!(parser(0xfe, 8, (&[0xff], 0)).is_err());
```
## Bitstreams and combinators

Remember the whole idea of a parser combinator library is: start with a few small parsers, then combine them with combinator functions. We've seen two primitive parsers for bitstreams, `tag` and `take`. Now we can combine them with e.g. the [`map`][mapcomb] combinator from my last post:

```rust
use nom::{bits::complete::take, combinator::map, IResult};
type BitInput<'a> = (&'a [u8], usize);

/// Takes one bit from the input, returning true for 1 and false for 0.
fn take_bit(i: BitInput) -> IResult<BitInput, bool> {
    map(take(1usize), |bits: u8| bits > 0)(i)
}

let input = ([0b10101010].as_ref(), 0);
let (input, first_bit) = take_bit(input).unwrap();
assert!(first_bit); // First bit is 1
let (_input, second_bit) = take_bit(input).unwrap();
assert!(!second_bit); // Second bit is 0
```

## Converting bytestreams to bitstreams and back

So far we've learned how to make simple bit parsers, and combine them into complex ones. We've even learned how Nom represents bitstreams. The last question is: where do these freaking bitstreams come from, anyway? After all, most Rust functions represent binary data in bytes (e.g. as `Vec<u8>` or using the [bytes crate](https://docs.rs/bytes/latest/bytes/)). If you're reading binary data from disk, or RAM, or the network, it's almost definitely going to be stored in bytes. So we need a way to turn a bytestream into a bitstream. 

Luckily the function [`nom::bits::bits`](https://docs.rs/nom/latest/nom/bits/fn.bits.html) does exactly that. The docs say it "converts a byte-level input to a bit-level input, for consumption by a parser that uses bits." Perfect!

Again, this function uses a lot of generics which can be confusing, so here's an example showing how it works.

```rust

use nom::IResult;
use nom::multi::many0;
use nom::number::complete::be_u16;

type BitInput<'a> = (&'a [u8], usize);

/// Stub example type. Imagine this has to be parsed from individual bits.
struct BitwiseHeader;

/// A bit-level parser
pub fn parse_header(i: BitInput) -> IResult<BitInput, BitwiseHeader> {
    todo!()
}

/// Stub example type. 
/// The header has to be parsed from bits, but the body can be parsed from bytes.
struct Message {
    header: BitwiseHeader,
    body: Vec<u16>,
}

/// A byte-level parser that calls a bit-level parser
pub fn parse_msg(i: &[u8]) -> IResult<&[u8], Message> {
    /// The header has to be parsed from bits
    let (i, header) = nom::bits::bits(parse_header)(i)?;
    /// But the rest of the message can be parsed from bytes.
    let (i, body) = many0(be_u16)(i)?;
    Ok((i, Message { header, body }))
}
```

I used something very similar to parse DNS messages in my [DNS client](dingoheader) if you want to see a real-world example. I'll probably write up my DNS client as a 

[^1]: Take a look at its [type signature](https://docs.rs/nom/7.1.0/nom/bits/complete/fn.take.html). The generics are, roughly:
- `input: I` is anything that works like a binary stream
- `count: C` is any uint type (with [some caveats][tousize])
- `output: O` is any uint type, but with different caveats
- And of course the usual generic `E` for verbose parser errors or the cheaper but less helpful default parser errors.

[bittake]: https://docs.rs/nom/7.1.0/nom/bits/complete/fn.take.html
[bytetake]: https://docs.rs/nom/7.1.0/nom/bytes/complete/fn.take.html
[bittag]: https://docs.rs/nom/7.1.0/nom/bits/complete/fn.tag.html
[tousize]: https://docs.rs/nom/7.1.0/nom/trait.ToUsize.html
[dingoheader]: https://github.com/adamchalmers/dingo/blob/d1a34e37b2c743dcd63bfe8612fd1d7c63ce9d63/src/message.rs#L239
[mapcomb]: https://docs.rs/nom/7.1.0/nom/combinator/fn.map.html

+++
title = "Parsing DNS headers with Nom"
date = 2022-03-11
description = "A real-world example"

[taxonomies]
tags = ["rust", "programming", "nom", "parsing", "binary", "dns"]
+++

In the last blog post, we learned how to [parse binary, bit-by-bit][bitnom] with Nom. I really wanted to give a real-world example of binary protocols, so here's one from [Dingo][dingo], a basic DNS client I made.

<!-- more -->

# Anatomy of a DNS header

DNS clients (like dig or Dingo) and DNS servers both use the DNS message protocol, which is defined in [RFC 1035][rfc1035]. This means that requests from clients and responses from servers will both conform to the same schema (although, some optional fields will only be present in requests, or in responses).

The RFC defines the Header in [section 4.1.1][sec411], and they include this helpful diagram showing each field, and how many bits long it is:

```
                                1  1  1  1  1  1
  0  1  2  3  4  5  6  7  8  9  0  1  2  3  4  5
+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
|                      ID                       |
+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
|QR|   Opcode  |AA|TC|RD|RA|   Z    |   RCODE   |
+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
|                    QDCOUNT                    |
+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
|                    ANCOUNT                    |
+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
|                    NSCOUNT                    |
+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
|                    ARCOUNT                    |
+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
```

Nice! They manage to fit all that data into just 12 bytes. I represented DNS headers as a Rust struct:

```rust

/// All DNS messages start with a Header (both queries and responses!)
/// Structure is defined at <https://datatracker.ietf.org/doc/html/rfc1035#section-4.1.1>
#[derive(Debug)]
pub struct Header {
    /// A 16 bit identifier assigned by the program that generates any kind of
    /// query.  This identifier is copied the corresponding reply and can be used
    /// by the requester to match up replies to outstanding queries.
    pub id: u16,
    /// A one bit field that specifies whether this message is a query (0), or a
    /// response (1).
    is_query: bool,
    /// A four bit field that specifies kind of query in this message. This 
    ///value is set by the originator of a query and copied into the response.
    opcode: Opcode,
    /// This bit is valid in responses, and specifies that the responding name 
    ///server is an authority for the domain name in question section. Note that
    /// the contents of the answer section may have multiple owner names because
    ///of aliases. The AA bit corresponds to the name which matches the query
    /// name, or the first owner name in the answer section.
    authoritative_answer: bool,
    /// Specifies that this message was truncated due to length greater than 
    ///that permitted on the transmission channel.
    truncation: bool,
    /// This bit may be set in a query and is copied into the response.  If RD 
    /// is set, it directs the name server to pursue the query recursively. 
    /// Recursive query support is optional.
    recursion_desired: bool,
    /// This be (sic) is set or cleared in a response, and denotes whether 
    /// recursive query support is available in the name server.
    recursion_available: bool,
    pub resp_code: ResponseCode,
    /// Number of entries in the question section.
    pub question_count: u16,
    /// Number of resource records in the answer section.
    pub answer_count: u16,
    /// Number of name server resource records in the authority records section.
    pub name_server_count: u16,
    /// Number of resource records in the additional records section.
    pub additional_records_count: u16,
}
```

# Parsing the header

Some of the fields' sizes are multiples of 8, so it's easy to parse them with the parsers in `nom::bytes`. DNS always encodes numbers with the most significant bit first, so we could just parse the 16-bit ID field with [`nom::bytes::complete::be_u16`][be_u16]. 

Other fields are a little trickier. The Opcode and RCODE fields are defined as 4-bit integers, and the flags like "AA" or "TC" are single-bit flags! But if you read my [article about bit-wise Nom parsers][bitnom], it shouldn't be too hard to parse these. Just use the functions from `nom::bits`!

For example, to parse the four flag fields (which are each one bit long), we'll define a helper function:

```rust
// See my blog about parsing bit-streams if this type confuses you!
type BitInput<'a> = (&'a [u8], usize);

/// Takes one bit from the BitInput.
pub fn take_bit(i: BitInput) -> IResult<BitInput, bool> {
    let (i, bit): (BitInput, u8) = take(1u8)(i)?;
    Ok((i, bit != 0))
}
```

Some of the fields (like "opcode") are actually enums, where each number value corresponds to some num. For example, the spec defines the opcode field as "A four bit field that specifies kind of query in this message." I encoded this as a Rust enum that can be converted from 4-bit numbers.

```rust
/// A four bit field that specifies kind of query in this message.
/// This value is set by the originator of a query and copied into the response.
#[derive(Debug)]
enum Opcode {
    /// 0: a standard query (QUERY)
    Query,
    /// 1: an inverse query (IQUERY)
    InverseQuery,
    /// 2: a server status request (STATUS)
    Status,
}

impl TryFrom<u8> for Opcode {
    type Error = anyhow::Error;

    fn try_from(value: u8) -> Result<Self, Self::Error> {
        let op = match value {
            0 => Self::Query,
            1 => Self::InverseQuery,
            2 => Self::Status,
            other => anyhow::bail!("Unknown opcode {other}"),
        };
        Ok(op)
    }
}
```

And, of course, we need to parse 4-bit numbers from bit-streams:

```rust
/// A "nibble" is half a "byte", i.e. a 4-bit number.
pub fn take_nibble(i: BitInput) -> IResult<BitInput, u8> {
    take(4u8)(i)
}
```

Then we can easily parse the opcode by parsing the 4-bit number, and trying to convert it into the `Opcode` enum.

```rust
let (i, opcode) = map_res(take_nibble, Opcode::try_from)(i)?;
```

## Full parser

Once you know the size of each field, and you have a struct to represent them all, it's actually pretty easy to parse the protocol.

```rust
impl Header {
    pub fn deserialize(i: BitInput) -> IResult<BitInput, Self> {
        use nom::combinator::map_res;
        let (i, id) = take_u16(i)?;
        let (i, qr) = take_bit(i)?;
        let (i, opcode) = map_res(take_nibble, Opcode::try_from)(i)?;
        let (i, aa) = take_bit(i)?;
        let (i, tc) = take_bit(i)?;
        let (i, rd) = take_bit(i)?;
        let (mut i, ra) = take_bit(i)?;
        // The spec defines the Z field as three consecutive 0s.
        for _ in 0..3 {
            let z;
            (i, z) = take_bit(i)?;
            assert!(!z);
        }
        let (i, rcode) = map_res(take_nibble, ResponseCode::try_from)(i)?;
        let (i, qdcount) = take_u16(i)?;
        let (i, ancount) = take_u16(i)?;
        let (i, nscount) = take_u16(i)?;
        let (i, arcount) = take_u16(i)?;
        let header = Header {
            id,
            is_query: qr,
            opcode,
            authoritative_answer: aa,
            truncation: tc,
            recursion_desired: rd,
            recursion_available: ra,
            resp_code: rcode,
            question_count: qdcount,
            answer_count: ancount,
            name_server_count: nscount,
            additional_records_count: arcount,
        };
        Ok((i, header))
    }
}
```

## Alternatives to Nom

The people on the [Rust subreddit][rustreddit] had some good points about my [last blog post][bitnom]. My [favourite comment][scottcomment] from [Scott Lamb][scottlamb] basically said that Nom was overkill for parsing DNS headers. He's right -- as you just saw, DNS headers don't need a parser combinator framework. The `Header::deserialize` method above doesn't even use any combinators! You could use a simpler bitstream crate, like [deku] or [bitstream-io]. 

Ultimately, for my use case (building a [DNS client][dingo]), Nom made sense. DNS headers are easy to parse, but the rest of the DNS message format is a little bit more dynamic, with variable-length fields, optional fields, backtracking, etc. So, although this simple example doesn't really use Nom's combinators, the full DNS message parser sure does. But if you only need to parse DNS headers, then yes, you should consider using a simpler crate.

And if you need to both serialize and deserialize a binary format, consider using [deku], which lets you just write one Serde-like struct declaration which can handle both. Just be warned that it imports Serde and all the proc-macro/syn/quote crates, which will take much longer to compile than plain old Nom.

You can see the full parser code [on GitHub](https://github.com/adamchalmers/dingo/blob/83e9e62d4740f44911ca968a1d79c4ae38467f7b/src/message/header.rs#L79). I hope this real-world example helps you understand Nom bitwise parsers a bit better. In my next blog post, I'll talk about some other aspects of my DNS client, and what I learned from making it.

[scottcomment]: https://www.reddit.com/r/rust/comments/t52iuz/comment/hz31599/?utm_source=reddit&utm_medium=web2x&context=3
[deku]: https://docs.rs/deku
[bitstream-io]: https://docs.rs/bitstream-io
[scottlamb]: https://github.com/scottlamb
[rustreddit]: https://reddit.com/r/rust
[bitnom]: /nom-bits
[dingo]: https://github.com/adamchalmers/dingo
[rfc1035]: https://datatracker.ietf.org/doc/html/rfc1035
[sec411]: https://datatracker.ietf.org/doc/html/rfc1035#section-4.1.1
[be_u16]: https://docs.rs/nom/latest/nom/number/complete/fn.be_u16.html
+++
title = "Parsing DNS headers with Nom"
date = 2022-02-06
description = "A real-world example"

[taxonomies]
tags = ["rust", "programming", "nom", "parsing", "binary", "dns"]
+++

In the last blog post, we learned how to [parse binary, bit-by-bit][bitnom] with Nom. I really wanted to give a real-world example of binary protocols, so here's one from [Dingo][dingo], a basic DNS client I made.

<!-- more -->

# Anatomy of a DNS header

DNS clients (like dig or Dingo) and DNS servers both use the DNS message protocol, which is defined in [RFC 1035][rfc1035]. This means that requests from clients and responses from servers will both conform to the same schema (although, some optional fields will only be present in requests, or in responses).

The RFC defines the Header in [section 4.1.1][sec411]:

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

You can read the RFC for the definition of each field. Some of these fields can be easily parsed using `nom::bytes`, e.g. the ID field is "a 16 bit identifier". DNS always encodes numbers with the most significant bit first, so we can just use [`be_u16`](be_u16) to parse it. 

However others are a little trickier. The Opcode and RCODE fields are defined as 4-bit integers, and the flags like "AA" or "TC" are single-bit flags! But if you read my [article about bit-wise Nom parsers][bitnom], it shouldn't be too hard to parse these.

[bitnom]: /nom-bits
[dingo]: https://github.com/adamchalmers/dingo
[rfc1035]: https://datatracker.ietf.org/doc/html/rfc1035
[sec411]: https://datatracker.ietf.org/doc/html/rfc1035#section-4.1.1
[be_u16]: https://docs.rs/nom/latest/nom/number/complete/fn.be_u16.html
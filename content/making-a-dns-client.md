+++
title = "What I learned from making a DNS client in Rust"
date = 2022-03-11
description = "A fun project that taught me a lot"
draft = true

[taxonomies]
tags = ["rust", "programming", "nom", "parsing", "binary", "dns", "bitvec"]
+++

Over the last few weeks I built my own DNS client. Mostly because I thought dig (the standard DNS client) was kinda clunky. Partly because I wanted to learn more about DNS. So here's how I built it, and how you can build your own too. It's a great weekend project, and I learned a lot from finishing it.

<!-- more -->

# Why?

[Julia Evans][b0rk] gave me the idea of making a DNS client. She's one of my favourite tech bloggers. Her blog always teaches me something, and often inspires me to go learn something new. She's also really good at summing up complex topics into really [simple little cartoons][wizardzines], like this one:

[![Comic explaining DNS query structure, when serialized for transmission over wire](/making-a-dns-client/julia-evans-dns-packet.png)][wizardzines_dns_packet]

When I read that comic, I was shocked. The DNS query protocol is nowhere near as complicated as I thought it would be. Also, I work at a company that is, uh, kind of a big deal in the DNS world. I should probably understand it better.

# The Plan

The other big reason is that I knew some really great Rust libraries that would simplify every step:

 - Parse the CLI args using [picoargs]
   - It's not as powerful as [clap], the standard Rust CLI crate, and it requires some more boilerplate, but it takes _way_ less time to compile, and I figured this would only use a very simple CLI.
 - Serialize the DNS query using [bitvec], an _awesome_ general-purpose crate for reading or writing individual bits.
 - Communicate with a DNS resolver with the stdlib [UdpSocket] type. 
   - I had no idea how this worked, but I know the Rust stdlib is really well-documented, so I didn't think it'd be too hard.
 - Parse the binary response with [Nom]. 
   - I learned how to parse bitwise protocols with Nom when doing [Advent of Code][aoc16]
   - See my [previous blog post][bitnom] for more about this!
 - Use plain old `println!` to print the response to the user.  

# How did I go?

It took around 800 lines of code, and I had it almost finished in a weekend. There was just one part of the spec I hadn't implemented: message compression. Unfortunately, the DNS server I was querying used message compression, so I really had to implement it, which took me another weekend.

I named it Dingo because it sounded like `dig`, and it reminds me of Australia, my home. Anyway, it works!

![Screenshot of a terminal running dingo, my DNS client, and resolving a name](/making-a-dns-client/dingo_screenshot.png)

You can install it or read the finished code [on GitHub][dingo]. So. What did I learn from all this?

# What did I learn?

## Reading RFCs

## Sockets

## Bitvec

## Dig's weird output

## I still fucking love enums


[clap]: https://crates.io/crates/clap
[Nom]: https://docs.rs/nom
[aoc16]: https://adventofcode.com/2021/day/16
[wizardzines]: https://wizardzines.com
[picoargs]: https://crates.io/crates/picoargs
[bitvec]: https://docs.rs/bitvec
[UdpSocket]: https://doc.rust-lang.org/stable/std/net/struct.UdpSocket.html
[wizardzines_dns_packet]: https://wizardzines.com/comics/dns-packet/
[b0rk]: https://jvns.ca/
[bitnom]: /nom-bits
[dingo]: https://github.com/adamchalmers/dingo
[rfc1035]: https://datatracker.ietf.org/doc/html/rfc1035
[sec411]: https://datatracker.ietf.org/doc/html/rfc1035#section-4.1.1
[be_u16]: https://docs.rs/nom/latest/nom/number/complete/fn.be_u16.html
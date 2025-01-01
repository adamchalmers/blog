+++
title = "EuroRust 2024 talk: Code as contract as code"
date = 2025-01-01
description = "How to automate your API servers, clients and schemae"

[taxonomies]
tags = ["rust", "programming", "video", "eurorust", "servers", "openapi"]
+++

A few months ago I was invited to EuroRust to talk about API servers and clients. When I worked at Cloudflare, I maintained a few API servers and the official Cloudflare Rust API client. There was a lot of toil and stress involved whenever the API servers changed: schemae had to be updated, and then so did clients.

When I joined Zoo I was very impressed to find that our CEO Jess Frazelle had already set up automatic schema generation from the API servers, and then automatic client generation from those schemae (in 4 different languages)! This made managing our API servers at Zoo MUCH easier than it was at Cloudflare.

So I wrote this conference talk aimed at my past self -- it's basically a guide on how to simplify your API operations.

<iframe width="560" height="315" src="https://www.youtube.com/embed/bjgGboWCTDw?si=OeGk9XbHN5JK8Vdh" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

<!-- more -->

I was very pleasantly surprised that a lot of people wanted to chat about this after I gave my talk -- I guess I'm not the only one who's struggled with the various parts of an API in sync. I particularly enjoyed chatting with [Rob Ede] of the [Actix Web crate], who took an opposite view from me (one should generate the server code from the schema, not the schema from the server code). I was excited for a big ideological showdown and passionate debate but unfortunately we both pretty quickly understood where the other was coming from, the differences in our respective workplaces and why each person's preference made sense for their particular environment. Like so many questions in life and software it boils down to tradeoffs. Sigh. Maybe one day I'll have a proper yelling match and start a serious beef at a conference. Until then, I can only dream.


Thanks for watching!

[Rob Ede]: https://robjtede.dev/
[Actix Web crate]: https://actix.rs/


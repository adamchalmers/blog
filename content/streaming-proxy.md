+++
title = "Static streams for faster async proxies"
date = 2022-08-26
description = "The borrow checker is a tough negotiating partner"
draft = false

[taxonomies]
tags = ["rust", "programming", "web", "streams", "axum", "multipart"]
+++

Last week in [Tokio's Discord chat server][tokiodiscord] someone asked a really interesting question about streaming proxies. How can you stream a multipart body from an incoming request into an outgoing request? My struggle to answer this really helped me understand Rust async lifetimes. We'll go _why_ this is tricky, then design and benchmark a solution. 

<!-- more -->

Note: all the code is in a full [GitHub example][ghexample].

> I'm rather new to Rust, so perhaps I'm trying to bite off more than I can chew, but I can't find any way to stream an `axum::extract::multipart::Field` to the body of a PUT request made with reqwest. Obviously, waiting for all the bytes works just fine, as in this test handler:

```rust
use axum::{extract, Extension};
use reqwest::StatusCode;

/// An Axum request handler. 
async fn upload(
    // The request body should be a Multipart.
    mut multipart: extract::Multipart,
    // The server should set a reqwest::Client in the extensions.
    Extension(client): Extension<reqwest::Client>,
// If this function succeeds, return HTTP 200 and the Ok string as the body.
// If it fails, return the given status code and the Err string as the body.
) -> Result<String, (StatusCode, String)> {

    // Get the first field of the multipart request.
    if let Some(field) = multipart.next_field().await.unwrap() {
        client
            .put(url::Url::parse("https://example.com/file").unwrap())
            .body(field.bytes().await.unwrap())
            .send()
            .await
            .map_err(|_| (StatusCode::INTERNAL_SERVER_ERROR, "Oops".to_string()))
            .map(|_| "Yay".to_string())
    } else {
        Err((StatusCode::INTERNAL_SERVER_ERROR, "Oh no".to_string()))
    }
}
```

> Everything I've found for using a stream as the request body requires that the stream be 'static, which of course the fields I'm getting from multipart aren't.

The problem sounded very simple -- the server should stream a request in, and stream a request out -- so I figured I'd help this person out, for two reasons:

1. It's nice to help people and I am Very Nice
2. My dayjob is writing a streaming HTTP proxy in Rust, so this would be good practice.

But it turned out to be harder than I thought. 

# Why streaming instead of buffering?

Why not just buffer the whole body into memory? Is streaming really that important? Well, yeah -- if you're building a proxy, streaming is worth the extra headache. Streaming beats buffering for two reasons:

First, **latency** is worse with buffering. Your proxy has to buffer everything from the client before sending everything to the server. This _doubles_ the latency of requests (assuming all three hosts are equidistant). You have to wait _n_ seconds for the request to reach the proxy, then _n_ seconds to transmit it to the server. But if your proxy used streams, you could get the first few bytes from the client, and send them to the server while you waited for the next few bytes from the client.

Second, **memory** overhead is _huge_ with buffering. Say you're proxying a 10gb MacOS system update file. If you buffer every request into memory, your server can't handle many parallel requests (probably <10). But with streaming, you can get the first _n_ bytes the client, proxy them to the server, and then free them. No more crashing the process because it ran out of memory.

So we _really_ want our proxy server to stream requests. There's just one problem. Our HTTP server and HTTP client seem to have contradictory lifetimes.

# The problem

Axum's [`Multipart` extractor][mpx] owns the incoming request body. It has a method [`next_field`][next_field] that returns a [`Field`][field] type. That `Field` is a view into the data from the main `Multipart` body. It just borrows part of the multipart data. This means the Field is actually `Field<'a>` where `'a` is the lifetime of the `Multipart` that created it.

![Field borrows from Multipart](/streaming-proxy/field_borrow.jpg)

Clearly the field can't outlive the Multipart, because then Field would be pointing at data that is no longer there. Rust borrow checker stops us from doing that. So far so good.

You can read data from a stream, either by reading the whole thing into memory (as above), or using the Stream trait. This trait is defined by the Futures crate [here](https://docs.rs/futures/latest/futures/prelude/trait.Stream.html) and implemented [here](https://docs.rs/axum/latest/axum/extract/multipart/struct.Field.html#impl-Stream-for-Field%3C%27a%3E). Streams are basically asynchronous iterators. When you try to get the next value from them, it might not be ready yet, so you have to `.await`. This is perfect for reading HTTP request bodies, because you can start processing the body as it comes in piece-by-piece. You don't have to wait for the entire body to be available. This saves you time and maybe RAM, if you don't actually need to store the entire body.

So, now we know how to stream data _out_ of the body. What about streaming data _into_ a new request's body?

We'll use [reqwest](docs.rs/reqwest) as the HTTP client. Reqwest can send various things as HTTP bodies -- strings, vectors of bytes, and even streams, using the [`wrap_stream`][wrap_stream] method. The problem is, the stream has to be `'static`, a special lifetime which means "either it's not borrowed, or it's borrowed for the entire length of the program". This means we can't use `Field` as a reqwest body, because it's not `'static`. It's borrowed, and it's only valid for as long as the `Multipart` that owns it.

Why does reqwest only allow static streams to be bodies? I asked its creator [Sean McArthur][sean].

> Sean: Internally the connection is in a separate task, to manage connection-level stuff (keep alive, if http2: pings, settings, and flow control). The body gets sent to that connection task
> Adam: So basically the connection task takes ownership of all request bodies? And therefore bodies have to be 'static because they need to be moved into that task? 
> Sean: Exactly

But there _is_ a solution. The field type is `Field<'a>`, so it's generic over various possible lifetimes. We just need to make sure that the _specific_ lifetime our server uses is `'static`.

# The solution

The key idea here is that the `Multipart` object itself is `'static`. Why? Because it's not borrowed. The `next_field` type signature is `next_field(&mut self) -> Field<'_>`. What's that `'_`? That's an _elided lifetime_. Basically these two signatures are equivalent:

```rust
fn next_field(&'a mut self) -> Field<'a>
fn next_field(&mut self) -> Field<'_>
```

So, the field type is `Field<'a>`, but that `'a` is generic. Its definition works for any possible lifetime. We just have to make sure that when our server's handler is compiled, it knows that `'a` is `'static`. 

The key insight is that, if you want to use a stream as a reqwest body, it can't borrow anything. Because if it borrowed data from some other value on some other thread, what happens if that thread dies? Your body would keep reading from the freed memory, and Rust won't let that happen. This means _the stream has to own all the data being streamed_. 

So, the stream needs to own the `Multipart`! After all, the multipart owns the data, and the stream has to own the data. So the stream has to own the multipart.

Once I realized that, the solution emerged. We'll define a new type of Stream that owns a Multipart and streams data out of its fields.

```rust
use axum::{body::Bytes, extract, extract::multipart::MultipartError};
use futures::prelude::Stream;

/// Wrapper for axum's Multipart type. 
struct MultipartStream(extract::Multipart);

impl MultipartStream {
    /// Stream every byte from every field.
    fn into_stream(mut self) -> impl Stream<Item = Result<Bytes, MultipartError>> {
        async_stream::stream! {
            while let Some(field) = self.0.next_field().await.unwrap() {
                for await value in field {
                    yield value;
                }
            }
        }
    }
}
```

We make a [newtype] for converting a Multipart into a Stream. I'm Very Creative so I chose the imaginative name MultipartStream. It owns the Multipart, and its `into_stream` method consumes (aka takes ownership of) `self`, so that method also owns the Multipart. This means the Multipart is `'static`. The method creates a stream, using the `stream!` macro from the [async-stream] crate. And that stream then takes ownership of `self` and therefore the Multipart.

All this means that the stream doesn't borrow anything. It owns all its data -- both the multipart and its fields -- so the stream is `'static`. Now you can pass it to `reqweset::Body::wrap_stream`. 

Note: the `into_stream` example is very rudimentary -- in a real server you might want to only stream certain fields, or maybe filter out fields, or maybe only stream the _nth_ field. 

The last thing to do is actually _use_ this `MultipartStream` wrapper in our endpoint. 

```rust
/// State for the proxy server.
/// See https://docs.rs/axum/0.6.0-rc.1/axum/extract/struct.State.html
#[derive(Clone)]
struct ProxyState {
    dst_port: u16,
    client: reqwest::Client,
}

async fn proxy_upload_streaming(
    // the State parameter can be destructed right here in the function signature, so let's
    // destructure it and unpack its two fields.
    State(ProxyState { dst_port, client }): State<ProxyState>,
    // The incoming request's multipart body.
    incoming_body: extract::Multipart,
) -> Result<String, (StatusCode, String)> {

    // Stream the incoming request's body, into the outgoing request's body.
    let stream = MultipartStream(incoming_body).into_stream();
    let outgoing_body = reqwest::Body::wrap_stream(stream);

    // Send the outgoing request
    client
        .post(url::Url::parse(&format!("http://127.0.0.1:{dst_port}/")).unwrap())
        .body(outgoing_body)
        .send()
        .await
        .map_err(|e| (StatusCode::INTERNAL_SERVER_ERROR, e.to_string()))
        .map(|_| format!("All OK\n"))
}
```

Note, this example uses Axum 0.6.0-rc.1 with its new [State][state] types. It's possible that it might change a little before the final 0.6 release. See [the announcement][axum06rc] for more. State is basically like an Axum extension where the compiler can guarantee it's always set. This perfectly solves the problem my [previous post about Axum](/what-are-extensions) complained about, where _I_ know the extension is always set, but the compiler doesn't, so I need a dubious `.unwrap()`

# Benchmarks?

I ran some benchmarks on the repo. Basically I used `curl` to send the proxy server a Multipart body with 20 copies of the Unix wordlist. Then the server proxied it to a second server, which prints it. I ran it with a streaming proxy and a buffered proxy. You can see the full setup in the [GitHub example][ghexample]. 

Because I ran this all locally, I don't expect much difference in total time. After all, the latency between processes running on my Macbook is pretty low. So doubling the latency won't matter much, because the latency is nearly zero anyway. But I expect the RAM usage to be very different.

|           | Time (seconds)  | RAM (mb) |
| --------- | --------------- | -------- |
| Streaming | 0.10            | 16       |
| Buffering | 0.32            | 128      |

Yep, that checks out.

# Takeaways

* reqwest connections are handled in their own thread, so they need to own their bodies.
 * This is just a design choice -- other HTTP libraries could work differently, although [tide] and [actix web client][awc] also require streaming bodies be `'static`. 
 * I think this is partly because of the Tokio runtime, and other runtimes might not require 'static, see [this talk][glommiotalk] from the creator of [glommio].
* `'static` means "either isn't borrowed, or is borrowed for the entire runtime of the program"
* A static stream can't borrow any data
* Implementing your own streams with [async-stream] is pretty easy

---

[async-stream]: https://docs.rs/async-stream/latest/async_stream/
[awc]: https://docs.rs/awc
[axum06rc]: https://tokio.rs/blog/2022-08-whats-new-in-axum-0-6-0-rc1
[field]: https://docs.rs/axum/latest/axum/extract/multipart/struct.Field.html
[ghexample]: https://github.com/adamchalmers/axum-reqwest
[glommio]: https://docs.rs/glommio/latest/glommio/
[glommiotalk]: https://www.youtube.com/watch?v=PbgTyCSDPrs
[mpx]: https://docs.rs/axum/latest/axum/extract/struct.Multipart.html
[newtype]: https://doc.rust-lang.org/rust-by-example/generics/new_types.html
[next_field]: https://docs.rs/axum/latest/axum/extract/struct.Multipart.html#method.next_field
[sean]: https://twitter.com/seanmonstar
[state]: https://docs.rs/axum/0.6.0-rc.1/axum/extract/struct.State.html
[tide]: https://docs.rs/tide
[tokiodiscord]: https://discord.com/invite/tokio
[wrap_stream]: https://docs.rs/reqwest/latest/reqwest/struct.Body.html#method.wrap_stream
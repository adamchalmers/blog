+++
title = "Introduction to HTTP Multipart"
date = 2023-04-25
description = "Why use it? How does it work?"
draft = false

[taxonomies]
tags = ["http"]
+++

Multipart, or "form-encoded data", is something I see everywhere but never had to actually understand or use myself, because HTTP libraries handled it for me. Recently, though, I had to dive deeper into how multipart works, because it's pretty important at both Cloudflare and my new job at KittyCAD. Used correctly, multipart makes your file uploads faster and use less memory. I'm going to explain how it works, at a high level, so you'll recognize when you could use it to save time and memory in your HTTP servers/clients.

<!-- more -->

By the way, this doc assumes you know what a MIME type is. If you don't know what a MIME type is, read this [MDN article](https://developer.mozilla.org/en-US/docs/Web/HTTP/Basics_of_HTTP/MIME_types) for a great explanation.

## Why use multipart?

Multipart solves the problem of efficient file upload. Before Multipart, the standard for uploading files was "application/x-www-form-urlencoded", which required clients to URL-encode the file before uploading it. URL-encoding is efficient if the file is mostly ASCII text, but if it's mostly binary data, then you have to URL-escape almost every byte, which is very inefficient. If you wanted to upload multiple files _without_ encoding them, you could send multiple HTTP requests. But that has more latency than sending them all in one request.

So in 1998, [RFC 2388] proposed a new standard, "multipart/form-data", which lets you send many files in one HTTP body without encoding them. No encoding means you save a lot of CPU cycles and keeps the total body size small.

This protocol was designed for uploading files from an HTML form, hence the name. But you can actually use it to upload files from whatever you want -- no part of the spec requires `<form>` or any HTML at all. You can use it to upload files from any HTTP client to any HTTP server.

Another advantage of multipart is that the server can stream each part separately. For example, say you're uploading 5 files by encoding them into a JSON object. Your server will have to buffer the entire JSON object into memory, decode it, and examine each file. But with multipart, the server can stream each part (i.e. file), one at a time, reducing memory usage and improving latency (because it can handle file number 1 without waiting for the other 4 to come in).

(oh, and JSON also can't handle binary files, so you'd need to convert each file into base64 strings. But as discussed above, multipart can handle raw binary, so that's another win)

## In Rust
Your Rust HTTP server or client framework probably already supports multipart. I wrote about [streaming multipart in axum previously][static-stream]. [reqwest](https://lib.rs/reqwest) has [built-in support](https://docs.rs/reqwest/latest/reqwest/struct.RequestBuilder.html#method.multipart) too.

If you're building your own HTTP server, I suggest using the [multer crate](https://lib.rs/multer) for async multipart support. This is the crate that axum uses, so it should be very fast and reliable. Reusing crates from the ecosystem means you don't even need to know how multipart is implemented. But if you're curious, keep reading.

## What _is_ multipart?

MIME types fall into two classes, discrete and multipart. 
 * Discrete hold one document. Examples include `application/` (binary), `image/`, `text/` etc. 
 * Multipart types are documents with many pieces, and those pieces can have their own MIME types. 

There are two multipart types: message/ and multipart/ -- yes it's confusing that multipart can be a type and also a class. The `message/` type is basically never used for anything anymore, but `multipart/` is still very important. You often see `multipart/form-data` for sending files from web browser to server via HTML forms. The *part* in *multipart* refers to a document. The type could have been called multidocument instead! Note it does not _have_ to contain multiple files. It could contain just one file, using multipart for the efficient binary encoding.

## How is multipart implemented?

If the content-type is `multipart/form-data` then the HTTP body contains multiple parts (i.e. documents). Each part is separated by a "boundary delimiter". The root HTTP message has a header which [defines the boundary](https://stackoverflow.com/questions/3508338/what-is-the-boundary-in-multipart-form-data) delimiter so the server knows where the boundary between each part is. Each part has some headers too:
* The [Content-Disposition header](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Disposition#as_a_header_for_a_multipart_body) defines each part's filename or the name of the form field that contained it (only relevant if you're using an actual HTML form element). 
* The [Content-Type header](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Type) defines each part's filetype (OK technically their MIME type but that's roughly equivalent). It defaults to text/plain. Unstructured binary data should use application/octet-stream, but if you know the type you should use it, e.g. application/zip,  application/pdf, etc.
* No other headers can be used! Quoting [RFC 7578]: "The multipart/form-data media type does not support any MIME header fields in parts other than Content-Type, Content-Disposition, and (in limited circumstances) Content-Transfer-Encoding. **Other header fields MUST NOT be included and MUST be ignored.**".

Here's a practical example from of [how the HTTP body looks](https://stackoverflow.com/questions/913626/what-should-a-multipart-http-request-with-multiple-files-look-like), from Stack Overflow. This body is a multipart which contains 3 GIFs.

```
POST /cgi-bin/qtest HTTP/1.1
Content-Type: multipart/form-data; boundary=2a8ae6ad-f4ad-4d9a-a92c-6d217011fe0f
Content-Length: 514

--2a8ae6ad-f4ad-4d9a-a92c-6d217011fe0f
Content-Disposition: form-data; name="datafile1"; filename="r.gif"
Content-Type: image/gif

GIF87a.............,...........D..;
--2a8ae6ad-f4ad-4d9a-a92c-6d217011fe0f
Content-Disposition: form-data; name="datafile2"; filename="g.gif"
Content-Type: image/gif

GIF87a.............,...........D..;
--2a8ae6ad-f4ad-4d9a-a92c-6d217011fe0f
Content-Disposition: form-data; name="datafile3"; filename="b.gif"
Content-Type: image/gif

GIF87a.............,...........D..;
--2a8ae6ad-f4ad-4d9a-a92c-6d217011fe0f--
```

## Compression

You can gzip the entire Multipart response, but you cannot pick and choose compression for particular parts. This is because the root HTTP body defines compression headers for the entire message, including all the parts within a multipart body. So a client has no way to tell the server "this particular part is compressed, but that one is not." And, as discussed above, only 3 specific HTTP headers are allowed inside the multipart's documents -- and compression headers aren't one of them. Thanks to [this great Stack Overflow answer](https://stackoverflow.com/a/66118265/531650) for explaining all this to me.

## Why is this interesting?

So, "multipart" or "form-encoded data" is a MIME type which contains multiple files. Each file has its own MIME type and name. Historically, this was a big improvement over other ways to upload multiple files, because it can send each file as raw binary without extra encoding or escapes.

Before I wrote this blog post, I found multipart kinda boring. It somehow seems archaic and backwards -- I mean, it was written at a time where HTML forms were the cutting edge of web technology. It's been 25 years since its RFC was first published. Surely we have better ways to upload files now!

But it's actually pretty interesting, in an abstract way. It's trying to efficiently compose multiple file uploads together, and the question of "how do we compose many of this together" is always an interesting question in computer science. 

Part of why I love JSON is that it's easy to compose JSON. If each file upload was a JSON body, it's trivial to compose them: just compose all _n_ separate JSON bodies into 1 big body with _n_ fields. But this has bad performance:

 - You have to base64 the file contents, because JSON can only handle text, not binary
 - The server has to buffer the entire JSON body into RAM before decoding it

I guess multipart/form-data is an attempt to compose multiple file uploads together _efficiently_. And that tradeoff brings some complexity, like boundaries and content-disposition. I wonder what a modern solution to this problem would look like. Clearly multipart/form-data is good enough, because it's being used everywhere. But if you know of any alternative solutions to this problem, please let me know in the comments!

[RFC 2388]: https://www.rfc-editor.org/rfc/rfc2388
[RFC 7578]: https://www.rfc-editor.org/rfc/rfc7578#section-4.8
[static-stream]: https://blog.adamchalmers.com/streaming-proxy/
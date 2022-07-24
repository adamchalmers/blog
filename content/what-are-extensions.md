+++
title = "What are Rust's HTTP extensions?"
date = 2022-07-24
description = "My definition, and how I use them at work"
draft = false

[taxonomies]
tags = ["rust", "programming", "web", "tower", "extensions", "tonic", "axum"]
+++

I learned about extensions when reading the [hyper docs][hyper_extns]. But they also pop up in lots of other Rust web libraries, like [http][http_extns], [tonic][tonic_extns], and [actix-web][actix_web_extns]. So they must be really useful, if so many libraries offer them. But I personally had no idea what they were or how to use them. In this blog, I'll explain what extensions really are (a set of values, keyed by type), and how you can use them to pass data between different parts of your web servers (e.g. middleware, router, and handler functions). I'll give you a real-world example from the gRPC server I'm building at my job.

<!-- more -->

By the way, this blog post assumes a little familiarity with Tonic, a gRPC library. If you've never used Tonic, maybe read [their excellent tutorial][tonic_hw_tut] first.

# What are extensions?

Extensions come from the [http crate][http_extns]. `http::Extensions` get wrapped by [tonic][tonic_extns], [actix-web][actix_web_extns], [hyper][hyper_extns] etc, because all those libraries are built on the http crate. There are two ways to think about what the Extensions type is:

 - It's a set of values, where no two values can have the same type.
 - It's a map, where each key is a type, and each key's value is a value of that type. 

These two descriptions are totally equivalent, so use whichever one is more intuitive to you.

Like normal Rust maps/sets, there's an `insert` and `get` operation. But unlike normal maps/sets, these methods don't have a key:

```rust
/// Insert a type into this Extensions.
/// If a extension of this type already existed, it will be returned.
fn insert<T: Send + Sync + 'static>(&mut self, val: T) -> Option<T>

/// Get a reference to a type previously inserted on this Extensions.
fn get<T: Send + Sync + 'static>(&self) -> Option<&T>
```

Why don't they have a key? Remember that `Extensions` can only store one value for a particular type. So if the Rust compiler knows what type you're inserting and getting, then it knows how to find the value.

```rust
use http::Extensions;
fn main() {
    let mut ext = Extensions::new();
    let value_in: i32 = 5;
    ext.insert(value_in);
    let value_out: &i32 = ext.get().unwrap();
    assert_eq!(&value_in, value_out)
}
```

Under the hood, this is implemented using a HashMap, where the keys are [TypeId][typeid]s and the values are [Any][any]s -- see [the http source][defn_of_extns] if you want to learn more. It's actually very straightforward. 

OK, so, `Extensions` are from the `http` crate, and they're just a set of values from many types. Why is that useful for web servers?

# Solving problems with extensions

## Passing extra information to handlers

A handler function is some function that accepts a request and returns a response. This definition is a little abstract, because there are handlers in all sorts of protocols -- HTTP, gRPC, your own custom protocol -- and they all use slightly different types. Handlers can be either async or blocking depending on your framework. 

[Axum][axum]'s docs have an example of a handler:

```rust
async fn create_user(
    // this argument tells axum to parse the request body
    // as JSON into a `CreateUser` type
    Json(payload): Json<CreateUser>,
) -> impl IntoResponse {
    // insert your application logic here
    let user = User {
        id: 1337,
        username: payload.username,
    };

    // this will be converted into a JSON response
    // with a status code of `201 Created`
    (StatusCode::CREATED, Json(user))
}
```

The `payload` variable is part of the request, and the `(StatusCode, Json)` return value is the response. But what if we wanted the response to depend on some server state, or a database? In Axum, you use `Extensions` to send this extra information to your handlers. In [Axum's TodoMVC example][axum_todo_ext], their handler uses extensions to get a database handle:

```rust
// https://github.com/tokio-rs/axum/blob/1fe45583626a4c9c890cc01131d38c57f8728686/examples/todos/src/main.rs#L86
async fn todos_index(
    pagination: Option<Query<Pagination>>,
    Extension(db): Extension<Db>,
) -> impl IntoResponse {
    let todos = db.read().unwrap();

    let Query(pagination) = pagination.unwrap_or_default();

    let todos = todos
        .values()
        .skip(pagination.offset.unwrap_or(0))
        .take(pagination.limit.unwrap_or(usize::MAX))
        .cloned()
        .collect::<Vec<_>>();

    Json(todos)
}
```

When I was researching extensions, I asked the Tokio discord what people used extensions for. [David Pedersen](https://github.com/davidpdrsn), one of the Axum devs, answered:

> axum uses them a bunch. The router will set extensions for “which router was matched”, “what was the original uri”, and a few others. There are then extractors that read those extensions.

In the above example, the `Extension(db): Extension<Db>` is an extractor that looks for a value in the `Extensions` with type `Db` and puts it in the variable named `db`. Axum's router sets the extension, and your handler reads its database handle. Pretty nice.

So here's your first answer: extensions are useful **because they let you pass extra information into handler functions**. 

### How do other frameworks solve this?

I've never actually used Axum. My current project at work uses [Tonic][tonic], and in Tonic, you don't need extensions to get a database handle. That's because Tonic's handler functions are actually methods of your server object. 

```rust
// Tonic servers are specific values, that you instantiate, of a particular type. 
// So your servers can have fields. In this example, the `RouteGuideService` type is a server, and
// handlers can read or modify the list of features just like a normal field.
pub struct RouteGuideService {
    features: Arc<Vec<Feature>>,
}


#[tonic::async_trait]
impl RouteGuide for RouteGuideService {
    // This is a handler function -- see how it accepts a request and returns a response?
    async fn get_feature(&self, request: Request<Point>) -> Result<Response<Feature>, Status> {

        // Handler functions can read the database as a property of `self`.
        for feature in &self.features[..] {
            if feature.location.as_ref() == Some(request.get_ref()) {
                return Ok(Response::new(feature.clone()));
            }
        }

        Ok(Response::new(Feature::default()))
    }
}
```

## Passing information from middleware to handlers

Extensions aren't as common in Tonic, because the problem of "how do I pass whatever types of parameters I want into my handlers" has an easier answer: just use fields of `self`.

But! That won't always work, because `self` is shared between every invocation of every handler. What if you want to pass a value for one request, specifically? For example, giving it log context. At Cloudflare we use structured logging (every log line has to be a JSON object, so they can be aggregated and filtered). I want to make sure that every log emitted by my handler logs some important context about the request, e.g. its URI, the User-Agent, and a unique-per-request ID. 

(aside: all the following code is [on GitHub](https://github.com/adamchalmers/tonic-extensions/blob/master/src/request_context.rs) in a full example project)

It's easy to extract this information from the HTTP request, and using [slog][slog] it's easy to create a logger instance that always logs those fields. 

```rust
// From https://github.com/adamchalmers/tonic-extensions/blob/master/src/request_context.rs
use hyper::{Body, Request, Uri};
use slog::{o, Logger};
use uuid::Uuid;

/// Information extracted from each request
pub struct RequestContext {
    pub user_agent: String,
    pub endpoint: Uri,
    pub request_id: Uuid,
}

impl RequestContext {
    /// Move the context fields from this value into the logger.
    pub fn into_logger(self, log: Logger) -> Logger {
        log.new(o!(
            "http.user_agent" => self.user_agent,
            "uri" => self.endpoint.to_string(),
            "request_id" => self.request_id.to_string(),
        ))
    }
}

/// You can create a RequestContext by examining a Hyper HTTP request.
impl From<&hyper::Request<Body>> for RequestContext {
    fn from(req: &Request<Body>) -> Self {
        let user_agent = req
            .headers()
            .get(hyper::header::USER_AGENT)
            .and_then(|ua| ua.to_str().ok())
            .unwrap_or("unknown")
            .to_owned();

        Self {
            user_agent,
            endpoint: req.uri().clone(),
            request_id: Uuid::new_v4(),
        }
    }
}
```

But how should the handler make its `RequestContext`? At work, my initial solution was:

1. The server object has a logger, configured when the program starts (i.e. outputs JSON to stderr, on its own thread so my gRPC handler tasks don't block waiting for the log mutex)
2. When a gRPC handler is invoked, it creates its own request-specific logger from that server-wide logger, like this: 
```rust
let req_ctx = RequestContext::from(&request)
let logger = req_ctx.into_logger(self.logger.clone())
```
3. Every handler should remember to use its request-specific `logger` object instead of `self.logger`.

But I don't want to rely on programmers (including my future self) remembering to do the right thing all the time. A better solution would _force_ every gRPC handler to use their own request-specific logger with those fields. So my new plan was:

1. The server _does not_ have a `self.logger` property. Instead it creates a [tower][tower] middleware with a logger. 
2. The middleware runs before every handler function, and creates a request-specific logger from the request context.
3. The handler function uses that request-specific logger, because it can't get a logger from anywhere else.

But how does the middleware send a value to the handler function? Using `Extensions`! The real example has a lot of boilerplate, so I'm only showing you the important part, where the middleware creates the logger and sends it to the request. 

```rust
// From https://github.com/adamchalmers/tonic-extensions/blob/master/src/middleware.rs
fn call(&mut self, mut req: hyper::Request<Body>) -> Self::Future {
    // Before the handler:
    let req_ctx = RequestContext::from(&req);
    let logger = req_ctx.into_logger(self.log.clone());
    req.extensions_mut().insert(logger.clone());
    // Call the handler:
    let resp = Box::pin(inner.call(req));
    // After the handler:
    resp
}
```

Then the handler gets the logger out of the request's extensions.

```rust
// From https://github.com/adamchalmers/tonic-extensions/blob/master/src/server.rs
pub struct MyGreeter;

#[tonic::async_trait]
impl Greeter for MyGreeter {
    async fn say_hello(
        &self,
        request: Request<HelloRequest>,
    ) -> Result<Response<HelloReply>, Status> {
        let log = request.extensions().get::<Logger>().unwrap().to_owned();
        let name = request.into_inner().name;
        if name == "David Bowie" {
            warn!(log, "Dead man sending requests...");
        }
        let reply = hello_world::HelloReply {
            message: format!("Hello {name}!"),
        };
        Ok(Response::new(reply))
    }
}
```

The handler function has **no other way** to get a logger, *except* the logger from the middleware. The one downside is that there's no way to know (at compile time) that the `Extensions` will have a logger, so at runtime, our handler has to do a `get()` which returns `Option<Logger>`, and unwrap it. This means our server will panic if the programmer forgot to set the logger on the `Extensions`. This kinda bothers me, but in practice, our middleware is so simple that it's not going to be a problem. I guess if our middleware gets so complicated that I need static guarantees that the logger is always there, I can refactor this and write a follow-up blog :)

## Passing information _from_ handlers back to middleware

We just saw that middleware can pass values into a handler with extensions. They do this using the `Request::extensions()` method. But responses have extensions too, so you can use `Response::extensions()` to send values _from_ the handler _back to_ the middleware! 

Let's continue the structured logging example. In my real work project, my service emits a log after every request. Let's add that log statement:

```rust
// From https://github.com/adamchalmers/tonic-extensions/blob/master/src/middleware.rs
fn call(&mut self, mut req: hyper::Request<Body>) -> Self::Future {
    // Before the handler:
    let req_ctx = RequestContext::from(&req);
    let logger = req_ctx.into_logger(self.log.clone());
    req.extensions_mut().insert(logger.clone());
    // Call the handler:
    let resp = Box::pin(inner.call(req));
    // After the handler:
    // THIS IS THE NEW LINE
    info!(logger, "Handled a request") 
    resp
}
```

It would be really helpful if those per-request logs also contained the "name" parameter from the gRPC. That way, I could filter for only logs from certain people, or aggregate logs to find the top 10 most common names. We can do that. Step one, the handler sets a new value in the response `Extensions`.

```rust
async fn say_hello(
    &self,
    request: Request<HelloRequest>,
) -> Result<Response<HelloReply>, Status> {
    let log = request.extensions().get::<Logger>().unwrap().to_owned();
    let name = request.into_inner().name;
    let reply = hello_world::HelloReply {
        message: format!("Hello {name}!"),
    };
    let mut response = Response::new(reply);
    response.extensions_mut().insert(name);
    Ok(response)
}
```

Step two, the middleware reads that value.

```rust
// From https://github.com/adamchalmers/tonic-extensions/blob/master/src/middleware.rs
fn call(&mut self, mut req: hyper::Request<Body>) -> Self::Future {
    // Before the handler:
    let req_ctx = RequestContext::from(&req);
    let logger = req_ctx.into_logger(self.log.clone());
    req.extensions_mut().insert(logger.clone());
    // Call the handler:
    let resp = Box::pin(inner.call(req));
    // After the handler:
    let name = response.extensions.get::<String>().unwrap();
    let logger = logger.new("name" => name);
    info!(logger, "Handled a request") 
    resp
}
```

### Tip: use newtypes with Extensions

Remember that `Extensions` can only have one value per type. It makes sense to store the name as a String. But this will cause problems if we want to put multiple strings into the `Extensions`, e.g. what if we need to log the "first name" and "surname" separately? Then we'd have two strings in the set, and one would overwrite the other. This could be especially bad in a large codebase, where multiple programmers are working in it. Imagine if I stored the name as a string in the extensions, and my coworker Olivia stored some authentication token as a string too. We'd need to coordinate and make sure nobody was using the same type in the extensions, otherwise our app might break horribly. 

The solution is to avoid storing standard Rust types in `Extensions`. Instead, use a newtype like `struct Name(String)`. That way, the `Extensions` can distinguish this particular string from any other strings you might want to include. Later on, we could change it to be `struct Names{first_name: String, last_name: String}` or use two structs, `FirstName(String)` and `LastName(String)`. In the full example code, that's exactly what I do.

# Summary/TLDR

Extensions come from [`http::Extensions`][http_extns] and get used by a lot of web libraries that build on the http crate. The `Extensions` type is just a set of values, where no two values can have the same type. This means, for any particular concrete type, you can query the set and get a value of that type (if it exists). This lets the different parts of your web server share data, in a way that's still _kinda_ type safe (if you ask for a `PostgresDbPool`, you'll get `Option<PostgresDbPool>`) but kinda dynamic (at compile-time, your web server can't be sure which types of values will be set in the extension, so you have to handle failure or panic if you need that `PostgresDbPool` and it's not there).

This lets you:
 - Pass a database connection from your router to your handler functions
 - Pass loggers or logging context from your request middleware to your handler functions, then back to your response middleware

I have some more examples of how Extensions can be useful, but this post is long enough. So, expect a second post with more Extensions examples. I'd love to hear how you're using Extensions, because I'm still pretty new to them. Please let me know in the comments, or [on twitter][twitter_question].

And, as always, if you want to get _paid_ to talk about Rust with me, Cloudflare is [hiring Rust engineers](https://www.cloudflare.com/careers/jobs/?title=rust).

---

#### Footnotes

[^foo]: bar


[actix_web_extns]: https://docs.rs/actix-web/latest/actix_web/dev/struct.Extensions.html
[axum_todo_ext]: https://github.com/tokio-rs/axum/blob/1fe45583626a4c9c890cc01131d38c57f8728686/examples/todos/src/main.rs#L88
[axum]: https://docs.rs/axum
[slog]: https://docs.rs/slog
[defn_of_extns]: https://docs.rs/http/latest/src/http/extensions.rs.html#6
[http_extns]: https://docs.rs/http/latest/http/struct.Extensions.html
[hyper_extns]: https://docs.rs/hyper/latest/hyper/struct.Request.html#method.extensions
[tonic_extns]: https://docs.rs/tonic/latest/tonic/struct.Extensions.html
[tonic]: https://docs.rs/tonic
[tower]: https://docs.rs/tower
[twitter_question]: https://twitter.com/adam_chal/status/1551229417208389635
[typeid]: https://doc.rust-lang.org/stable/std/any/struct.TypeId.html
[routeguide]: https://github.com/hyperium/tonic/blob/23c1392fb7e0ac50bcdedc35509917061bc858e1/examples/src/routeguide/server.rs#L28
[tonic_hw_tut]: https://github.com/hyperium/tonic/blob/master/examples/helloworld-tutorial.md

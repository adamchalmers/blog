+++
title = "Signals vs. Servers"
date = 2023-02-06
description = "A better UX for managing processes"
draft = false

[taxonomies]
tags = ["programming"]
+++

Say you're running a long-lived program, like a server. Let's say the server needs to read some files from disk, like certificates or keys. Every so often, the certificates change, so your server has to reload them. How do you tell the server to reload those files? The traditional way is to use Unix signals. Your server _listens_ for a particular signal, like `SIGUSR1` (user-defined signal #1) or `SIGHUP` (hangup signal), and can execute whatever code you programmed in when the signal is received.

But this has some problems, and I've been thinking about better ways to do this. I wanted to blog about one such way, and see if anyone has better ideas.

<!-- more -->

# Signal handling
The Tokio docs have a [great example of signal handling](https://docs.rs/tokio/latest/tokio/signal/index.html#examples). It's easy to make your server handle this signal. When you start your server, also start an async task (or goroutine, or thread) which listens for that signal, and when the signal is received, reload the certs. Also on [GitHub](https://github.com/adamchalmers/example-reload-server/blob/main/src/bin/main-signal.rs).

```rust
use axum::{routing::get, Router};
use std::process;
use tokio::signal::unix::{signal, SignalKind};

#[tokio::main]
async fn main() {
    let _cert = std::fs::read_to_string("cert.pem");
    println!("Loaded cert, starting web server");
    println!("My pid is {}", process::id());
    tokio::select! {
        _ = start_normal_server(8080) => {
            println!("the web server shut down")
        }
        _ = listen_for_reload(SignalKind::hangup()) => {
            println!("the signal listener stopped")
        }
    }
}

async fn start_normal_server(port: u32) {
    // build our application
    let app = Router::new().route("/hello", get(|| async { "Hello, world!" }));

    // run it
    let addr = format!("127.0.0.1:{port}").parse().unwrap();
    axum::Server::bind(&addr)
        .serve(app.into_make_service())
        .await
        .unwrap();
}

async fn listen_for_reload(signal_kind: SignalKind) -> Result<(), std::io::Error> {
    // An infinite stream of signals.
    let mut stream = signal(signal_kind)?;

    loop {
        stream.recv().await;

        match std::fs::read_to_string("cert.pem") {
            Ok(_) => eprintln!("Successfully reloaded cert"),
            Err(e) => eprintln!("could not reload cert: {e}"),
        }
    }
}
```


This works, but it's not a very good user experience for whoever's sending the signal. Say you're an SRE or a sysadmin who realizes the server needs to be reloaded. You look up the process's PID (5249) and you send the signal with `kill -s sighup 5249`. Now what?

Well, the server probably reloaded. But maybe it didn't. Maybe there was an error, like the new certificates are invalid. Or maybe the server doesn't have permission to read the new certs. How would the sysadmin know if that happened? Well, they should check the server's logs. But this requires switching windows, or opening a different program. Ideally, the sysadmin would _already_ have the logs open, and after they send the signal, they can just refresh the log page.

This isn't a great user experience. Generally, when you run a command, you expect to get some feedback. If you're running the command through the terminal, you expect an exit code (0 means OK, nonzero numbers refer to specific errors that you look up in manpages or a help document). Ideally, if something goes wrong, the program logs the error to stderr.

But when you send a Unix signal, you don't get any feedback in response. You have to go look up the other server's logs. At work, we've occasionally had incidents where SREs didn't notice the log errors, because they aren't familiar enough with the program to find the specific error they caused, in a sea of other errors and warnings.

The main problem with signals is the user can signal a process, but doesn't get any response from the process.

# A better way: control servers

So, we want the process to accept a request ("reload your certificates") and respond back ("yes it succeeded" or "it failed, here's why"). Well, this sounds familiar -- it's just a normal request-response protocol. No need to reinvent the wheel -- why not just start a little HTTP server in your process? Also on [GitHub](https://github.com/adamchalmers/example-reload-server/blob/main/src/bin/main-http.rs).

```rust
use axum::{
    http::StatusCode,
    response::IntoResponse,
    routing::{get, post},
    Router,
};

#[tokio::main]
async fn main() {
    let _cert = std::fs::read_to_string("cert.pem");
    println!("Loaded cert, starting web server");
    tokio::select! {
        _ = start_normal_server(8080) => {
            println!("the web server shut down")
        }
        _ = start_control_server(3000) => {
            println!("the control server shut down")
        }
    };
}

async fn start_normal_server(port: u32) {
    // build our application
    let app = Router::new().route("/hello", get(|| async { "Hello, world!" }));

    // run it
    let addr = format!("127.0.0.1:{port}").parse().unwrap();
    axum::Server::bind(&addr)
        .serve(app.into_make_service())
        .await
        .unwrap();
}

async fn start_control_server(port: u32) {
    // build our application
    let app = Router::new().route(
        "/reload_certs",
        post(|| async {
            println!("Reloading cert");
            match std::fs::read_to_string("cert.pem") {
                Ok(_) => "Successfully reloaded cert".into_response(),
                Err(e) => {
                    let error = format!("could not reload cert: {e}");
                    eprintln!("{error}");
                    let resp = (StatusCode::INTERNAL_SERVER_ERROR, error);
                    resp.into_response()
                }
            }
        }),
    );

    // run it
    let addr = format!("0.0.0.0:{port}").parse().unwrap();
    axum::Server::bind(&addr)
        .serve(app.into_make_service())
        .await
        .unwrap();
}
```

This is a much better user experience for your SRE or sysadmin. They run

```
$ curl -X POST 0.0.0.0:3000/reload_certs
Successfully reloaded cert
```

And if the cert wasn't found, they get immediate feedback about the error:
```
$ curl -X POST 0.0.0.0:3000/reload_certs
could not reload cert: No such file or directory (os error 2)
```

curl will even set the exit code correctly, so this becomes very easily scriptable. In a large system, you probably won't have SREs manually executing those commands, they'll be run on a schedule or triggered by Kubernetes or systemd in response to some other event. These systems can check the exit code from curl to know if the reload failed.

# Problems and tradeoffs

If your process doesn't already need HTTP or networking, then bringing in a whole HTTP framework just to listen for signals might be a bit of overkill. Especially if you're building a Rust program, a HTTP server might be a somewhat heavyweight dependency that adds a lot of transitive dependencies and compiletime. So, depending on the size of the program (and of your system administrator or SRE team), this might not be worth it. Adding a HTTP server makes your program larger and more complicated, but it has a better UX for the people and software that are managing your process. This is a trade-off and there's no right or wrong answer. For the programs I've been writing at work over the last few years, the control server approach is a no-brainer, but your needs might differ.

By the way, at my actual job, my control servers listen over Unix domain sockets, not TCP ports. Unix sockets are simpler than TCP, and maybe faster depending on your OS. They're also easier to secure, because the socket is just a file. So you can lock it down with the normal Unix file permission system -- no need for TLS. Of course, your OS might not support Unix sockets -- I don't know if there's an equivalent on Windows.

I'm sure this pattern has been talked about many times, but I wanted to write up my thoughts on it anyway to explain the pattern to other teammates, and I didn't know its existing name. If you've got any pointers to existing material about this pattern, please leave me a link below. Or if you've got a different approach to solving this problem, I'd love to hear about it.

[gh]: https://github.com/adamchalmers/example-reload-server/
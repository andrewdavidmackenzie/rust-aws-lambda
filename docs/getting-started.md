---
title: Getting Started
layout: default
---

# Getting Started

## Install

Because this project is still in an early (but functional!) stage, it has not yet been published to the `crates` registry. You will therefore need to depend directly on the Github repository. Add the following to the `[dependencies]` section in your `Cargo.toml` file.

```toml
aws_lambda = { git = "https://github.com/srijs/rust-aws-lambda" }
```

## Create

The `start` function will launch a runtime which will listen for messages from the lambda environment, and call a handler function every time the lambda is invoked. This handler function can be async, as the runtime itself is based on top of `futures` and `tokio`.

```rust,no_run
extern crate aws_lambda as lambda;

fn main() {
    // start the runtime, and return a greeting every time we are invoked
    lambda::start(|()| Ok("Hello ƛ!"))
}
```

Alternatively, the `gateway` module contains functionality to implement a lambda function that can be used to [build an API Gateway API with Lambda Proxy Integration](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-create-api-as-simple-proxy-for-lambda.html).

```rust,no_run
extern crate aws_lambda as lambda;

fn main() {
    lambda::gateway::start(|_req| {
        let res = lambda::gateway::response()
            .status(200)
            .body("Hello ƛ!".into())?;
        Ok(res)
    })
}
```

## Input

To provide input data to your handler function, you can change the type of the argument that the function accepts. For this to work, the argument type needs to implement the `serde::Deserialize` trait (most types in the standard library do).

```rust,no_run
extern crate aws_lambda as lambda;

use std::collections::HashMap;

fn main() {
    lambda::start(|input: HashMap<String, String>| {
        Ok(format!("the values are {}, {} and {}",
            input["key1"], input["key2"], input["key3"]))
    })
}
```

Additionally, the `event` module provides strongly-typed lambda event types for use with [AWS event sources](https://docs.aws.amazon.com/lambda/latest/dg/invoking-lambda-function.html).

For example, this would print out all the `S3Event` record names, assuming your lambda function was subscribed to the [proper S3 events](https://docs.aws.amazon.com/lambda/latest/dg/with-s3-example.html):

```rust,no_run
extern crate aws_lambda as lambda;

use lambda::event::s3::S3Event;

fn main() {
    lambda::start(|input: S3Event| {
        let mut names = Vec::new();
        for record in input.records {
            names.push(record.event_name);
        }
        Ok(format!("Event names:\n{:#?}", names))
    })
}
```

The types in the `event` module are automatically generated from the [official Go SDK](https://github.com/aws/aws-lambda-go/tree/master/events) and thus are generally up-to-date.

#### Dealing with `null` and empty strings in lambda input

The official Lambda Go SDK sometimes marks a field as required when the underlying lambda event json could actually be `null` or an empty string. Normally, this would cause a panic as Rust is much more strict.

The `event` module deals with this reality by marking all required json string fields as `Option<String>` in Rust. Json `null` or the empty string are deserialized into Rust structs as `None`.

## Context

While your function is running you can call `Context::current()` to get additional information, such as the ARN of your lambda, the Amazon request id or the Cognito identity of the calling application.

```rust,no_run
extern crate aws_lambda as lambda;

fn main() {
    lambda::start(|()| {
        let ctx = lambda::Context::current();

        Ok(format!("Hello from {}!", ctx.invoked_function_arn()))
    })
}
```

## Logging

The `aws_runtime` crate bundles its own logger, which can be used through the
[`log`](https://crates.io/crates/log) facade.

To initialize the logging system, you can call `logger::init()`.

```rust,no_run
extern crate aws_lambda as lambda;
#[macro_use] extern crate log;

fn main() {
    lambda::logger::init();

    lambda::start(|()| {
        info!("running lambda function...");

        Ok("Hello ƛ!")
    })
}
```

## Deploy

_Note: These instructions will produce a static musl binary of your rust code. If you are looking for non-musl binaries, you might try [docker-lambda](https://github.com/lambci/docker-lambda)._

To deploy on AWS lambda, you will need a zip file of your binary. If you are running Linux, this may be as simple as running `cargo --target x86_64-unknown-linux-musl` if your dependencies do not require OpenSSL. If you are on non-Linux platforms, the binary needs to built against amazonlinux or cross-compiled for musl, as described in [this tutorial](https://medium.com/@bernardo.belchior1/running-rust-natively-in-aws-lambda-and-testing-it-locally-57080421426d).

A Dockerfile is provided as an example and will work for single project binaries that need OpenSSL. The Dockerfile is based off of [rust-musl-builder](https://github.com/emk/rust-musl-builder).

    docker pull amazonlinux
    docker build --force-rm -t aws-lambda:latest --build-arg SRC=example -f docker/dockerfile .
    docker run -v /tmp/artifacts:/export --rm aws-lambda:latest

Build your lambda function and upload your zip file. Change the lambda runtime to `go 1.x` and set the handler function to the name of your application as defined in your `Cargo.toml`.

#### SSL considerations

If your binary requires SSL, add the following environment variables:

    SSL_CERT_DIR=/etc/ssl/certs
    SSL_CERT_FILE=/etc/ssl/certs/ca-bundle.crt

If you are still running into SSL issues using the Docker image or are building directly on Linux, you may need to modify your application per https://github.com/emk/rust-musl-builder#making-openssl-work.

When using [`rusoto`](https://github.com/rusoto/rusoto) it is highly suggested to use the crate's [`rustls` feature](https://github.com/rusoto/rusoto/blob/master/rusoto/core/README.md#usage-with-rustls) instead of building OpenSSL.

#### `error_chain`

In general we suggest you use the [`failure`](https://github.com/rust-lang-nursery/failure) crate for error handling. If you are instead using [`error_chain`](https://github.com/rust-lang-nursery/error-chain) in your rust code, you will also have to disable default features to use the example musl Dockerfile. Add the following to your `Cargo.toml`:

    [dependencies.error-chain]
    version = "~0.12"
    default-features = false

## Troubleshooting

To help you debug your lambda function, `aws_lambda` integrates with the [`failure`](https://github.com/rust-lang-nursery/failure)
crate to extract stack traces from errors that are returned from the handler function.

In order to take advantage of this, you need to compile your program to include debugging symbols. When working with `cargo` using `--release`, you can add the following section to your `Cargo.toml` to include debug info in your release build:

```toml
[profile.release]
debug = true
```

Next, you want to instruct the runtime to collect stack traces when errors occur. You can do this by modifying the configuration of your function in AWS to set the `RUST_BACKTRACE` environment variable to `1`.

After both of these changes have been deployed, you should start to see stack traces included in both the error info returned from invocations, as well the CloudWatch logs for your function.

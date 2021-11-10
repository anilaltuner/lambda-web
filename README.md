# lambda-web

Run Rust web server frameworks on AWS Lambda.
Currently, it supports Actix web, Rocket, Warp.

[![crates.io](https://img.shields.io/crates/v/lambda-web?label=latest)](https://crates.io/crates/lambda-web)
[![API docs](https://docs.rs/lambda-web/badge.svg)](https://docs.rs/lambda-web)

## Features

### Supported web frameworks

- [Actix Web](https://crates.io/crates/actix-web/4.0.0-beta.10) 4.0.0-beta.10
- [Rocket](https://crates.io/crates/rocket/0.5.0-rc.1) 0.5.0-rc.1
- [Warp](https://crates.io/crates/warp) 0.3.1

### Supported AWS infrastructure

- [API Gateway HTTP API](https://docs.aws.amazon.com/apigateway/latest/developerguide/http-api.html) with [payload format version 2.0](https://docs.aws.amazon.com/apigateway/latest/developerguide/http-api-develop-integrations-lambda.html#2.0)
- [API Gateway REST API](https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-rest-api.html)

### Not supported

- API Gateway HTTP API with payload format version **1.0**
- [Application Load Balancer (ALB)](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/lambda-functions.html)

## Example

### Actix Web

Cargo.toml

```toml
[[bin]]
name = "bootstrap"
path = "src/main.rs"

[dependencies]
lambda-web = { version = "0.1.7", features=["actix4"] }
```

main.rs

```rust
use lambda_web::actix_web::{self, get, App, HttpServer, Responder};
use lambda_web::{is_running_on_lambda, run_actix_on_lambda, LambdaError};

#[get("/")]
async fn hello() -> impl Responder {
    format!("Hello")
}

#[actix_web::main]
async fn main() -> Result<(),LambdaError> {
    let factory = move || {
        App::new().service(hello)
    };

    if is_running_on_lambda() {
        // Run on AWS Lambda
        run_actix_on_lambda(factory).await?;
    } else {
        // Local server
        HttpServer::new(factory)
            .bind("127.0.0.1:8080")?
            .run()
            .await?;
    }
    Ok(())
}
```

### Rocket

Cargo.toml

```toml
[[bin]]
name = "bootstrap"
path = "src/main.rs"

[dependencies]
lambda-web = { version = "0.1.7", features=["rocket05"] }
rocket = "0.5.0-rc.1"
```

main.rs

```rust
use rocket::{self, get, routes};
use lambda_web::{is_running_on_lambda, launch_rocket_on_lambda, LambdaError};

#[get("/hello/<name>/<age>")]
fn hello(name: &str, age: u8) -> String {
    format!("Hello, {} year old named {}!", age, name)
}

#[rocket::main]
async fn main() -> Result<(), LambdaError> {
    let rocket = rocket::build().mount("/", routes![hello]);
    if is_running_on_lambda() {
        // Launch on AWS Lambda
        launch_rocket_on_lambda(rocket).await?;
    } else {
        // Launch local server
        rocket.launch().await?;
    }
    Ok(())
}
```

### Warp

Cargo.toml

```toml
[[bin]]
name = "bootstrap"
path = "src/main.rs"

[dependencies]
lambda-web = { version = "0.1.7", features=["warp03"] }
warp = "0.3"
tokio = { version = "1" }
```

main.rs

```rust
use lambda_web::{is_running_on_lambda, run_warp_on_lambda, LambdaError};
use warp::Filter;

#[tokio::main]
async fn main() -> Result<(), LambdaError> {
    // GET /hello/warp => 200 OK with body "Hello, warp!"
    let hello = warp::path!("hello" / String).map(|name| format!("Hello, {}", name));

    if is_running_on_lambda() {
        // Run on AWS Lambda
        run_warp_on_lambda(warp::service(hello)).await?;
    } else {
        // Run local server
        warp::serve(hello).run(([127, 0, 0, 1], 8080)).await;
    }
    Ok(())
}
```

## Create deploy ZIP file

Currentry (Jun 2021), we have two options to run Rust on AWS Lambda: [Amazon Linux 2 custom runtime](https://docs.aws.amazon.com/lambda/latest/dg/runtimes-custom.html) or Docker container image.

I recommend Amazon Linux 2 custom runtime deploy because it's faster cold start time than container image.

To build Amazon Linux 2 compatible binary, it's better to build inside container. First, build Amazon Linux 2 container with Rust toolchain. This repository contains [sample Dockerfile](https://github.com/hanabu/lambda-web/blob/main/docker/Dockerfile) .

```console
$ git clone https://github.com/hanabu/lambda-web
...
$ docker build -t lambda_builder lambda-web/docker
...

or
$ buildah bud -t lambda_builder lambda-web/docker
...
```

Once you get lambda\_builder image, then build your code with Amazon Linux.

```console
$ cd your_app_crate_dir
$ docker run -it --rm -v ~/.cargo/registry:/root/.cargo/registry:z -v .:/build:z lambda_builder
...

or
$ podman run -it --rm -v ~/.cargo/registry:/root/.cargo/registry:z -v .:/build:z lambda_builder
...
```

Then, you get deploy ZIP package in your\_app\_crate\_dir/target\_lambda/deploy.zip .

Make sure, your Cargo.toml has `bootstrap` binary name.

```toml
[[bin]]
name = "bootstrap"
path = "src/main.rs"
```

## Setup AWS Lambda & API gateway

### Lambda

- Create lambda function with `provided.al2` custom runtime. Choose "Provide your own bootstrap on Amazon Linux 2" .
- Upload ZIP file described above.
- IAM role, memory settings, etc. are as your demands. \
  As sample code above consumes only 30MB of memory, many simple Rust app can fit in 128MB setting.

### API Gateway (HTTP)

- Create HTTP API
- Create single route "$default" and attach Lambda integration. Make sure, payload format version is "2.0"

### API Gateway (REST)

- Create REST API
- Create two resources:
  - ANY method on route `/` and attach Lambda proxy integration.
  - ANY method on route `/{proxy+}` and attach Lambda proxy integration.
- In settings tab, add `*/*` binary media type.
- Then, deploy API to stage.

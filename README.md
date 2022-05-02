# rust-grpc-example

A minimal Rust based gRPC client and server (using [tonic-rs](https://github.com/hyperium/tonic))

Background reading/references:
- [protobuf Primer](https://developers.google.com/protocol-buffers/docs/tutorials)
- [gRPC Primer](https://www.grpc.io/docs/what-is-grpc/introduction/)


## Protobuf service definitions to Rust codegen

Some popular crates:

- [Tonic](https://github.com/hyperium/tonic)
- [grpc-rust](https://github.com/stepancheg/grpc-rust)
- [Dropbox's `pb-jelly`](https://github.com/dropbox/pb-jelly)

Some references to popular project(s) which are using Tonic (tonic-build):

- [Linkerd - The Kubernetes Service Mesh](https://github.com/linkerd/linkerd2-proxy/blob/main/opencensus-proto/build.rs#L5)

I chose Tonic because it's used in [Linkerd](https://linkerd.io/) a project I really like. I couldn't find an ADOPTERS file on the project, so I'm not sure who else is using it.
A simple google search wasn't really useful. So if you do find any other major projects using Tonic or if you yourself are using it, I would [love to hear about how you're using it.](https://www.swiftdiaries.com)

## Tonic

### Goals for today

- Define a service in a proto file
- Generate Rust protobuf definitions and service definitions
- Create a gRPC server binary for the service
- Create a gRPC client binary for interacting with the service

### Setup a Rust project

Cargo is the dependency management tool for Rust. You can install it from [here](https://doc.rust-lang.org/cargo/getting-started/installation.html).

To create a new package with Cargo, use `cargo new`:
```bash
cargo new rust-grpc-example
cd rust-grpc-example
```

Cargo will now automatically create a git repo. And populate it with bare-essentials.

Recently, I used the [CLion IDE's](https://www.jetbrains.com/clion/) in-built project creation workflow and it offers the same Cargo workflow but with a couple of clicks.
There's a free license program available and you can get it if you qualify, [check your qualification](https://www.jetbrains.com/community/opensource/#support).

### Add dependencies

Let's take a look at the `Cargo.toml` file. For people coming from Go, this is kind of like the `go.mod` but better.
For people coming from python, well this is a really cool way to do dependency management.

```toml
[package]
name = "rust-grpc-example"
version = "0.1.0"
authors = ["swiftdiaries <adhita94@gmail.com>"]
edition = "2018"

# See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html

[dependencies]
```

Add tonic related dependencies to the project.

```toml
[dependencies]
tonic = "0.7"
prost = "0.10.1"
tokio = { version = "1.18", features = ["macros", "rt-multi-thread"] }

[build-dependencies]
tonic-build = "0.7"
```

The `tonic-build` crate is built on top of the `prost-build` crate that we used in our last blog post. Check out the [`tonic-build` docs](https://docs.rs/tonic-build/0.3.1/tonic_build/) for more configuration options while generating service definitions.

### Fun part

### Create the service definition in a proto file

Generally, I find it easier to grok if I have proto files in a separate directory. Especially when dealing with multiple languages as is common with `gRPC-based` services.

```bash
# Create a directory for proto files and a file to hold them
mkdir -p api/protos
touch api/protos/greeter.proto

# Create the service definition
cat << EOF >> src/greeter.proto
syntax = "proto3";
package greeter;

message HelloRequest {
    string name = 1;
}

message HelloResponse {
    string message = 1;
}

service Greeter {
    rpc SayHello (HelloRequest) returns (HelloResponse);
}
EOF
```

### Tonic-build to compile the proto-based service definition

Any file named `build.rs` and placed at the root of the package will act like a build-script. Cargo will compile and execute it before compiling the files inside the `src/` directory.

```bash
touch build.rs
cat << EOF >> build.rs
fn main() -> Result<(), Box<dyn std::error::Error>> {
    tonic_build::compile_protos("api/protos/greeter.proto")?;
    Ok(())
}
EOF
```

>
```rust
tonic_build::compile_protos("api/protos/greeter.proto")?;
```
> We're using the `tonic-build` library's `compile_protos` function with the filepath to the proto file relative to the root of the project as the input.

#### Build the stubs

```bash
cargo build
``` 

And boom, you have the generated stubs ! :tada::tada::tada::tada:

##### But, where is it?

The name of the package defined in the proto file determines the name of the output `.rs` file. 

By default, tonic-build stores the generated output stubs in `target/debug/build/{project-name}-{hash}/out` directory.

The `{project-name}` corresponds to your project name and the `{hash}` refers to the current `cargo build` hash.

#### Configuration options

You can also use the `configure` function in tonic_build instead of `compile_protos` to customize your code generation.
The [configure function](https://docs.rs/tonic-build/0.3.1/tonic_build/fn.configure.html) returns a [Builder struct](https://docs.rs/tonic-build/0.3.1/tonic_build/struct.Builder.html).

Reference: [All the options to configure](https://docs.rs/tonic-build/0.3.1/tonic_build/struct.Builder.html#implementations).

#### Configuration example

To see how it works in context, let's take a look at how a sub-project in the linkerd project uses the function.

```rust
fn main() {
    let iface_files = &["opencensus/proto/agent/trace/v1/trace_service.proto"];
    let dirs = &["."];

    tonic_build::configure()
        .build_client(true)
        .compile(iface_files, dirs)
        .unwrap_or_else(|e| panic!("protobuf compilation failed: {}", e));

    // recompile protobufs only if any of the proto files changes.
    for file in iface_files {
        println!("cargo:rerun-if-changed={}", file);
    }
}
```

>
```rust
let iface_files = &["opencensus/proto/agent/trace/v1/trace_service.proto"];
```
> This defines proto files we're going to generate stubs for. If you recall, with `protoc` these are the files you'd pass with the `-I` flag. \

>
```rust
let dirs = &["."];
```
> The directories under which `protoc` should look for dependencies in the proto file.
Then the `configure` function in itself. \

>
```rust
tonic_build::configure()
        .build_client(true)
        .compile(iface_files, dirs)
        .unwrap_or_else(|e| panic!("protobuf compilation failed: {}", e));
```
> The `build_client` option determines whether to build the client-specific code, `build-server` being another option.
> The `compile` option takes in the `protos` as the first argument and `includes` as the second argument (read: `-I` from `protoc`). \

References: \
[Source Code](https://github.com/linkerd/linkerd2-proxy/blob/main/opencensus-proto/build.rs) \
[Credits: Author](https://github.com/hawkw)

### Use the stubs

#### Generated stubs
Let's take a peek at the generated stubs.

```rust
#[derive(Clone, PartialEq, ::prost::Message)]
pub struct HelloRequest {
    #[prost(string, tag = "1")]
    pub name: std::string::String,
}
```
> This is the generated struct for the HelloRequest message
>
```proto
message HelloRequest {
    string name = 1;
}
``` \


```rust
#[derive(Clone, PartialEq, ::prost::Message)]
pub struct HelloResponse {
    #[prost(string, tag = "1")]
    pub message: std::string::String,
}
```
> This is the generated struct for the HelloResponse message
>
```proto
message HelloRequest {
    string name = 1;
}
``` \


```rust
#[doc = r" Generated client implementations."]
pub mod greeter_client {
    ...
}
#[doc = r" Generated server implementations."]
pub mod greeter_server {
    ...
}
```

> These are the client and server stubs for the Greeter service
>
```proto
service Greeter {
    rpc SayHello (HelloRequest) returns (HelloResponse);
}
```
\

Now, let's see how to import this into our gRPC Server and Client code.
### Build a gRPC Server with Tonic

First, we're going to build the gRPC Server. Create a file `src/server.rs`.

```bash
touch src/server.rs
```

> Define the Tonic-based building blocks for the server. \
> Import the generated stubs by specifying the package name. \

```rust
use tonic::{transport::Server, Request, Response, Status};

use greeter::greeter_server::{Greeter, GreeterServer};
use greeter::{HelloResponse, HelloRequest};

// Import the generated proto-rust file into a module
pub mod greeter {
    tonic::include_proto!("greeter");
}
```
> Define the service skeleton for the Greeter Service. Implement the individual functions that the service holds. \
> For example, `SayHello`\

```rust
// Implement the service skeleton for the "Greeter" service
// defined in the proto
#[derive(Debug, Default)]
pub struct MyGreeter {}

// Implement the service function(s) defined in the proto
// for the Greeter service (SayHello...)
#[tonic::async_trait]
impl Greeter for MyGreeter {
    async fn say_hello(
        &self,
        request: Request<HelloRequest>,
    ) -> Result<Response<HelloResponse>, Status> {
        println!("Received request from: {:?}", request);

        let response = greeter::HelloResponse {
            message: format!("Hello {}!", request.into_inner().name).into(),
        };

        Ok(Response::new(response))
    }
}
```

> Use the tokio runtime to create an instance of a gRPC Server.
> The tokio instance takes in the implemented service definition above as input. \
> And serves it over the port `50051` while waiting for requests. \

```rust
// Use the tokio runtime to run our server
#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let addr = "[::1]:50051".parse()?;
    let greeter = MyGreeter::default();

    println!("Starting gRPC Server...");
    Server::builder()
        .add_service(GreeterServer::new(greeter))
        .serve(addr)
        .await?;

    Ok(())
}
```

#### Run the gRPC Server

Add the `server.rs` to `Cargo.toml` to able to build, run it as a binary.
```toml
[[bin]] # Bin to run the HelloWorld gRPC server
name = "helloworld-server"
path = "src/server.rs"
```

```bash
cargo run --bin helloworld-server
```

And boom, you now have a running gRPC Server in Rust.

#### Test the server

Quickly test the server with [grpcurl](https://github.com/fullstorydev/grpcurl#installation).

```bash
 grpcurl -plaintext -import-path ./api/protos \
    -proto ./api/protos/greeter.proto \
    -d '{"name": "Tonic"}' \
    [::]:50051 \
    greeter.Greeter/SayHello
```

### Build a gRPC Client with Tonic

Second, we build the client to send requests to the gRPC server. Create a file `src/client.rs`.

```bash
touch src/client.rs
```
> Import the generated stubs with the package name. \

```rust
use greeter::greeter_client::GreeterClient;
use greeter::HelloRequest;

// Import the generated proto-rust file into a module
pub mod greeter {
    tonic::include_proto!("greeter");
}
```
> Use generated client interface on the stubs to connect to the gRPC server at port `50051`.\
> Create a request with the tokio library's Request method and the generated request struct to call the function `SayHello` \
> Wait for the request to fetch a response \
```rust
#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let mut client = GreeterClient::connect("http://[::1]:50051").await?;

    let request = tonic::Request::new(HelloRequest {
        name: "Tonic".into(),
    });

    println!("Sending request to gRPC Server...");
    let response = client.say_hello(request).await?;

    println!("RESPONSE={:?}", response);

    Ok(())
}
```

#### Run the gRPC Client

Add the `client.rs` to the `Cargo.toml` to build, run it as a binary.
```toml
[[bin]] # Bin to run the HelloWorld gRPC client
name = "helloworld-client"
path = "src/client.rs"
```

In a separate terminal run,
```bash
cargo run --bin helloworld-client
```

And voila :tada: :tada: :tada:

Please feel free to open an issue in the [GitHub repo](https://github.com/swiftdiaries/rust-grpc-example) if you find mistakes here.

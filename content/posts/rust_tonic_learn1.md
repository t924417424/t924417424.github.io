---
title: "Rust学习之tonic-一个grpc的rust实现"
date: 2022-07-27T14:26:21+08:00
author: "#"              # 文章作者
description : "tonic是一个rust语言的grpc实现"    # 文章描述信息
categories: ["rust","tonic","grpc"]
tags: [
    "rust",
    "grpc",
    "tonic",
    "development",
]
---

## tonic简介（引用自[crates.io](https://crates.io/crates/tonic)）

>A rust implementation of gRPC, a high performance, open source, general RPC framework that puts mobile and HTTP/2 first.
tonic is a gRPC over HTTP/2 implementation focused on high performance, interoperability, and flexibility. This library was created to have first class support of async/await and to act as a core building block for production systems written in Rust.

## tonic特点：
- 由rust社区使用rust语言实现
- 脱离google官方grpc工具链，如protoc
- 支持rust异步运行时
- 支持build.rs

## tonic使用笔记
### Cargo.toml中引入相关依赖
```toml
[dependencies]
tonic = "0.7.2" # 包含grpc中的核心功能，如开启服务监听等
prost = "0.10"  # 主要用于处理proto生成的rust源代码，如序列化、反序列化消息
tokio = {version = "1.20.0", features = ["rt-multi-thread", "time", "fs", "macros", "net"]} # 目前的主流rust异步运行时

[build-dependencies]
tonic-build = "0.7.2"   # 编译时依赖，顾名思义，主要为build.rs提供编译protoc的相关功能
```

### 编写protoc文件（这里以tonic官方示例做演示）
```protobuf
// proto/echo/echo.proto
syntax = "proto3";

package grpc.examples.echo;

// EchoRequest is the request for echo.
message EchoRequest {
  string message = 1;
}

// EchoResponse is the response for echo.
message EchoResponse {
  string message = 1;
}

// Echo is the echo service.
service Echo {
  // UnaryEcho is unary echo.
  rpc UnaryEcho(EchoRequest) returns (EchoResponse) {}
}
```

### 编写build.rs
```rust
fn main(){
    tonic_build::compile_protos("proto/echo/echo.proto").unwrap();
}
```
### 知识扩展
- tonic支持一些生成配置，如添加自定义trait、自定义输出目录等，可使用`tonic_build::configure()`来操作这些属性
```rust
// 自定义proto产出文件的保存目录，如不自定义这个的化，需要使用 tonic的相关宏来引入（后续展开说明）
// 自定义proto产出文件相关结构体的 message->struct 的derive宏属性
tonic_build::configure().out_dir("src/").type_attribute(".", "#[derive(serde::Serialize, serde::Deserialize)]").compile(
        &["proto/echo/echo.proto"], 
        &["protobuf"]
    ).unwrap();
```
### 创建并开启grpc服务
```rust
use echo::{EchoRequest,EchoResponse,echo_server::{Echo,EchoServer}};
use prost::Message;
use tonic::{Response, transport::Server};

pub mod echo {
    // 因为上面的build.rs没有定义产出rust源文件的输出目录，所以这里需要使用这个宏来引用你proto对应的rust文件
    // grpc.examples.echo 对应的是 proto 文件中的 package 属性
    tonic::include_proto!("grpc.examples.echo");
}

#[derive(Default,Debug)]
struct MyServer{}

// 这里使用异步的方式来实现服务
#[tonic::async_trait]
impl Echo for MyServer{
    async fn unary_echo(
        &self,
        request: tonic::Request<EchoRequest>,
    ) -> Result<tonic::Response<EchoResponse>, tonic::Status>{
        println!("remote addr:{:?}  message:{:?}",request.remote_addr(),request.get_ref().message);
        let reply = EchoResponse{
            message:format!("hi client:{:?}",request.remote_addr())
        };
        // 序列化和反序列化测试，这里的主要功能由 prost 包提供
        let buf = reply.encode_to_vec();
        println!("proto encode buf:{:?}",buf);
        let r:EchoResponse = Message::decode(&buf[..buf.len()]).unwrap();
        println!("decode:{:?}",r.message);
        Ok(Response::new(reply))
    }
}


#[tokio::main]
async fn main() {
    let addr = "[::1]:50051".parse().unwrap();
    let echo = MyServer::default();
    println!("EchoServer listening on {}", addr);
    Server::builder()
        .add_service(EchoServer::new(echo))
        .serve(addr)
        .await.unwrap();
}
```

### 创建客户端
```rust
pub mod echo {
    tonic::include_proto!("grpc.examples.echo");
}

#[tokio::main]
async fn main(){
    let mut echo_client = echo::echo_client::EchoClient::connect("http://[::1]:50051").await.unwrap();
    let request = tonic::Request::new(echo::EchoRequest{
        message: "hi server".into()
    });
    let resp = echo_client.unary_echo(request).await.unwrap();
    println!("recv server data:{:?}",resp.get_ref().message);
}
```

### 启动服务端并测试
```
client output:
   Compiling test_0 v0.1.0 (/home/oldcat/project/rust/test_0)
    Finished dev [unoptimized + debuginfo] target(s) in 6.00s
     Running `target/debug/client`
recv server data:"hi client:Some([::1]:47758)"
```
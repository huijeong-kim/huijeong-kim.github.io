---
layout: post
title: Async Rust에서 mpsc queue 사용하기
tags: rust mpsc tokio async
---

mpsc(multi produce single consumer) queue는 thread 간 message를 주고받는 channel로 많이 쓰입니다. [std mpsc](https://doc.rust-lang.org/std/sync/mpsc/index.html)와 [crossbeam channel](https://docs.rs/crossbeam/latest/crossbeam/channel/index.html)가 많이 쓰이는 mpsc channel이고, async rust에서는 [tokio mpsc](https://docs.rs/tokio/1.20.1/tokio/sync/mpsc/index.html)를 사용할 수 있습니다.

async rust에서 mpsc queue를 사용하는 방법을 알아보겠습니다.

 
 

### 1. std::sync::mpsc 사용하기

Async rust에서 mpsc를 사용하려면 여러 future들이 message sender(tx)를 갖고 있고 하나의 future가 message receiver(rx)를 갖고 있어야 합니다. mpsc의 `Sender`는 clone 가능하므로 다음과 같이 tx를 clone하여 message channel을 공유할 수 있습니다.

```rust
usd std::sync::mpsc;

#[tokio::main]
async fn main() {
    let (tx, rx) = mpsc::channel();

    for sender_id in 0..10 {
        let tx_channel = tx.clone();
        tokio::spawn(async move {
            tx_channel.send(sender_id).expect("Failed to send message");
        });
    }

    drop(tx);

    let handle = tokio::task::spawn(async move {
        while let Ok(i) = rx.recv() {
            println!("got = {}", i);
        }
    });

    handle.await;
}
```

`Sender`가 필요한 async future를 spawn 할 때 마다 message channel의 `Sender`를 clone 하였습니다. 중간에 처음 생성한 tx를 drop 하는 것은, 모든 tx 가 drop 되어야 channel이 close 되고 `rx.recv()`가 `Err`를 받게 되어 프로그램을 종료할 수 있기 때문입니다. 

실행 결과는 다음과 같습니다.
```rust
got = 0
got = 2
got = 3
got = 1
got = 4
got = 5
got = 6
got = 7
got = 8
got = 9
```

 

위의 예시에서 `std::sync::recv`는 blocking function 입니다. 새로운 메시지를 받거나 `Err`를 받지 않는 한, spawn 된 future는 executor에서 다음 async task 수행을 blocking 할 수 있습니다. 즉 timing에 따라 deadlock을 발생시킬 수 있습니다.

문제가 발생할 만한 예시를 만들기 위해, tokio Runtime을 single-thread (`new_current_thread()`)로 생성하고, message send 하기 전에 3초 sleep 하도록 수정해 보았습니다.

```rust
fn main() {
    console_subscriber::init();

    let rt1 = tokio::runtime::Builder::new_current_thread()
                    .enable_all()
                    .build()
                    .unwrap();

    let (tx, rx) = mpsc::channel();

    for sender_id in 0..10 {
        let tx_channel = tx.clone();
        rt1.spawn(async move {
            tokio::time::sleep(tokio::time::Duration::from_secs(3)).await;
            tx_channel.send(sender_id).expect("Failed to send message");
        });
    }

    drop(tx);

    let handle = rt1.spawn(async move {
        while let Ok(i) = rx.recv() {
            println!("got = {}", i);
        }
    });

    rt1.block_on(async {
        handle.await;
    });
}
```

위 코드에서는 모든 tx들이 sleep하며 await 을 하는 동안 `rx.recv()`가 시작될 테고, 이 `recv()` 함수는 blocking 함수이므로 Runtime executor를 막고 있어서, tx를 갖고 있는 future들이 실행 될 기회를 주지 않습니다.

실행 하면 이전과 다르게 아무런 출력이 나오지 않습니다. 여기서 `tokio-console`을 사용하여 future들의 상태를 확인하면 다음과 같습니다. [tokio-console](https://tokio.rs/tokio/topics/tracing-next-steps) 사용법은 링크 참고하세요.

``` rust
connection: http://127.0.0.1:6669/ (CONNECTED)
views: t = tasks, r = resources
controls: left, right or h, l = select column (sort), up, down or k, j = scroll, enter = view details, i = invert sort
(highest/lowest), q = quit gg = scroll to top, G = scroll to bottom
Warnings
/!\ 1 tasks have lost their waker

Tasks (12) BUSY Running (1) IDLE Idle (11)
Warn  ID  State  Name  Total-     Busy       Idle       Polls  Target      Location
        9 IDLE          244.0032s 118.4160us  244.0031s 1      tokio::task src/bin/tokio-mpsc.rs:15:13
       10 IDLE          244.0031s  89.2080us  244.0030s 1      tokio::task src/bin/tokio-mpsc.rs:15:13
        6 IDLE          244.0031s  86.3330us  244.0030s 1      tokio::task src/bin/tokio-mpsc.rs:15:13
       11 IDLE          244.0030s  85.7080us  244.0029s 1      tokio::task src/bin/tokio-mpsc.rs:15:13
        2 IDLE          244.0029s  84.9580us  244.0028s 1      tokio::task src/bin/tokio-mpsc.rs:15:13
        3 IDLE          244.0029s  95.5410us  244.0028s 1      tokio::task src/bin/tokio-mpsc.rs:15:13
        8 IDLE          244.0028s  84.5830us  244.0027s 1      tokio::task src/bin/tokio-mpsc.rs:15:13
        1 IDLE          244.0028s  86.4160us  244.0027s 1      tokio::task src/bin/tokio-mpsc.rs:15:13
        7 IDLE          244.0027s  86.0410us  244.0026s 1      tokio::task src/bin/tokio-mpsc.rs:15:13
        5 IDLE          244.0026s  86.5830us  244.0025s 1      tokio::task src/bin/tokio-mpsc.rs:15:13
       12 BUSY          244.0026s  244.0015s   1.0940ms 1      tokio::task src/bin/tokio-mpsc.rs:23:22
! 1     4 IDLE          244.0025s  18.0410us  244.0025s 1      tokio::task <cargo>/tokio-1.20.1/src/runtime/mod.rs:477:2
```

모든 future들은 IDLE 상태인데, BUSY 상태인 future가 하나 있습니다. `tokio-mpsc.rs:23`에서 시작 한 future 인데, 이는 `recv()`함수가 포함된 future의 시작점입니다.


이와 같이 async rust에서 기존의 mpsc queue를 사용할 순 있지만 경우에 따라서 문제가 발생할 수 있습니다. 물론 `try_recv`를 사용하여 workaround 코드를 작성할 순 있겠으나 조금 복잡해 질 것 같습니다. Async rust에서는 async로 동작하는 mpsc queue를 사용하는 게 좋을 것 같습니다.


 
 

### 2. tokio::sync::mpsc 사용하기

`recv()` 동작이 executor를 blocking 하지 않도록 tokio의 async mpsc queue를 사용해 봅시다. 위의 코드에서 mpsc를 tokio 의 mpsc로 바꾸고, await 하도록 수정하면 됩니다. 그 외 API 사용법이 조금씩 다른 부분이 있으니 주의하세요.

```rust
use tokio::sync::mpsc;
const BUFFER_SIZE: usize = 128;

fn main() {
    console_subscriber::init();

    let rt1 = tokio::runtime::Builder::new_current_thread().enable_all().build().unwrap();

    let (tx, mut rx) = mpsc::channel(BUFFER_SIZE);

    for sender_id in 0..10 {
        let tx_channel = tx.clone();
        rt1.spawn(async move {
            tokio::time::sleep(tokio::time::Duration::from_secs(3)).await;
            tx_channel.send(sender_id).await.expect("Failed to send message");
        });
    }

    drop(tx);

    let handle = rt1.spawn(async move {
        while let Some(i) = rx.recv().await {
            println!("got = {}", i);
        }
    });

    rt1.block_on(async {
        handle.await;
    });
}

```

실행 시 다음과 같이 정상적인 결과가 나옵니다.
```rust
got = 0
got = 1
got = 2
got = 3
got = 4
got = 5
got = 6
got = 7
got = 8
got = 9
```

 
 

### 3. Tonic server의 async function에서 tx 공유하기

Tokio을 기반으로 하는 Tonic을 사용할 때 mpsc queue가 필요한 경우가 있습니다. gRPC server에서 받은 message를 message queue를 통해 다른 future에게 전달하고 싶을 때가 그 예 중 하나입니다.
 

Tonic의 [hello world](https://github.com/hyperium/tonic/tree/master/examples/src/helloworld) 예제에서 mpsc queue를 사용해 봅시다. Hello message를 받을 때 마다 이를 tokio의 다른 future에게 전달하도록 하였습니다.

```rust
use std::sync::mpsc;
use tonic::{transport::Server, Request, Response, Status};

use hello_world::greeter_server::{Greeter, GreeterServer};
use hello_world::{HelloReply, HelloRequest};

pub mod hello_world {
    tonic::include_proto!("helloworld");
}

pub struct MyGreeter {
    tx: mpsc::Sender<String>,
}
impl MyGreeter {
    pub fn new(tx: mpsc::Sender<String>) -> Self {
        Self {
            tx,
        }
    }
}
#[tonic::async_trait]
impl Greeter for MyGreeter {
    async fn say_hello(
        &self,
        request: Request<HelloRequest>,
    ) -> Result<Response<HelloReply>, Status> {
        println!("Got a request from {:?}", request.remote_addr());

        let name = request.into_inner().name;
        let reply = hello_world::HelloReply {
            message: format!("Hello {}!", name.clone()),
        };

        self.tx.send(name);
        Ok(Response::new(reply))
    }
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let (tx, rx) = mpsc::channel();

    let addr = "[::1]:50051".parse().unwrap();
    let greeter = MyGreeter::new(tx);

    println!("GreeterServer listening on {}", addr);

    tokio::spawn(async move {
        println!("START RECEIVING MESSAGES");
        while let Ok(msg) = rx.recv() {
            println!("received a message: {:?}", msg);
        }
    });

    Server::builder()
        .add_service(GreeterServer::new(greeter))
        .serve(addr)
        .await?;

    Ok(())
}
```

 

`std::sync::mpsc`를 사용하면, 이전과는 다르게 compile error부터 납니다. `std::sync::mpsc::Sender`가 Sync 를 구현하고 있지 않기 때문입니다. gRPC Server의 message handle 함수들은 여러 thread에서 동시에 수행될 수 있으므로(같은 gRPC가 여러 개 들어오면 한 번에 모두 처리 될 수 있음) `Sync` trait을 가질 것을 강제하고 있습니다. 
```rust
error[E0277]: `std::sync::mpsc::Sender<String>` cannot be shared between threads safely
   --> src/bin/tokio-mpsc.rs:55:6
    |
55  | impl Greeter for MyGreeter {
    |      ^^^^^^^ `std::sync::mpsc::Sender<String>` cannot be shared between threads safely
    |
    = help: within `MyGreeter`, the trait `Sync` is not implemented for `std::sync::mpsc::Sender<String>`
note: required because it appears within the type `MyGreeter`
   --> src/bin/tokio-mpsc.rs:44:12
    |
44  | pub struct MyGreeter {
    |            ^^^^^^^^^
note: required by a bound in `Greeter`
   --> /Users/huijeongkim/Workspace/rust_practice/target/debug/build/rust_practice-41089e7fe9e0adbb/out/helloworld.rs:111:31
    |
111 |     pub trait Greeter: Send + Sync + 'static {
    |                               ^^^^ required by this bound in `Greeter`

For more information about this error, try `rustc --explain E0277`.
error: could not compile `rust_practice` due to previous error
```

 

이 경우도 앞에서와 같이 `tokio::sync::mpsc`를 사용하는 것으로 해결할 수 있습니다. 수정한 코드는 아래와 같습니다.

```rust
use tokio::sync::mpsc;
const BUFFER_SIZE: usize = 128;

use tonic::{transport::Server, Request, Response, Status};

use hello_world::greeter_server::{Greeter, GreeterServer};
use hello_world::{HelloReply, HelloRequest};

pub mod hello_world {
    tonic::include_proto!("helloworld");
}

pub struct MyGreeter {
    tx: mpsc::Sender<String>,
}
impl MyGreeter {
    pub fn new(tx: mpsc::Sender<String>) -> Self {
        Self {
            tx,
        }
    }
}
#[tonic::async_trait]
impl Greeter for MyGreeter {
    async fn say_hello(
        &self,
        request: Request<HelloRequest>,
    ) -> Result<Response<HelloReply>, Status> {
        println!("Got a request from {:?}", request.remote_addr());

        let name = request.into_inner().name;
        let reply = hello_world::HelloReply {
            message: format!("Hello {}!", name.clone()),
        };

        self.tx.send(name).await;
        Ok(Response::new(reply))
    }
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let (tx, mut rx) = mpsc::channel(BUFFER_SIZE);

    let addr = "[::1]:50051".parse().unwrap();
    let greeter = MyGreeter::new(tx);

    println!("GreeterServer listening on {}", addr);

    tokio::spawn(async move {
        println!("START RECEIVING MESSAGES");
        while let Some(msg) = rx.recv().await {
            println!("received a message: {:?}", msg);
        }
    });

    Server::builder()
        .add_service(GreeterServer::new(greeter))
        .serve(addr)
        .await?;

    Ok(())
} 
```

 
 

`tokio::sync::mpsc`의 구현을 살펴 보면 이유를 알 수 있습니다. tokio mpsc queue의 `Sender` 구현은 아래와 같이 Arc와 Semaphore로 보호되어 있습니다.

```rust
// tokio::sync::mpsc 의 Sender
pub struct Sender<T> {
    chan: chan::Tx<T, Semaphore>,
}

// ..

// channel sender
pub(crate) struct Tx<T, S> {
    inner: Arc<Chan<T, S>>,
}
```


 
 

### 4. 결론

Async rust에서는 async mpsc 를 사용해야 한다는 당연한 얘기를 돌고 돌아 알게 되었습니다. 제 삽질기가 도움이 되었길 바랍니다....  


 
 


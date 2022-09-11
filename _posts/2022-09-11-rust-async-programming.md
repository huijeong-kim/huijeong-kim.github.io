---
layout: post
title: Rust Asynchronous Programming
tags: rust async runtime
---

오늘은 Rust에서 제공하는 Asynchronous Programming 관련 feature들에 대해 정리하면서, `async` 관련 포스트를 쓸 때 마다 사용되는 단어들, `async`, `future`, `runtime`, `executor`에 대해 정리해 보겠습니다. 오늘은 유독 내용이 추상적인 느낌에 부족한 부분이 많은 것 같은데, 틀린 부분이나 부족한 부분이 있다면 코멘트 남겨 주세요 :) 


 
 

## Asynchronous Programming
---

[Async book](https://rust-lang.github.io/async-book/01_getting_started/02_why_async.html)에는 Asynchronous programming이 다음과 같이 정의되어 있습니다.

 

> Asynchronous programming, or async for short, is a **<u>concurrent programming model</u>** supported by an increasing number of programming languages. It lets you run a large number of concurrent tasks on a small number of OS threads, while preserving much of the look and feel of ordinary synchronous programming, through the **<u>async/await syntax</u>**.

 

중요 포인트는 두 가지 일 것 같습니다.
- OS thread 수와 관계 없이 task들을 **<u>concurrent하게 실행</u>**시킬 수 있게 하는 **<u>프로그래밍 모델</u>**이다
- **<u>async/await syntax</u>**를 사용하여 기존의 synchronous programming과 비슷한 look & feel을 준다

 
 

## Concurrent Execution
---

Application의 task를 concurrent하게 실행하는 방법으로 가장 먼저 multi-threading을 떠올릴 수 있습니다. 가장 단순하게는, 병렬 실행할 task 마다 새로운 thread를 생성하고 그 thread에서 task를 실행시켜볼 수 있을 것입니다. [std::thread](https://doc.rust-lang.org/std/thread/)를 사용하면 native OS thread를 생성 및 시작할 수 있습니다.

하지만 OS thread 를 사용하여 task를 병렬 실행하면 OS 의 제약 내에서 thread를 사용해야 합니다. 단일 process 당 만들 수 있는 최대 thread 수가 제한되어 있고, thread 생성 시 발생하는 overhead, thread pool 관리 등 고려 해야 할 점이 많을 것 입니다.

 

많은 프로그래밍 언어에서는 `Green Thread`와 같은 OS thread 위에서 동작하는 virtual thread와 이 virtual thread를 실행시키는 환경, `Runtime`을 제공합니다. 이를 사용하면 OS thread의 제약 없이 task parallelism을 구현할 수 있고 개발자가 직접 코드에서 thread를 다루지 않아도 됩니다.

하지만 Rust의 경우, `Green Thread` 혹은 `Built-in Runtime`을 제공하지 않습니다. `Async/Await` syntax만 제공합니다. 이는 의도적인 디자인으로, 어플리케이션의 workload 에 따라 적합한(성능이 좋은) `Runtime`은 다를 수 있기 때문 입니다. Rust에는 다양한 `Runtime` Library들이 존재하고, 개발자는 이 중 application에 적합한 `Runtime`을 선택하여 async task를 실행시킬 수 있습니다.

 
 

## Async/Await
---

Rust 언어에서 제공하는 `Async/await` syntax에 대해 간단히 알아보고, 이 것이 `Runtime`에서 실행 되는 방식을 이어서 알아 보겠습니다.

The Book에서 정의한 관련 키워드들은 다음과 같습니다.

- `async` - return a Future instead of blocking the current thread
- `await` - suspend execution until the result of a Future is ready

`async`는 `Future`를 리턴하는 asynchronous 동작(현재 thread를 blocking하지 않는 동작)을 의미하고, `Future`는 `Runtime`에서 실행될 수 있는 trait 입니다. `async` 키워드를 사용해 function, block expression, closure를 `Future`로 만들 수 있습니다. `Future` trait이 `Runtime`에서 어떻게 사용 되는지는 뒤에서 간단히 설명하겠습니다.


다음은 async func, async block, async closure를 만들어 실행시키는 간단한 코드입니다. 기존의 function, block, closure에 `async` 키워드를 추가하였고, 이를 실행시킬 `Runtime`으로는 `futures` crate를 사용했습니다. 각 `Future`들은 `futures::executor::block_on`에 의해 실행됩니다.

```rust
use futures::executor;
async fn async_func() {
    println!("async_func");
}
fn main() {
    let future1 = async_func();
    let future2 = async {
        println!("async block");
    };
    let async_closure = || async {
        println!("async closure");
    };
    let future3 = async_closure();

    executor::block_on(future3);
    executor::block_on(future2);
    executor::block_on(future1);
}
```

출력 결과
```cpp
async closure
async block
async_func
```

 

`await`는 Future의 동작이 완료되기를 기다릴 때 사용합니다. `async` 안에서만 사용될 수 있습니다. 다음은 3개의 task, 1) learn_song, 2) sing_song, 3) dance가 주어졌을 때, sing_song과 dance는 learn_song이 완료된 후에 실행되도록 하는 예제 코드입니다. sing_song, dance를 실행시키기 전에 `learn_song().await`를 사용하여 동작 완료를 기다립니다.

![async-await]({{site.baseurl}}/assets/images/image04.png)

```rust 
struct Song(String);

async fn learn_song() -> Song {
    println!("Learn song!");
    Song(String::from("Hype boy"))
}

async fn dance(song: &Song) {
    println!("Dance to {}!!", song.0);
}

async fn sing_song(song: &Song) {
    println!("Sing a song!! {}", song.0)
}

fn main() {
    futures::executor::block_on(async {
        let song = learn_song().await;

        let f1 = sing_song(&song);
        let f2 = dance(&song);

        futures::join!(f1, f2);
    });
}
```

실행 결과
```cpp
Learn song!
Sing a song!! Hype boy
Dance to Hype boy!!
```

 

`await`을 사용하여 <u>'기다리는'</u> 동작은 현재 thread를 blocking 하지 않고 동작을 yield 하는 방식으로 이루어집니다. 위의 예제에서는 그 동작이 잘 보이지 않을 수 있지만, I/O intensive application을 생각해 보면 이해하기 쉬울 것 입니다.

다음 예제는 rust의 대표적인 `Runtime`인 `tokio`를 사용한 TCP I/O 예시입니다. tokio runtime 생성 시  `new_current_thread`을 사용하여 현재 thread(단일 thread)만 사용하도록 했습니다. 단일 thread에서 Server 동작과 Client 동작 둘 다를 실행하고 있지만, Server 동작을 하는 async block에서 TcpListner가 `accept()`를 기다리는(await) 동안 thread를 blocking 하지 않고 동작을 yield 하므로 Client 동작의 async block에서 불리는 `connect()`가 실행될 수 있습니다.   

```rust
use tokio::io::{self, AsyncReadExt, AsyncWriteExt};
use tokio::net::{TcpStream, TcpListener};

fn main() {
    tokio::runtime::Builder::new_current_thread()
        .enable_all()
        .build()
        .unwrap()
        .block_on(async {
            let addr = "127.0.0.1:6142";
            
            let server = tokio::spawn(async move {
                let listener = TcpListener::bind(&addr).await?;
    
                loop {
                    let (mut socket, _) = listener.accept().await?;
                    let mut buf = vec![0; 1024];
                    match socket.read(&mut buf).await {
                        Ok(n) => { println!("{:?}", buf); }
                        _ => { return Ok::<_, io::Error>(()); }
                    }
                }
                Ok::<_, io::Error>(())
            });
    
            let client = tokio::spawn(async move {
                let socket = TcpStream::connect(&addr).await?;
                let (mut rd, mut wr) = io::split(socket);
    
                wr.write_all(b"hello\r\n").await?;
                wr.write_all(b"world\r\n").await?;
    
                Ok::<_, io::Error>(())
            });
            
            tokio::join!(server, client);
        });
}
```

이 예제에서 사용한 TcpStream, TcpListener 등은 std crate가 아닌 tokio crate에 구현된 async 버전입니다. [이전 글](https://huijeong-kim.github.io/2022/08/27/tokio-mpsc/)에서 본 것과 같이, async rust를 사용하는 경우 synchronous하게 구현 된 standard library 대신 async 로 구현 된 library를 사용 해야 합니다. 


 
 


## Runtime
---

`Runtime`은 `Future`를 실행할 수 있는 환경입니다. `async`를 실행시키는 주체이므로 `Async Executor`라고도 부르는 것 같습니다. `Runtime` 혹은 `Executor`는 `async`를 실행시킵니다. 만약 `async`가 완료될 수 없는 상태라면 추후 실행 가능할 때 재 실행하고, 그 동안 다른 실행 가능한 `async`를 실행시켜 줍니다. 일종의 non-preemptive async task scheduler 인 것 같습니다.

 

```rust 
// Future trait
pub trait Future {
    type Output;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}

// poll 함수의 결과값 enum
pub enum Poll<T> {
    // Represents that a value is immediately ready.
    #[lang = "Ready"]
    #[stable(feature = "futures_api", since = "1.36.0")]
    Ready(#[stable(feature = "futures_api", since = "1.36.0")] T),

    // Represents that a value is not ready yet.
    //
    // When a function returns `Pending`, the function *must* also
    // ensure that the current task is scheduled to be awoken when
    // progress can be made.
    #[lang = "Pending"]
    #[stable(feature = "futures_api", since = "1.36.0")]
    Pending,
} 
```
`async`를 실행시키는 것은, `async`가 구현하고 있는 `Future` trait의 `poll` 함수를 호출하는 것을 의미합니다. async function/block/closure는 `Future` trait, 즉 `poll` 함수를 구현하고 있습니다. `poll` 함수가 불리면 `async`에 구현된 동작을 실행시키는데, `async`의 모든 라인이 실행 되었다면 `Poll::Ready`를 리턴하며 동작 완료합니다. 하지만 다른 일을 기다리는 등의 이유(e.g., I/O)로 `async`가 바로 완료될 수 없다면 `Poll::Pending`을 리턴하며 추후 재 실행되기를 기대합니다.

 

``` rust
// Future trait 실행 시 주어지는 Context. Waker는 virtual function table을 갖고 있다
pub struct Context<'a> {
    waker: &'a Waker,
    // Ensure we future-proof against variance changes by forcing
    // the lifetime to be invariant (argument-position lifetimes
    // are contravariant while return-position lifetimes are
    // covariant).
    _marker: PhantomData<fn(&'a ()) -> &'a ()>,
}
```
`poll` 함수의 인자로 주어지는 `Context`에는 `Waker`가 포함되어 있습니다. `Waker`는 `Poll::Pending` 시 기다리던 동작이 완료되면 이를 executor에게 알려주는 함수입니다(함수를 갖고 있습니다). `Waker` 는 executor-specific 합니다. `Context`는 `async`가 어디 까지 실행 되었는지를 알 수 있는 정보 또한 포함 하고 있습니다. `Poll::Pending`을 리턴하게 된 지점을 기억하고, `Waker`에 의해 재 실행된 경우 그 지점 부터 재 실행 합니다. 

async function/block/closure 구현이 어떻게 `Future::poll`로 변환(?) 되는지 궁금한데 이 부분은 아직 코드 혹은 문서로 확인하지 못했어요. 기존 `async` 코드에서 바로 완료될 수 없는 함수를 만난 경우 혹은 코드 내에서 호출한 `async`의 `poll`도 `Poll::Pending`을 리턴하는 경우 `Poll::Pending` 을 리턴할 것 같다는 추측만 해 봅니다 ㅎㅎ

이러한 polling 기반의 `async` 동작은 [zero-cost futures](https://blog.rust-lang.org/2019/11/07/Async-await-stable.html#zero-cost-futures)를 가능하게 한다고 합니다!

 

`Runtime`의 spawn이나 block_on과 같은 함수를 사용하면 `async`를 실행시킬 수 있습니다. 첫 번째 예제에서 사용한 futures crate의 `futures::executor::block_on`이 그러한 함수 중 하나 입니다. `Runtime`이 `async`를 실행 했을 때(=`poll`을 호출 했을 때), 그 결과가 `Poll::Ready(val)`인 경우 `async`는 실행 완료된 것이고, val를 리턴할 겁니다. 하지만 결과가 `Poll::Pending`이라면 어딘가에 담아 두었다, `Waker`가 호출된 후 재 실행할 것 입니다. 간단한 `Runtime` 구현은 async book의 [Build an Executor](https://rust-lang.github.io/async-book/02_execution/04_executor.html)를 참고하면 되겠습니다.

 

Rust에서 널리 사용되는 [Async Runtime](https://rust-lang.github.io/async-book/08_ecosystem/00_chapter.html?highlight=tokio#popular-async-runtimes) 들은 아래와 같습니다. tokio의 경우, std의 synchronous 동작을 asynchronous 버전(async fn)으로 구현한 버전도 제공합니다.

- **Tokio**: A popular async ecosystem with HTTP, gRPC, and tracing frameworks.
- **async-std**: A crate that provides asynchronous counterparts to standard library components.
- **smol**: A small, simplified async runtime. Provides the Async trait that can be used to wrap structs like UnixStream or TcpListener.
- **fuchsia-async**: An executor for use in the Fuchsia OS.


 
 


### References
---
- [Rust Async Book](https://rust-lang.github.io/async-book/01_getting_started/02_why_async.html)
- [std thread](https://doc.rust-lang.org/std/thread/index.html)
- [std Future](https://doc.rust-lang.org/std/future/trait.Future.html)
- [tokio io](https://tokio.rs/tokio/tutorial/io)


 
 

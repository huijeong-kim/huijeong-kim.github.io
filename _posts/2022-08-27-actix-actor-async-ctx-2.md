---
layout: post
title: Actix actor의 AsyncContext 활용하기 (2)
tags: rust actix actor
---

[지난 글](https://huijeong-kim.github.io/2022/08/21/actix-actor-async-ctx-1/)에 이어 Actix actor의 `AsyncContext` 함수를 사용하는 방법을 알아봅니다.

 
 

### add_stream, add_message_stream
```rust
fn add_stream<S>(&mut self, fut: S) -> SpawnHandle
where
    S: Stream + 'static,
    A: StreamHandler<S::Item>;

fn add_message_stream<S>(&mut self, fut: S)
    where
        S: Stream + 'static,
        S::Item: Message,
        A: Handler<S::Item>;
```

Context에 stream을 등록하는 함수입니다. 두 함수는 거의 비슷하나 차이점은 `add_message_stream`은 stream error를 무시한다는 점입니다.

다음은 [API Doc](https://docs.rs/actix/0.13.0/actix/prelude/trait.AsyncContext.html#method.add_stream)의 예제인데요. Stream을 만들고 message를 하나 보냅니다. `Stream`의 사용법과 예제는 다음 글에서 따로 정리해 보겠습니다 :) 

```rust
use actix::prelude::*;
use futures_util::stream::once;

#[derive(Message)]
#[rtype(result = "()")]
struct Ping;

struct MyActor;

impl StreamHandler<Ping> for MyActor {

    fn handle(&mut self, item: Ping, ctx: &mut Context<MyActor>) {
        println!("PING");
        System::current().stop();
    }

    fn finished(&mut self, ctx: &mut Self::Context) {
        println!("finished");
    }
}

impl Actor for MyActor {
    type Context = Context<Self>;

    fn started(&mut self, ctx: &mut Context<Self>) {
        // add stream
        ctx.add_stream(once(async { Ping }));
    }
}

fn main() {
    let mut sys = System::new();
    let addr = sys.block_on(async { MyActor.start() });
    sys.run();
}
```


실행 결과는 다음과 같습니다.

```cpp
PING
finished
```

 

--- 

### notify, notify_later
```rust
fn notify<M>(&mut self, msg: M)
where
    A: Handler<M>,
    M: Message + 'static;

fn notify_later<M>(&mut self, msg: M, after: Duration) -> SpawnHandle
    where
        A: Handler<M>,
        M: Message + 'static;
```

`notify`는 자기 자신에게 message를 보내는 함수입니다. 이전 글에서 `address`를 통해 자기 자신의 주소를 찾고, 여기에 message를 보내는 예제를 작성하였는데요. `notify` 함수로 바로 message handler를 부를 수도 있습니다.
이 함수는 actor의 <u>mailbox를 bypass</u> 하고 항상 message를 전달한다고 합니다. 하지만 actor가 `Stopped` 상태라면 error 발생합니다.

이전에 자신의 `Addr`를 찾아 메시지를 보내는 예제를 `notify` 함수를 사용하도록 수정하면 다음과 같습니다.

```rust
fn send_message_to_myself_using_address(&mut self, ctx: &mut Context<Self>) {
    let my_address = ctx.address();
    actix::spawn(async move {
        my_address.send(Message).await;
    });
}

fn send_message_to_myself_using_notify(&mut self, ctx: &mut Context<Self>) {
    ctx.notify(Message);
}
```
 

`notify_later`은 `notify` 함수와 유사하지만, 지정된 시간 이후에 message를 보냅니다. 또 하나의 차이점은 `SpawnHandle`을 리턴한다는 것 인데요. 실행되기 전에는 `cancel_future` 통해 동작을 취소할 수 있습니다. 이전 글에서 언급한 바와 같이, `cancel_future` 함수의 리턴 값은 항상 true 입니다. 취소 성공 여부를 리턴하지 않음을 참고하세요.

```rust
fn send_message_to_myself_and_cancel(&mut self, ctx: &mut Context<Self>) {
    let handle = ctx.notify(Message, Duration::from_secs(3));
    let _success = ctx.cancel_future(handle);
}
```

 

--- 

### run_later, run_interval
```rust
fn run_later<F>(&mut self, dur: Duration, f: F) -> SpawnHandle
where
    F: FnOnce(&mut A, &mut A::Context) + 'static;

fn run_interval<F>(&mut self, dur: Duration, f: F) -> SpawnHandle
    where
        F: FnMut(&mut A, &mut A::Context) + 'static;
```


`run_later`은 주어진 시간 이후에 주어진 future를 실행시키고, `run_interval`은 주어진 시간 주기로 future를 실행시킵니다. 사용법은 거의 동일하고, `run_interval`의 사용 예제는 [이 글](https://huijeong-kim.github.io/2022/08/17/actix-actor-usage/)을 참고하시면 되겠습니다.



 
--- 


### References
[https://docs.rs/actix/0.13.0/actix/trait.AsyncContext.html](https://docs.rs/actix/0.13.0/actix/trait.AsyncContext.html)

 
 
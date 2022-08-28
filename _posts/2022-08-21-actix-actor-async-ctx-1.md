---
layout: post
title: Actix actor의 AsyncContext 활용하기 (1)
tags: rust actix actor
---

Actix에서 actor를 생성하면 해당 actor를 실행시키는 문맥(context)에 해당하는 `Context`가 생성 되고, `Context`는 다양한 actor의 동작을 정의합니다. 그 중 `Context`를 활용해서 actor가 future를 실행, 취소, 기다리는 방법을 알아봅니다. 아래 예제 코드들은 actix 0.13.0 버전을 사용했습니다.

 
 

### Actor Context
먼저, Actor의 `Context`의 정의, 생성 시기를 알아보겠습니다. 

사용자가 정의한 struct를 actor로 만들기 위해서 최소로 필요한 것은 Actor trait을 구현(implement) 하고 Context를 정의하는 것 입니다.
```rust
struct MyActor;

impl Actor for MyActor {
    type Context = Context<Self>;
}
```

위 actor는 다음과 같이 생성되고 시작될 수 있습니다.
```rust
let actor = MyActor;
let addr = actor.start();
```

`actor.start()`는 앞에서 정의한 `Actor` trait에 구현되어 있습니다. 다음은 `Actor`에 구현된 `start` 함수입니다. Actor가 시작되면, Actor가 실행 될 `Context`가 생성되고 실행됩니다.
```rust
fn start(self) -> Addr<Self>
    where
        Self: Actor<Context = Context<Self>>,
    {
        Context::new().run(self)
    }
```

 

`Context`는 mailbox, future handles 등 actor가 실행되는 문맥을 갖고 있습니다. 그리고 이 `Context`는 `AsyncContext`라는 trait를 구현하고 있는데요, 이를 활용하면 actor가 실행해야 하는 async task들을 다룰 수 있습니다.

`AsyncContext`가 정의하는 함수들은 아래와 같습니다. 함수 목록만 보기 위해 trait에 구현된 내용은 생략하였습니다. 예제를 통해 각 함수들을 사용하는 방법을 알아보겠습니다.

```rust
pub trait AsyncContext<A>: ActorContext
where
    A: Actor<Context = Self>,
{
    fn address(&self) -> Addr<A>;

    fn spawn<F>(&mut self, fut: F) -> SpawnHandle
    where
        F: ActorFuture<A, Output = ()> + 'static;

    fn wait<F>(&mut self, fut: F)
    where
        F: ActorFuture<A, Output = ()> + 'static;

    fn waiting(&self) -> bool;

    fn cancel_future(&mut self, handle: SpawnHandle) -> bool;

    fn add_stream<S>(&mut self, fut: S) -> SpawnHandle
    where
        S: Stream + 'static,
        A: StreamHandler<S::Item>;

    fn add_message_stream<S>(&mut self, fut: S)
    where
        S: Stream + 'static,
        S::Item: Message,
        A: Handler<S::Item>;

    fn notify<M>(&mut self, msg: M)
    where
        A: Handler<M>,
        M: Message + 'static;

    fn notify_later<M>(&mut self, msg: M, after: Duration) -> SpawnHandle
    where
        A: Handler<M>,
        M: Message + 'static;

    fn run_later<F>(&mut self, dur: Duration, f: F) -> SpawnHandle
    where
        F: FnOnce(&mut A, &mut A::Context) + 'static;

    fn run_interval<F>(&mut self, dur: Duration, f: F) -> SpawnHandle
    where
        F: FnMut(&mut A, &mut A::Context) + 'static;
}
```

 
 

---
### address
```rust
fn address(&self) -> Addr<A>;
```
현재 Actor의 `Addr`를 리턴합니다. `Addr`는 해당 Actor에게 message 를 보내는 channel과 같습니다. 어떤 경우에 사용 가능할까 고민을 해 본다면,,


- **자기 자신에게 message를 보내야 할 때**: 아래 예제와 같이 자기 자신에게 message를 보내야 한다면 사용할 수 있겠습니다. 하지만 다음에 설명 될 `notify`, `run_later` 등의 함수가 있어 사용하게 될 지 모르겠습니다.

```rust
fn send_message_to_myself(&mut self, ctx: &mut Context<Self>) {
    let my_address = ctx.address();
    actix::spawn(async move {
        my_address.send(Message).await;    
    });
}
```

- **자기 자신의 주소가 바뀐 것을 알려줘야 할 때**: 예를 들어 message 처리 중 `self.clone()` 한 뒤 바뀐 `Addr`를 사용자에게 알려줘야 하는 경우엔 사용할 수 있을 것 같긴 하네요. 자주 쓰일지는 모르겠지만요..


- **Message에 응답 받을 주소를 기재할 때**: Message를 보낼 때 상대 actor에게 답장을 받을 주소, 즉 자신의 `Addr`를 전달할 수도 있겠습니다. [다음](https://riker.rs/actors/#sending-messages)은 또 다른 actor framework인 riker의 message handling 함수인데요. Actor는 Message를 받을 때 sender의 주소도 받게 됩니다. 이는 actor가 앞으로 message를 보내야 하는 모든 상대의 `Addr`를 미리 알 필요 없게 해 주고, 답장 받을 주소를 바꾸기 쉽게 하여 unit test 작성하기 쉽게 해 줍니다. 


```rust
// Riker implement the Actor trait
impl Actor for MyActor {
    type Msg = String;

    fn recv(&mut self,
            _ctx: &Context<String>,
            msg: String,
            _sender: Sender) {

        println!("Received: {}", msg);
    }
}
```

 

--- 

### spawn
```rust 
fn spawn<F>(&mut self, fut: F) -> SpawnHandle
    where
        F: ActorFuture<A, Output = ()> + 'static;
```
주어진 future를 현재 `Context`에 spawn 합니다. 리턴 된 `SpawnHandle`은 아래에서 설명할 `cancel_future` 함수 등에 활용될 수 있습니다.
Actor가 stopping 상태에 진입되면 spawn 된 함수들은 모두 cancel 됩니다. 

바로 위, 자기 자신에게 message를 보내는 코드에서 message 보내는 future를 `actix::spawn()`을 사용해 시작했는데요, 이를 `Context::spawn()`을 사용하여 시작할 수도 있습니다. Actor가 멈추게 된다면 spawn 된 future들도 함께 멈추므로, 이미 멈춘 actor에게 message를 보내는 일이 없도록 `Context::spawn()`을 사용하는 게 더 안전할 것 같습니다.

여기서 future의 type은 `ActorFuture` 입니다. `into_actor`를 통해 future를 actor의 context를 담은 `ActorFuture`로 변환할 수 있습니다.

```rust
fn send_message_to_myself(&mut self, ctx: &mut Context<Self>) {
    let my_address = ctx.address();
    ctx.spawn(async {
        my_address.send(Message).await;    
    }.into_actor(self));
}
```

 

--- 

### wait
```rust
fn wait<F>(&mut self, fut: F)
where
    F: ActorFuture<A, Output = ()> + 'static; 
```
future를 spawn 하고 이 것이 "resolve" 될 때 까지 기다립니다. 즉, async 함수의 await이 끝날 때 까지 기다립니다. 그 동안 actor는 새로운 message를 받을 수 없습니다.

```rust
fn wait_future(&mut self, ctx: &mut Context<Self>) {
    tracing::info!("Before wait ");
    ctx.wait(async {
        tracing::info!("async block started...");
        tokio::time::sleep(tokio::time::Duration::from_secs(3)).await;
        tracing::info!("async block ended...");
    }.into_actor(self));
    tracing::info!("After wait ");
}
```

위와 같은 코드를 실행 해 보면 결과는 아래와 같습니다. `wait`이 호출 되면 async block 이 끝날 때 까지 기다릴 것 같아 보이지만, 이 함수 호출에서 resolve를 기다리는 것은 아니고 **<u>함수는 바로 종료됩니다</u>**. 다만 context가 wait 합니다. 이 부분은 바로 다음에서 설명하겠습니다.  

```cpp
2022-08-20T12:55:48.307811Z  INFO actix_context_usage: actor started
2022-08-20T12:55:48.307916Z  INFO actix_context_usage: Before wait 
2022-08-20T12:55:48.307946Z  INFO actix_context_usage: After wait 
2022-08-20T12:55:48.307973Z  INFO actix_context_usage: async block started...
2022-08-20T12:55:51.310184Z  INFO actix_context_usage: async block ended...
```

 

--- 

### waiting
```rust 
fn waiting(&self) -> bool;
```

현재 context가 waiting(paused) 상태인지 알려줍니다. 위에서 설명한 `Context::wait()` 함수를 통해 시작한 future가 실행되고 있는 경우, context는 해당 future가 끝나기를 기다리며 다음 future를 실행하지 않는 pause 상태가 됩니다. `Context::waiting()` 함수는 이 상태를 확인하는 용도입니다. 


다음은 바로 위에서 작성한 함수에 waiting을 확인하여 출력하도록 수정한 코드입니다.

```rust
use actix::{Actor, AsyncContext, Context, Handler, Message, System, WrapFuture};
use tracing_subscriber::FmtSubscriber;

struct MyActor;

impl Actor for MyActor {
    type Context = Context<Self>;

    fn started(&mut self, ctx: &mut Context<Self>) {
        self.wait_future(ctx);
    }
}
impl MyActor {
    fn wait_future(&mut self, ctx: &mut Context<Self>) {
        tracing::info!("Before wait ");
        tracing::info!("is waiting: {}", ctx.waiting());
        ctx.wait(async {
            tracing::info!("async block started...");
            tokio::time::sleep(tokio::time::Duration::from_secs(3)).await;
            tracing::info!("async block ended...");
        }.into_actor(self));
        tracing::info!("After wait ");
        tracing::info!("is waiting: {}", ctx.waiting());
    }
}

#[derive(Message)]
#[rtype(result="bool")]
struct IsWaiting;

impl Handler<IsWaiting> for MyActor {
    type Result = bool;

    fn handle(&mut self, _msg: IsWaiting, ctx: &mut Context<Self>) -> Self::Result {
        ctx.waiting()
    }
}

fn main() {
    let system = System::new();
    system.block_on(async {
        let subscriber = FmtSubscriber::new();
        tracing::subscriber::set_global_default(subscriber);

        let addr = MyActor.start();
        tracing::info!("actor started");
        
        tracing::info!("check if actor is waiting");
        let is_waiting = addr.send(IsWaiting).await.unwrap();
        tracing::info!("is actor waiting: {}", is_waiting);
    });
    system.run();
}
```

실행 결과는 다음과 같습니다. `Context::wait()`이 시작된 이후로 waiting은 false 입니다. `Context::wait()`이 불린 순간부터 `waiting()`이 true가 됩니다. waiting 상태가 된 후에는 `IsWaiting` 메시지는 처리되지 않습니다. 이후 async block(future)이 완료된 후 `waiting()`이 false가 되고, `IsWaiting` 메시지에 응답합니다.

```cpp
2022-08-20T12:56:07.653870Z  INFO actix_context_usage: actor started
2022-08-20T12:56:07.653898Z  INFO actix_context_usage: check if actor is waiting
2022-08-20T12:56:07.653915Z  INFO actix_context_usage: Before wait 
2022-08-20T12:56:07.653922Z  INFO actix_context_usage: is waiting: false
2022-08-20T12:56:07.653929Z  INFO actix_context_usage: After wait 
2022-08-20T12:56:07.653935Z  INFO actix_context_usage: is waiting: true
2022-08-20T12:56:07.653942Z  INFO actix_context_usage: async block started...
2022-08-20T12:56:10.656440Z  INFO actix_context_usage: async block ended...
2022-08-20T12:56:10.656739Z  INFO actix_context_usage: is actor waiting: false
```

![timeline]({{site.baseurl}}/assets/images/image02.png)

 

--- 

### cancel_future
```rust
fn cancel_future(&mut self, handle: SpawnHandle) -> bool;
```

spawn 된 future를 취소하는 함수입니다. actix repo의 `contxtimpl.rs`를 보면, 리턴 값은 항상 true인 것으로 보입니다.
```rust
pub fn cancel_future(&mut self, handle: SpawnHandle) -> bool {
    self.handles.push(handle);
    true
}
```

다음은 actor가 시작하면 두 개의 future를 spawn 하고, 이후 spawn 된 future들을 취소하는 예제입니다. 아주 단순하게, 두 future는 각각 1초, 5초를 sleep 하고, main 에서는 actor 시작 3초 후에 cancel을 시도합니다.

![timeline]({{site.baseurl}}/assets/images/image03.png)

```rust
impl MyActor {
    fn spawn_futures(&mut self, ctx: &mut Context<Self>) {
        let handle1 = ctx.spawn(async move { // Future 01
            tracing::info!("async block 1 started...");
            tokio::time::sleep(tokio::time::Duration::from_secs(1)).await;
            tracing::info!("async block 1 ended...");
        }.into_actor(self));

        let handle2 = ctx.spawn(async move { // Future 02
            tracing::info!("async block 2 started...");
            tokio::time::sleep(tokio::time::Duration::from_secs(5)).await;
            tracing::info!("async block 2 ended...");
        }.into_actor(self));

        self.spawned_tasks.push(handle1);
        self.spawned_tasks.push(handle2);
    }
}

#[derive(Message)]
#[rtype(result="()")]
struct CancelAllFutures;

impl Handler<CancelAllFutures> for MyActor {
    type Result = ();

    fn handle(&mut self, _msg: CancelAllFutures, ctx: &mut Context<Self>) -> Self::Result {
        for h in &self.spawned_tasks {
            let success = ctx.cancel_future(*h);
            tracing::info!("Cancel handle success: {}", success);
        }
    }
}

fn main() {
    let system = System::new();
    system.block_on(async {
        let subscriber = FmtSubscriber::new();
        tracing::subscriber::set_global_default(subscriber);

        let addr = MyActor::default().start();
        tracing::info!("actor started");

        tokio::time::sleep(tokio::time::Duration::from_secs(3)).await;
        addr.send(CancelAllFutures).await.unwrap();
    });
    system.run();

}
```


결과는 다음과 같습니다. actor가 시작하면서 두 개의 future가 모두 시작되고, 1초만 sleep 하는 첫 번째 future는 바로 완료 됩니다. 이후 모든 future들을 취소 했고, **취소 함수의 리턴 값은 모두 true입니다**. 앞에 설명한 것 처럼, 항상 true로 리턴하는 것으로 보입니다. 하지만 로그에서 볼 수 있듯 첫 번째 future는 이미 끝나 취소되지 않았고 두 번째 future는 sleep 중에 취소된 것으로 보입니다.

```cpp
2022-08-21T09:09:08.685024Z  INFO actix_context_usage: actor started
2022-08-21T09:09:08.685072Z  INFO actix_context_usage: async block 1 started...
2022-08-21T09:09:08.685085Z  INFO actix_context_usage: async block 2 started...
2022-08-21T09:09:09.687806Z  INFO actix_context_usage: async block 1 ended...
2022-08-21T09:09:11.686872Z  INFO actix_context_usage: Cancel handle success: true
2022-08-21T09:09:11.687038Z  INFO actix_context_usage: Cancel handle success: true
```

 
 

남은 함수들은 다음 편에서 살펴보겠습니다.

 
 

### References

- [https://github.com/actix/actix/blob/master/actix/src/actor.rs](https://github.com/actix/actix/blob/master/actix/src/actor.rs)
- [https://github.com/actix/actix/blob/master/actix/src/context.rs](https://github.com/actix/actix/blob/master/actix/src/context.rs)


 
 


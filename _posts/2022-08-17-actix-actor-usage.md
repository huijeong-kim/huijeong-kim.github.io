---
layout: post
title: Actix actor를 사용하는 두 가지 방법
tags: rust actix actor
---

Actix에서 actor를 사용하는 방법은 두 가지로 나눌 수 있을 것 같습니다. 첫 번째는 특정 task를 실행시키는 용도로, client가 message를 보내면 이를 받아서 처리하는 actor를 만드는 것 이고, 두 번째는 주기적으로 task를 실행시키는 actor를 만드는 것 입니다. 두 경우 각각 actor를 사용하는 방법에 대해 알아봅니다.  

 
 

### Message를 처리하는 actor

Actor를 시작하면 `Addr`가 리턴됩니다. `Addr`는 해당 actor에게 message를 전달할 수 있는 channel 입니다. Client module은 이 `addr`를 통해서 message를 전달할 수 있습니다.

```rust
let addr = MyActor::new().start();
addr.send(Ping(10)).await?;
```


위 예제는 actor를 생성, 시작한 뒤 actor에게 메시지를 보내는 코드입니다. 이 때 Actor는 시작되면서 `Started` 상태가 됩니다. `Started` 상태에선 `Actor::started` method가 불린 뒤 `Running` 상태, message 를 처리할 수 있는 상태가 됩니다. 이후 Actor는 다음 중 한 가지 조건이 만족되면 `Stopping` 상태가 되고 종료됩니다.
- Actor 자신이 `Context::stop` 을 불렀을 때
- Actor의 모든 `Addr`가 drop 되었을 때
- Context에 등록 된 evented object가 없을 때 

특정 Message를 처리하는 용도로 actor를 사용할 땐, 의도적으로 `Context::stop`을 호출했을 때 혹은 `Addr`를 사용하는 client가 없을 때 actor는 멈추게 됩니다. 즉 actor를 멈추기 위해선 `Context::stop`을 호출하거나, 사용 중인 `Addr`를 모두 drop하면 됩니다. 참고로 Actor를 시작한 후 리턴 된 `Addr`를 사용하지 않는다면(다른 variable에 bind하지 않는다면) actor는 `Actor::started` 실행 후 바로 `Stopping` 상태가 될 수 있습니다.

세번째 경우는 아래에서 설명됩니다.

 
 

### 주기적으로 task를 실행하는 actor

주기적인 task를 실행하는 actor의 대표적인 예는 자기 자신 혹은 외부의 상태를 주기적으로 확인 하는 heartbeat이 있습니다.

이를 가장 naive하게 구현하면, 특정 주기마다 actor에게 message를 보내 task를 수행하게 할 수 있습니다. 아래는 actix repo의 [ping example](https://github.com/actix/actix/blob/master/actix/examples/ping.rs)을 주기적으로 ping을 보내도록 수정한 코드입니다. main 함수에서 특정 주기로 ping message를 보내도록 했습니다.

여기서 주의할 점은, `std::thread::sleep`이 아닌 `tokio::time::sleep`을 사용했다는 점 입니다. `std::thread::sleep`을 통해 sleep하는 경우, 현재 thread가 sleep 상태에 빠져 Actix System이 다른 future들을 실행시킬 수 없습니다. Single thread를 사용하는 기본 Actix System의 경우, application 전체가 block 상태가 됩니다. `tokio::time::sleep`은 현재 thread가 아닌 현재 future만 sleep 시켜, 전체 system을 block시키지 않습니다. 
```rust
fn main() {
    System::new().block_on(async {
        let addr = MyActor { count: 10 }.start();
        loop {
            let res = addr.send(Ping(10)).await;
            println!("RESULT: {}", res.unwrap());

            tokio::time::sleep(tokio::time::Duration::from_secs(3)).await;
        }
    });
}
```


 

하지만 Actix에는 이 경우 사용할 수 있는 `run_interval` 함수가 있습니다. 다음과 같이 Actor의  `started` 함수에 `run_interval`을 통해 주기적으로 실행시킬 함수를 등록해 놓는다면, 위와 같은 loop를 작성하지 않아도 됩니다.
```rust
fn started(&mut self, ctx: &mut Self::Context) {
    println!("Actor started");
    
    ctx.run_interval(std::time::Duration::from_secs(3), |_, _| {
        MyActor::do_something();
    });
}
```

이 경우 Actor의 `Addr`가 사용되지 않아도 actor는 멈추지 않습니다. 앞에서 언급한 Actor가 `Stopping` 상태가 되는 조건 중 세 번째를 충족시키지 않기 때문, 즉 evented object 가 등록된 상태이기 때문입니다. 


`run_interval` 함수를 조금 더 살펴봅시다. 함수는 다음과 같이 실행 주기에 해당하는 `Duration`과 실행시킬 함수, `f`를 인자로 받습니다. 
`f`는 `FnMut(&mut A, &mut A::Context) + 'static` 을 만족해야 합니다. 여기서  `A`는 `Actor<Context=Self>` 입니다. 결국 `f`는 `Actor`와 `Actor::Context`를 인자로 받는 `FnMut`여야 합니다. 

위의 예제에서 `|_, _| {}`로 구현된 closure가 이 타입이며, 사용되지 않은 두 인자는 `Actor`와 `Actor::Context` 입니다.

```rust
fn run_interval<F>(&mut self, dur: Duration, f: F) -> SpawnHandle
where
    F: FnMut(&mut A, &mut A::Context) + 'static,
{
    self.spawn(IntervalFunc::new(dur, f).finish())
}
```

 
 

이를 활용하여 다양한 함수를 `run_interval`로 등록할 수 있습니다. 여기서 만약, 주기적으로 실행되어야 하는 동작이 async 함수라면 어떻게 해야 할까요? 현재 Rust는 async closure를 지원하지 않습니다. `run_interval`함수의 두 번째 인자인 f도 Future 타입은 아니죠.

해결하는 방법은 [지난 글](https://huijeong-kim.github.io/2022/08/14/run-grpc-server-in-actix/)에서 사용한 방식과 동일합니다. fn에서 future를 spawn하면 됩니다. 지난 글처럼 해당 context에 spawn 하거나, actix system(현재 arbiter)에 spawn 할 수 있습니다.
```rust
let fut = async move {
    do_something().await;
};
let actor = fut.into_actor(fut);

ctx.spawn(actor);        // 방법 1, 현재 context에 spawn 하기
actix::spawn(fut);       // 방법 2, 현재 system에 spawn 하기
```

 
 

아래 코드는 지금까지 설명한 `run_interval`을 다양한 방식으로 사용하는 코드 예시입니다. 아래 코드에서 특이한(?) 포인트는 `send_ping` 함수에서 `self.clone()`를 하는 건데요. async block 내에서 self의 `async_send_ping`를 호출하고 있기 때문 입니다. 여기서 self가 async block 내로 move될 수 없기 때문에 clone 하였습니다. 
```rust
use actix::{Actor, Addr, AsyncContext, Context, Handler, Message};

/// Define `Ping` message
#[derive(Default, Message)]
#[rtype(result = "usize")]
struct Ping(usize);

/// Actor
#[derive(Clone)]
struct MyActor {
    count: usize,
}

/// Declare actor and its context
impl Actor for MyActor {
    type Context = Context<Self>;

    fn started(&mut self, ctx: &mut Self::Context) {
        println!("Actor started");

        ctx.run_interval(std::time::Duration::from_secs(3), |_act, _ctx| {
            MyActor::run_associated_func();
        });

        ctx.run_interval(std::time::Duration::from_secs(1), |act, _ctx| {
            act.run_func();
        });

        ctx.run_interval(std::time::Duration::from_secs(1), |act, ctx| {
            act.send_ping(ctx);
        });
    }
    fn stopped(&mut self, _ctx: &mut Self::Context) {
        println!("Actor stopped");
    }
}

impl MyActor {
    pub fn new(count: usize) -> Self {
        Self {
            count,
        }
    }

    fn run_associated_func() {
        println!("CASE 1. run associated func");
    }

    fn run_func(&self) {
        println!("CASE 2. run func, count {}", self.count);
    }

    fn send_ping(&mut self, ctx: &mut Context<Self>) {
        let actor = self.clone();
        let address = ctx.address();
        let fut = async move {
            actor.async_send_ping(address).await;
        };

        actix::spawn(fut);
    }

    async fn async_send_ping(&self, addr: Addr<MyActor>) {
        addr.send(Ping(10)).await.unwrap();
        println!("CASE 3. Send ping, now count is {}", self.count);
    }
}

/// Handler for `Ping` message
impl Handler<Ping> for MyActor {
    type Result = usize;

    fn handle(&mut self, msg: Ping, _: &mut Context<Self>) -> Self::Result {
        self.count += msg.0;
        self.count
    }
}

fn main() {
    let system = actix::System::new();

    system.block_on(async move {
        MyActor::new(0).start();
    });
    system.run().unwrap();
}
```

실행 결과
```rust
Actor started
CASE 2. run func, count 0
CASE 3. Send ping, now count is 0
CASE 2. run func, count 10
CASE 3. Send ping, now count is 10
CASE 1. run associated func
CASE 2. run func, count 20
CASE 3. Send ping, now count is 20
CASE 2. run func, count 30
CASE 3. Send ping, now count is 30
CASE 2. run func, count 40
CASE 3. Send ping, now count is 40
CASE 1. run associated func
CASE 2. run func, count 50
CASE 3. Send ping, now count is 50
CASE 2. run func, count 60
CASE 3. Send ping, now count is 60
CASE 2. run func, count 70
CASE 3. Send ping, now count is 70
CASE 1. run associated func
CASE 2. run func, count 80
CASE 3. Send ping, now count is 80
CASE 2. run func, count 90
CASE 3. Send ping, now count is 90

(생략)
```

 

결과를 보면, `send_async_ping` 에서 출력한 count가 현재 ping으로 인해 증가된 값이 반영되지 않는 문제가 있습니다. `self.clone()`의 영향 일까요? 이 부분은 분석이 완료되면 새로운 글로 정리해 보겠습니다 :) 


 
 

### References
[https://github.com/actix/actix/blob/master/actix/src/actor.rs](https://github.com/actix/actix/blob/master/actix/src/actor.rs)
[https://docs.rs/actix/latest/actix/prelude/trait.AsyncContext.html#method.run_interval](https://docs.rs/actix/latest/actix/prelude/trait.AsyncContext.html#method.run_interval)

 
 
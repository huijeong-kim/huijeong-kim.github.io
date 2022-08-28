---
layout: post
title: Rust 코드에서 macro 펼쳐보기
tags: rust cargo
---

Rust에서 사용 된 macro의 구현이 궁금할 때 [cargo expand](https://github.com/dtolnay/cargo-expand)를 사용할 수 있습니다.

 
 

### 설치
```rust
cargo install cargo-expand
```

 
 

### 사용법
[이전 글](https://huijeong-kim.github.io/2022/08/14/run-grpc-server-in-actix/)에서 사용한 코드를 expand해 봅시다. `cargo expand` 시 expand 된 코드가 stdout으로 출력됩니다.
```rust
cargo expand --bin actix-and-tonic
```

이 예제는 하나의 파일로 이루어져 있어 binary를 지정했지만, 특정 module 코드를 expand하기 위해서 module을 지정할 수도 있습니다.
```rust
cargo expand path::to:module
```

 

Message 구현에 사용했던 Message macro는
```rust
#[derive(Default, Message)]
#[rtype(result = "String")]
struct GetName;
```

이렇게 풀어쓸 수 있습니다.  
```rust
impl ::core::default::Default for GetName {
    #[inline]
    fn default() -> GetName {
        GetName {}
    }
}
impl ::actix::Message for GetName {
    type Result = String;
}
```
`derive(Default)`에 의해 Default 구현이 추가되었고, `derive(Message)`에 의해 Message 구현이 추가 되었습니다. `rtype(result="String")`은 그대로 `type Result = String`으로 변환됨을 확인할 수 있습니다.

 
 
 
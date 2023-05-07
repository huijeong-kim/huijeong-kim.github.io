---
layout: post
title: Exploring Rust Strings
tags: rust String str 
---

안녕하세요.

오늘은 Rust의 문자열 개념을 간단하게 알아보고, 코드 작성 시 문자열 관련 혼란스러웠던 포인트들을 정리해 보고자 합니다.

&nbsp;

> **Table of contents**
- [1. String vs. str](#1-string-vs-str)
- [2. str vs. \&str](#2-str-vs-str)
- [3. String vs. Box\<str\>](#3-string-vs-boxstr)
- [4. CString, CStr, OsString, OsStr](#4-cstring-cstr-osstring-osstr)
- [5. Char](#5-char)
- [6. Anti-patterns](#6-anti-patterns)
- [마무리하며](#마무리하며)

&nbsp;
&nbsp;

# 1. String vs. str

`String` 과 `str` 는 모두 **valid** UTF-8 문자열을 나타냅니다. Invalid UTF-8 데이터로 `String` 타입을 생성할 수 없습니다. UTF-8, Unicode 등에 대한 설명은 생략합니다.

Rust에서는 C와 같이 null-terminating string 개념을 사용하지 않습니다. 대신 `String` 타입은 문자열과 그 길이를 갖고 있는 **"fat pointer"** 입니다. "fat pointer"는 raw pointer과 additional metadata (e.g., length)를 갖고 있는 포인터를 의미합니다.

`String`과 `str`(string slice) 타입의 주요 차이점은 다음과 같습니다.

||**`String`**|**`str`**|
|저장 형태|`Vec<u8>`|`[u8]`|
|할당 위치|Heap|Stack|
|Ownership|O|X|
|Mutability|growable|immutable|

&nbsp;

`as_str()`을 사용하면 해당 `String`을 참조하는 `&str`을 얻을 수 있습니다. 이 때 `String`의 lifetime이 `&str`보다 길어야 합니다.

`into_string()`을 사용하면 `&str`을 `String` 형태로 변환할 수 있습니다. 이 때 heap 위에 새로운 메모리가 할당되고 데이터가 복사됩니다. `to_owned()`를 사용하여 ownership이 없는 `&str`을 ownership이 있는 `String`으로 바꿀 수도 있습니다. 이 때도 복사가 일어납니다.

&nbsp;

기본적으로는 mutable, growable string이 필요할 경우 `String`을, 그렇지 않을 땐 `str`을 사용하면 될 것 같아요.

&nbsp;

&nbsp;

# 2. str vs. &str

`str` 타입은 단독으로 쓰이는 경우는 거의 없습니다. 주로 `&str` 과 같은 reference 타입으로 사용됩니다. 

다음과 같이 `str` 타입을 사용하려 하면 compile error 가 발생합니다.

```rust
let str: str = "hello world";
```
컴파일 결과
```rust
   Compiling playground v0.0.1 (/playground)
error[E0308]: mismatched types
 --> src/main.rs:2:20
  |
2 |     let str: str = "hello world";
  |              ---   ^^^^^^^^^^^^^ expected `str`, found `&str`
  |              |
  |              expected due to this

error[E0277]: the size for values of type `str` cannot be known at compilation time
 --> src/main.rs:2:9
  |
2 |     let str: str = "hello world";
  |         ^^^ doesn't have a size known at compile-time
  |
  = help: the trait `Sized` is not implemented for `str`
  = note: all local variables must have a statically known size
  = help: unsized locals are gated as an unstable feature
help: consider borrowing here
  |
2 |     let str: &str = "hello world";
  |              +

Some errors have detailed explanations: E0277, E0308.
For more information about an error, try `rustc --explain E0277`.
error: could not compile `playground` due to 2 previous errors
```

문제는 두 가지 입니다. 첫 번째는 대입하고자 하는 데이터인 "hello world"가 `str`가 아닌 `&str` 타입이라는 것 이고, 두 번째는 `str` 타입의 크기는 compile type에 알 수 없다는 것 입니다.


"hello world"와 같은 **string literal**은 `&str` 타입입니다. 실행 파일의 text 영역에 하드코딩 된 문자열을 참조하는 형태기 때문입니다.

`str` 은 **string slice**로 DST(Dynamic Sized Type) 중 하나 입니다. [DST](https://doc.rust-lang.org/reference/dynamically-sized-types.html)는 compile time에 그 크기를 알 수 없는 타입으로, `Slice`, `Trait` 이 대표적인 예 입니다. 해당 타입들은 그냥 사용할 순 없고, `Box`를 사용하여 heap 에 위치시키거나, reference 로 Sized object를 가리키도록 하는 등의 방식으로 사용해야 합니다. `Vec<dyn Trait>`을 사용할 때 발생하는 컴파일 에러는 [이전 포스트](https://huijeong-kim.github.io/2023/01/29/rust-polymorphism/)에서 살펴본 적 있습니다.


`str`은 reference 형태로 많이 사용되고, reference 대상은 <U>Heap-allocated String</U>, <U>String literal</U> 등이 됩니다.

```rust
let heap_allocated_strings = String::from("Hello from heap");
let str = "Hello from binary";

let ref_to_heap: &str = &heap_allocated_strings;
let ref_to_literal: &str = &str;

println!("{}", ref_to_heap);
println!("{}", ref_to_literal);
```
실행결과
```rust
Hello from heap
Hello from binary
```
&nbsp;

String literal은 프로그램 실행 시간 전체에서 유효한 값으로, 그 lifetime은 static 입니다. 위의 예시 코드에서 string literal에 다음과 같이 lifetime을 명시할 수도 있습니다.
```rust
let str = "Hello world from bin";
let ref_to_literal: &'static str = &str;
```

&nbsp;

&nbsp;

# 3. String vs. Box\<str\>

DST인 `str`을 reference 형태가 아닌 `Box` 형태로 heap에 할당, 이를 가리키도록 할 수 있습니다. 이 경우, 애초에 heap에 메모리 할당받는 `String` 타입과 유사해 집니다.

이를 위해 단순히 `Box<str>` 타입을 `&str` 타입으로부터 생성하려 하면 컴파일 에러가 발생합니다.

```rust
let boxed_str: Box<str> = Box::new("hello world");
println!("boxed_str: {}", boxed_str);
```
컴파일 결과
```rust
error[E0308]: mismatched types
  --> src/main.rs:15:31
   |
15 |     let boxed_str: Box<str> = Box::new("hello world");
   |                    --------   ^^^^^^^^^^^^^^^^^^^^^^^ expected `Box<str>`, found `Box<&str>`
   |                    |
   |                    expected due to this
   |
   = note: expected struct `Box<str>`
              found struct `Box<&str>`

For more information about this error, try `rustc --explain E0308`.
error: could not compile `playground` due to previous error
```

&nbsp;

대신 `&str`을 `String` 타입으로 변환(heap 영역 할당 후 복사)한 뒤 이를 `into_boxed_str()`로 변환할 수 있습니다.

```rust
let string: Box<str> = String::from("Hello").into_boxed_str();
let string: Box<str> = "Hello".to_string().into_boxed_str();
let string: Box<str> = Box::from("Hello");
```

세 번째 방식인 `Box::from(&str)` 타입 또한 내부적으로 heap에 메모리를 할당받고 값을 복사합니다. (참고: [docs](https://doc.rust-lang.org/std/boxed/struct.Box.html#impl-From%3C%26str%3E-for-Box%3Cstr,+Global%3E))

> | This conversion allocates on the heap and performs a copy of s. 

&nbsp;

`String`은 mutable, growable 합니다. 값 변경/추가가 필요한 경우라면 `Box<str>`가 아닌 `String`을 사용해야 합니다.

`String`과 `Box<str>`의 또 다른 차이점은 자료구조 형태입니다. `String`은 `Vec<u8>` 형태로 문자를 저장하고, `Vec` 는 dynamic array로 실제 데이터의 크기와 자료구조 크기(capacity)는 다를 수 있습니다. 따라서 `String`은 실제 데이터보다 많은 메모리를 사용할 수 있고, "데이터 포인터, 크기(len), 용량(capacity)"를 저장하므로 동일 문자열을 저장하더라도 포인터의 크기가 더 큽니다.

다음과 같이 두 타입의 크기를 출력해 보면, `Box<str>`는 문자열 포인터와 문자열 길이를 포함하여 16Bytes, `String`은 capacity 까지 포함하여 24Bytes 임을 확인할 수 있습니다.

```rust
println!("{}", core::mem::size_of::<usize>()); // 8
println!("{}", core::mem::size_of::<Box<str>>()); // 16
println!("{}", core::mem::size_of::<String>()); // 24
```

[문서](https://doc.rust-lang.org/std/string/struct.String.html#method.into_boxed_str)에 나와있는 것과 같이, `String`을 `Box<str>`로 변환 시 빈 공간 (남아 있는 capacity)는 drop 될 수 있습니다.

> | This will drop any excess capacity.

&nbsp;

String mutability 가 필요하지 않은 상황, memory footage가 중요한 상황에선 `Box<str>` 사용이 중요할 것 같아요.

&nbsp;
&nbsp;

# 4. CString, CStr, OsString, OsStr

앞서 Rust의 문자열은 "null-terminated string"이 아니라고 하였는데, 이는 C와 다른 형태입니다. C와 같은 형태의 문자열은 `std::ffi::CString`, `std::ffi::CStr` 을 사용하여 표현할 수 있습니다. Rust 프로그램에서 이런 문자열을 사용할 필요는 없겠지만, C/C++ 코드와 FFI(Foreign Function Interface)로 연결할 때 필요합니다. `CString`의 `into_string()`, `CStr`의 `to_str()`를 통해 Rust 문자열로 변환할 수 있습니다. 이 때 invalid UTF-8 데이터가 포함되었다면 `IntoStringError` 발생합니다.

`OsString`과 `OsStr`은 platform-native string 으로, 각 플랫폼(unix, windows 등)에 맞는 문자열 형태입니다. `into_string()` 함수를 통해 `String`으로 변환 가능하며, 이 때도 invalid UTF-8 데이터가 포함되어 있다면 변환 실패합니다. 이런 platform-native 문자열 타입이 정의되어 있고 `String` 변환 함수가 있어 코드 작성하기도 편리하고 안전한 코드를 짤 수 밖에 없게 만드는 것 같아요.

`CString`과 `OsString`은 `String`의 대응, `CStr`과 `OsStr` 은 `str`의 대응입니다.

`String`으로 변환 시 `from_utf8_lossy()` 함수를 사용하면, invalid UTF-8 글자는 [U+FFFD REPLACEMENT CHARACTER (�)](https://doc.rust-lang.org/std/char/constant.REPLACEMENT_CHARACTER.html)로 바꿔서 변환할 수도 있습니다.

&nbsp;

&nbsp;

# 5. Char

Rust의 `Char` 타입은 "unicode scalar value" 입니다. [이전 포스트](https://huijeong-kim.github.io/2023/05/02/rust-as-operator/)에서 `as`를 통한 `char` type casting은 u8에 대해서만 가능한 걸 확인했는데, 이 때문이더라고요. 알고보니 너무 당연한 것..

String literal 의 경우 `"hello world"` 와 같이 쌍따옴표로 표시하였는데요. `'A'`와 같이 따옴표로 character literal를 정의할 수 있습니다. 이 값은 single Unicode character 입니다. 

`String` 타입과 `str` 타입은 `chars()` 함수를 통해 안전하게 문자열 내 Char들을 iterate 할 수 있습니다. 

&nbsp;

&nbsp;

# 6. Anti-patterns

Rust의 문자열 타입에 대한 크게 생각하지 않고 무작정 사용하다 보면 컴파일 지옥에 빠져 헤매다 대충 이리저리 변환해서 사용하게 되는데요. 이런 것들이 모두 anti-pattern일지는 모르겠지만, 고민되는 내용들을 적어 봤습니다.

&nbsp;

**move, lifetime 이슈를 피하기 위한 clone**

다음과 같이 self 내에 정의된 `String` 타입을 계속해서 재활용할 때, move가 일어나지 않도록, 혹은 lifetime 이슈가 발생하지 않도록 clone을 할 수도 있습니다. 

```rust
fn do_something(&self, path: &String) {
    let new_dir = self.name.clone();
    let path = Path::new(path.clone());
    path.push(new_dir);
    std::fs::create_dir(&path);
}
```

하지만 대부분의 경우 reference 값으로 원하는 결과를 만들 수 있습니다. 필요 이상으로 clone하지 않도록 노력하는 게 좋은 것 같습니다.

```rust
fn do_something(&self, path: &String) {
    let path = Path::new(path);
    let path = path.join(&self.filename);
    std::fs::create_dir(&path);
}
```

&nbsp;

`&String` **타입을 clone**

이 경우는 조금 헷갈리긴 하지만, reference를 clone하는 것은 직관적인 것 같진 않습니다.

```rust
fn create_something(name: &String) -> SomeStruct {
    SomeStruct {
        name: name.clone(),
    }
}
fn create_other(name: &String) -> OtherStruct {
    OtherStruct {
        name: name.clone(),
    }
}
fn main() {
    let my_name = "hailey".to_string();
    let some = create_something(&my_name);
    let other = create_other(&my_name);
}
```

이 보다는 소유권을 가질 struct에게 ownership을 가진 변수를 넘겨주면 좋을 것 같습니다.

```rust
fn create_something(name: String) -> SomeStruct {
    SomeStruct {
        name,
    }
}
fn create_other(name: String) -> OtherStruct {
    OtherStruct {
        name,
    }
}
fn main() {
    let my_name = "hailey".to_string();
    let some = create_something(my_name.clone());
    let other = create_other(my_name.clone());
}
```

혹은 애초에 변하지 않는 값인 name은 `&str` 타입을 사용해도 좋을텐데, 이 경우 주어지는 `&str` 타입의 lifetime과 두 struct 의 lifetime을 잘 고려해야 합니다. 사용자 요청으로 받은 `String`에 대한 reference를 전달하고 이를 어떤 struct 멤버로 둔다면, 사용자 요청이 종료될 때 struct도 같이 소멸되는지 잘 확인해 봐야 할 것입니다. 그 전에 컴파일 지옥에 빠지겠지만요..

```rust
fn create_something(name: &str) -> SomeStruct {
    SomeStruct {
        name, // type is &str
    }
}
fn create_other(name: &str) -> OtherStruct {
    OtherStruct {
        name, // type is &str
    }
}
fn name_provided(my_name: &str) {
    let some = create_something(my_name);
    let other = create_other(my_name);
}
```

애초에 하나의 값을 여러 struct 가 공유할 일을 만들지 않는 것이 더 좋을 것 같기는 합니다.

&nbsp;

`String`**보다는** `&str` **사용하기**

그러고 싶은데 이게 잘 안 됩니다. 예를 들어 error message의 타입을 `String` 보다는 `&str`으로 정의하고 전달하고 싶은데요. 단순히 `Invalid request` 와 같은 메시지를 전달하는 거라면 `&'static str` 으로 정의할 수 있을텐데요. Error message에 여러 값을 포함시키고 싶다면 `String`으로 만들어야 합니다. 이를 `as_str()`으로 받은 `&str` 값을 전달할 순 있겠지만, 현재 함수에 정의한 임시 `String`변수를 참조하는 `&str`을 전달하므로 컴파일 되지 않습니다.

이럴 경우 위에서 언급한 `Box<str>`을 사용할 수 있을 것 같은데, 어느게 더 좋은 패턴일까요..?

&nbsp;

분명 더 많은 anti-pattern이 있었던 것 같은데 오늘은 여기까지만 생각나는군요. 다음에 또 업데이트 하겠습니다 :) 

&nbsp;

&nbsp;

# 마무리하며

Rust 컴파일러를 통과하기 위한 코드를 짜다 보면 `as_str()`, `to_string()`, `clone()` 등을 남발하며 마음 한 켠이 불편했는데요. 이렇게 생각을 안하면 전체적인 코드의 구조, 의도도 불명확해지고, 언젠가는 지나친 memory copy overhead 로 인한 성능 문제로도 고생할 것 같습니다. 변수의 type을 정하는 것은 생각보다 어려운 일인 것 같아요.

&nbsp;

&nbsp;

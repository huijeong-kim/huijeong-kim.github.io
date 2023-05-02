---
layout: post
title: as operator in Rust
tags: rust as type casting  
---

안녕하세요. 

오늘은 Rust의  `as` 를 사용한 type casting에 대해 알아보고자 합니다. C++ 프로그램에서는 잘못된 type 사용 으로 인한 오류를 종종 볼 수 있습니다. `uint64_t` 값을 `uint32_t` 값에 대입하여 잘못된 값으로 동작하는 경우가 그 예입니다. 조금은 어이 없는 실수이긴 한데, 생각보다 자주 발견됩니다. 이런 류의 버그는 처음 봤을 때 원인을 가늠하기 힘들기도 하지만, 고치기 귀찮거나 어렵기도 합니다. Type 재정의를 사용하지 않는 경우도 많고, 가끔씩은 통일할 필요가 크게 없는 경우도 있고, 의미 상 같은 변수를 모두 찾아내기 힘든 코드들도 종종 있습니다. 테스트를 더 잘 하면 된다고 하지만, 모든 변수에 대해 boundary test를 하는 건 현실적으로 조금 어렵습니다. 

Rust에서는 엄격하게 type 검사를 합니다. Rust로 작성할 땐 `u64`를 `u32`에 대입할 수 없습니다. `into()` 혹은 `try_into()` 등을 통해 type conversion하여야 하고, 적절하지 못한 type 변환 시도는 컴파일 타임에 발견되거나 런타임에 에러 리턴이 됩니다. 그래서 Rust 사용 시에는 type 관련해서는 매우 안심하고 있었는데, 생각치 못한 예외가 있더라고요. 바로 `as`를 사용한 type casting 입니다.

`as`는 Rust의 **type cast operator** 입니다. [Reference book](https://doc.rust-lang.org/reference/expressions/operator-expr.html#type-cast-expressions)와 [std reference](https://doc.rust-lang.org/std/keyword.as.html)를 참고하여 그 용도를 정리하고, type casting과 type conversion 으로 막을 수 있는/없는 invalid casting을 살펴보겠습니다.

<br><br>

## Type casting
`as`는 primitive 의 type casting에 사용됩니다. Type을 강제 할당하는 것이기 때문에, 런타임 casting 실패는 존재하지 않습니다. 하지만 casting이 가능한 케이스는 한정되어 있습니다. 그 외의 경우는 compile time에 `invalid casting`으로 실패합니다.

<br>

### Numeric cast
- 동일 size integer: no-op
- smaller integer -> larger integer: zero/sign-extend
- larger integer -> smaller integer: truncate (!!)

<br>

제가 최근에 습관적으로 `as`를 사용하였는데, 이 경우 특별한 경고(compile error/warning 등) 없이 값을 잃을 수 있더라고요. `as`는 type 변환이 아닌 **casting**을 해 주는 것 이기 때문에 사용에 주의해야 할 것 같습니다.

```rust
let number: u64 = u64::MAX;
let cast_as = number as u32;
println!("Numbers: {:x}, {:x}", number, cast_as);
```
실행 결과
``` cpp
Numbers: ffffffffffffffff, ffffffff
```

<br>

Numeric casting 하고 싶을 때는 `as`를 사용한 casting 대신 `into()` 혹은 `try_into()`를 사용하여 type conversion을 하는 것이 좋겠습니다.
위와 같은 예제에서 `into()`를 사용하면, `u32`에 대한 `From<u64>`는 구현되어 있지 않다는 이유로 compile error가 발생합니다. 물론 직접 `into()` 함수를 구현할 수도 있겠지만 굳이 그럴 필욘 없겠지요. `try_into()`를 사용한다면 결과값은 `TryFromIntError` 에러가 됩니다.

```rust
// error[E0277]: the trait bound `u32: From<u64>` is not satisfied
//let cast_into: u32 = number.into();
    
let cast_into: Result<u32, _> = number.try_into();
println!("Conversion result: {:?}", cast_into);
```
실행 결과
```cpp
Conversion result: Err(TryFromIntError(()))
```

<br>

### Enum cast
Enum의 numeric casting 도 가능합니다. 단,
- Unit-only enums
- Field-less enums (without explicit discriminants)
만 가능합니다.

<br>

[Field-less enum](https://doc.rust-lang.org/reference/items/enumerations.html#field-less-enum)은 *"no constructors contain fields"* 인 enum 입니다. Tuple, Struct 와 같은 enum 값이 있더라도 그 안에 field가 없는 경우를 말하는 것 같습니다. 이 경우에는 explicit discriminants(`=3` 과 같은 값 지정)가 없어야만 numeric casting 가능합니다.

``` rust
#[derive(Debug)]
enum FieldOnlyEnum {
    Tuple(),
    Struct{},
    Unit,
}

println!("FieldOnlyEnum: Tuple {:?}, Struct {:?}, Unit {}",
    FieldOnlyEnum::Tuple() as u8, 
    FieldOnlyEnum::Struct{} as u32, 
    FieldOnlyEnum::Unit as u8);
```
실행 결과
```cpp
FieldOnlyEnum: Tuple 0, Struct 1, Unit 2
```

<br>

여기에 `FieldOnlyEnum::Unit`에 `Unit = 1,`과 같이 명시적으로 값을 지정하면 *"discriminant value `1` assigned more than once"* 에러가 발생합니다. 하지만 enum에 `#[repr(u8)]` 지정할 경우 explicit discriminant가 있는 field-less enum에서도 numeric casting 가능하다고 합니다. 이 때 tuple type이나 struct type에 discriminant value 지정하는 경우에는 *"non-primitive cast"*로 compile error 발생합니다.

```rust
#[derive(Debug)]
#[repr(u8)]
enum FieldOnlyEnum2 {
    OtherUnit = 3,
    Tuple(),
    Struct{},
    Unit,
}

println!("FieldOnlyEnum2: Tuple {:?}, Struct {:?}, Unit {}",
    FieldOnlyEnum2::Tuple() as u8, 
    FieldOnlyEnum2::Struct{} as u32, 
    FieldOnlyEnum2::Unit as u8);
```
실행 결과
``` cpp
FieldOnlyEnum2: Tuple 4, Struct 5, Unit 6
```

<br>

여기서 또 한 가지 신기한 점은, casting 시 `FieldOnlyEnum::Struct as u8` 와 같이 작성하는 경우(생성자 사용 X)엔 compile error 가 발생하고, `FieldOnlyEnum::Tuple as u8` 로 작성하면 빌드는 성공하나 값이 이상하게 `FieldOnlyEnum: Tuple 224, Struct 1, Unit 2` 로 나오더라고요. 혼란스럽지만 이런 enum type casting이 얼마나 필요할 지 모르겠으니 그냥 넘어가 보겠습니다..

<br>

Unit-only enum은 field-less enum 중 unit으로만 이루어진 경우입니다. 이 때는 Field-only enum과 다르게 *"explicit discriminant value"*를 갖고 있어도 상관 없습니다. 아래 코드에서 주석 처리 된 것 처럼 `UnitNum3`이 `i32`란 field를 갖고 있는 경우(unit-only enum이 아닌 경우), 모든 type casting에서 에러가 발생합니다. 

``` rust

#[derive(Debug)]
enum UnitOnlyEnum1 {
    UnitNum,
    UnitNum2,
    UnitNum3, //UnitNum3(i32), => error[E0605]: non-primitive cast: `UnitOnlyEnum1` as `u32`
}
println!("UnitOnlyEnum: UnitNum {}, UnitNum2 {} UnitNum3 {}",
    UnitOnlyEnum1::UnitNum as u8,
    UnitOnlyEnum1::UnitNum2 as u16,
    UnitOnlyEnum1::UnitNum3 as u32,
);

enum UnitOnlyEnum2 {
    UnitNum = 1,
    UnitNum2 = 4,
}
println!("UnitOnlyEnum2: UnitNum {}, UnitNum2 {}",
    UnitOnlyEnum2::UnitNum as u32,
    UnitOnlyEnum2::UnitNum2 as u32,
);
```
실행 결과
```cpp
UnitOnlyEnum: UnitNum 0, UnitNum2 1, UnitNum3 2
UnitOnlyEnum2: UnitNum 1, UnitNum2 4
```

<br>

### Other primitives
- false -> 0, true -> 1
- char -> numerics: 해당 char code
- numerics -> char: 해당하는 char code의 char

<br>

numerics -> char 변환은 `u8`만 가능합니다. 그보다 큰 값을 가진 emoji 들은 `try_into`나 `into`를 사용해 type conversion 하면 되겠습니다.

```rust
println!("False as u8: {}", false as u8);
println!("True as u8: {}", true as u8);

println!("'C' as u8: {}", 'C' as u8);
println!("67 as char: {}", 67 as char);

println!("'❤️' as u32: {}", '❤' as u32);

// error: only `u8` can be cast into `char`
//println!("10084 as char: {}", 10084 as char);

//error[E0604]: only `u8` can be cast as `char`, not `u32`
//println!("10084 as char: {}", 10084u32 as char);

let result: Result<char, _> = (10084 as u32).try_into();
println!("Converting result: {:?}", result);

// error[E0277]: the trait bound `char: From<u32>` is not satisfied
// let result: char = (10084 as u32).into();
// println!("Converting result: {:?}", result);
```
실행 결과
```cpp
False as u8: 0
True as u8: 1
'C' as u8: 67
67 as char: C
'❤️' as u32: 10084
Converting result: Ok('❤')
```

<br>

pointer to address cast의 경우 예상하는 바와 유사한 것 같습니다. casting 되는 type에 따라 값이 truncate 되어 잘못된 주소를 반환할 수 있으니 주의해야 합니다. `usize` 로 casting 하거나, 다음과 같이 `_`를 사용하여 컴파일러가 추론하게 만들 수도 있습니다.

```rust
let num = 42;
let address = &num as *const _;
println!("The address of num is {:?}", address);
```
실행 결과
``` cpp
The address of num is 0x7fffd6002dfc
```

<br><br>

## Type coercions

Type 을 강제할 때 사용합니다. 흔히 사용하는 literal 에 type 지정하는 경우가 그 예입니다. 이 때 잘못된 casting을 하면 (e.g., 너무 큰 값을 u8로 casting) compile error가 발생합니다.

```rust
let number_coercions = 123 as u8;

// error: literal out of range for `u8`
//let number_coercions2 = 10002 as u8;

println!("numbers: {}", number_coercions);
```
실행 결과
```cpp
numbers: 123
```

<br>

또 다른 예는 Trait 으로의 변환입니다. 다음과 같이 `&dyn Animal` trait 으로 casting 하여 해당 trait의 함수를 사용할 수 있습니다.

```rust
use std::any::Any;

trait Animal {
    fn speak(&self);
}

#[derive(Debug)]
struct Dog {
    name: String,
}

impl Animal for Dog {
    fn speak(&self) {
        println!("{} says woof!", self.name);
    }
}

fn trait_coercions() {
    let dog = Dog { name: String::from("Happy") };
    
    let animal = &dog as &dyn Animal;
    animal.speak();
}
```
실행 결과
``` cpp
Happy says woof!
```

<br>

근데 casting 없이 그냥 호출해도 되고, 다음과 같이 함수 인자로 전달하면 자연스럽게(암묵적으로) casting 되는데, 위와 같이 `as Trait` 을 사용하는 경우가 어떤 경우인지 감이 오질 않네요.

```rust
fn trait_coercions() {
    let dog = Dog { name: String::from("Happy") };
    let_animal_speak(dog);
    
}
fn let_animal_speak(animal: &dyn Animal) {
    animal.speak();
}
```
실행 결과
```cpp
Happy says woof!
```

<br>

이 보다는 Trait을 downcasting를 하고 싶은 경우가 더 많을 것 같습니다. 꼭 downcasting을 해야 하겠느냐고 하면 할 말이 없지만(다른 방식으로 해결하는 게 좋은 경우가 더 많지만) 그래도 현실적으로 필요할 때가 종종 있습니다. 이 땐 `std::any::Any::downcast_ref`를 사용할 수 있습니다([참고](https://bennett.dev/rust/downcast-trait-object/)). 이 과정에서 `animal as &dyn Any`와 같이 casting 하면 downcast가 정상 동작하지 않는 등 예상대로 동작하지 않는데요.  `as_any()` 의 신비는 다음에 기회 되면 알아보고 정리해 보겠습니다.

다음 예제에서 Dog, Cat은 `Animal` trait을 구현하고 있고, Desk는 구현하고 있지 않습니다.

```rust
trait Animal {
    fn speak(&self);
    fn as_any(&self) -> &dyn Any;
}
// .. 생략 ..

impl Animal for Dog {
    // .. 생략 ..

    fn as_any(&self) -> &dyn Any {
        self
    }
}

// .. 생략 ..

let animal_vec: Vec<Box<dyn Animal>> = vec![
    Box::new(Dog { name: String::from("Fido") }),
    Box::new(Cat { name: String::from("Whiskers") }),
];

for animal in animal_vec {
    animal.speak();

    if let Some(dog) = animal.as_any().downcast_ref::<Dog>() {
        println!("animal's name is {}", dog.name);    
    }
}

let desk = Desk { name: String::from("birch") };
let desk_as_any = &desk as &dyn Any;
let is_dog = desk_as_any.downcast_ref::<Dog>();
println!("desk is dog?: {:?}", is_dog);
```
실행 결과
```cpp
Fido says woof!
animal's name is Fido
Whiskers says meow!
desk is dog?: None
```

<br><br>

## Rename imports

마지막으로 C++ 의 namespace renaming과 유사한 rename imports 가 있는데, 이는 type casting이 아니므로 생략합니다.

<br><br>

## 결론

<br>

Rust 의 type casting은 C++ 대비 매우 제한적이고, 정의되지 않은 casting은 invalid casting으로 처리되는 것 같습니다. 하지만 이 기준을 Reference Book만 보고 파악하기는 조금 어렵더라고요. 

이번 편을 작성하다 보니, Type Casting이 꼭 필요한 경우는 사실 없지 않을까 싶었습니다.(Type Casting이 유용한 예가 있다면 공유해 주시길!!) 오늘도 조금 뻔한 결론, type casting 보다는 type conversion을 사용하여 암묵적인 변환으로 인해 발생할 수 있는 버그를 피하자로 마무리 하겠습니다.

<br><br>
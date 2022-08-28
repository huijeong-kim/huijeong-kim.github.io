---
layout: post
title: Rust로 Parameterised Test 작성하기
tags: rust unittest
---

Unit test를 작성 하다 보면 간단한 함수에 대해서도 꽤나 많은 수의 테스트 케이스를 작성하게 됩니다. Input 값의 조합, 특히나 Invalid Input 값의 조합은 무지 많아질 수 있기 때문 입니다. 그 모든 케이스를 하나 씩 테스트로 작성 하는 것은 꽤 귀찮기도 하고, 테스트 함수 작명 지옥에 빠지면서 테스트 가독성도 떨어지게 됩니다. 


이럴 때 사용할 수 있는 것이 [table-driven test](https://github.com/golang/go/wiki/TableDrivenTests), 혹은 parameterised test 입니다. Input-Output 조합을 table로 표현하는 방식입니다. 

Rust로 간단한 함수의 parameterised test를 작성해 보고, rust의 `test-case`, `rtest` crate를 활용해 이를 더 간편하게 작성하는 방법을 알아보겠습니다.


 
 

### Parameterised Test

다음과 같은 아주 아주 간단한 함수의 테스트를 작성해 본다고 합시다.(`test-case` crate repo의 기본 예시 입니다 ^^) 두 개의 `i8` 값을 받아 두 수의 곱의 절대값을 반환하는 함수입니다. 

```rust
pub fn multiplication(x: i8, y: i8) -> i8 {
    (x * y).abs()
}
```

Input의 두 `i8` 각각이 음수 & 양수일 때 절대값을 돌려주는 지 확인하는 테스트를 작성해 보겠습니다. 너무 간단해서 와닿지 않을 수 있겠지만 예시로만 참고해 주세요.
``` rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn multiplication_tests_negative_negative() {
        let x = -2;
        let y = -4;

        let actual = multiplication(x, y);
        assert_eq!(8, actual);
    }

    #[test]
    fn multiplication_tests_negative_positive() {
        let x = -2;
        let y = 4;

        let actual = multiplication(x, y);
        assert_eq!(-8, actual);
    }

    #[test]
    fn multiplication_tests_positive_negative() {
        let x = 2;
        let y = -4;

        let actual = multiplication(x, y);
        assert_eq!(-8, actual);
    }

    #[test]
    fn multiplication_tests_positive_positive() {
        let x = 2;
        let y = 4;

        let actual = multiplication(x, y);
        assert_eq!(8, actual);
    }
}
```

**실행 결과**
```rust
running 4 tests
test tests::multiplication_tests_negative_negative ... ok
test tests::multiplication_tests_negative_positive ... ok
test tests::multiplication_tests_positive_negative ... ok
test tests::multiplication_tests_positive_positive ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

양수, 음수의 조합 4가지를 테스트로 작성하였습니다. 간단한 함수의 테스트인데 꽤 많은 코드가 필요합니다. 게다가 테스트 fail 발생할 경우 각 테스트를 하나 하나 읽으며 의도를 파악해야 합니다. 이 예제는 간단하여 금방 의도를 파악할 수 있겠으나, 코드가 조금만 더 복잡해진다면 배로 힘들어 질 것 같습니다. 게다가 만약 이 함수가 절대값이 아닌 곱셈 결과를 반환하도록 바뀌어야 한다면, 테스트 케이스 중 바뀌어야 하는 expected 값을 찾는 게 조금 힘들 수 있겠습니다.

 

이를 개선하기 위해 여기에 parameterised test를 적용하면 테스트 코드는 다음과 같이 바뀌게 됩니다.
```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn multiplication_tests() {
        let test_cases = vec![
            (-2, -4, 8, "when negative-negative value provided"),
            (-2, 4, 8, "when negative-positive value provided"),
            (2, -4, 8, "when positive-negative value provided"),
            (2, 4, 8, "when positive-positive value provided"),
        ];

        for (arg1, arg2, expected, err_msg) in test_cases {
            let actual = multiplication(arg1, arg2);
            assert_eq!(expected, actual, "Result is wrong {}", err_msg);
        }
    }
}
```

test case table에 input값, 예상 값과 함께 실패했을 경우 출력할 error message를 정의하였습니다. Fail 발생 시 적절한 error message를 출력하기 위해서 입니다. 만약 `multiplication` 함수에서 `abs()`를 제거한다면 다음과 같이 적절한 에러 메시지가 출력될 것 입니다.

```rust
running 1 test
test tests::multiplication_tests ... FAILED

failures:

---- tests::multiplication_tests stdout ----
thread 'tests::multiplication_tests' panicked at 'assertion failed: `(left == right)`
left: `8`,
right: `-8`: Result is wrong when negative-positive value provided', src/bin/test-cases.rs:21:13
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace


failures:
tests::multiplication_tests

test result: FAILED. 0 passed; 1 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

error: test failed, to rerun pass '--bin test-cases' 
```

test case table을 만드는 것 만으로도 테스트 코드가 간결해 졌습니다. 다른 모든 unit test code에도 동일하게 적용할 수 있겠습니다. 다만 매 번 table 을 만들고 for loop 를 만드는 수고로움이 있습니다. 이 부분은 test case table 생성을 도와주는 rust crate들의 도움을 받을 수 있습니다! 


 
 

### test-case 
[test-case](https://github.com/frondeus/test-case) crate를 사용하면 macro로 test case table을 만들 수 있습니다. 위에서 작성한 test case table을 `test_case` macro를 사용하여 수정하면 다음과 같습니다. 

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use test_case::test_case;

    #[test_case(-2, -4 => 8; "when negative-negative value provided")]
    #[test_case(-2, 4 => 8; "when negative-positive value provided")]
    #[test_case(2, -4 => 8; "when positive-negative value provided")]
    #[test_case(2, 4 => 8; "when positive-positive value provided")]
    fn multiplication_test(x: i8, y: i8) {
        let actual = multiplication(x, y);
        assert_eq!(8, actual)
    }
}
```

실행 결과는 다음과 같습니다. 실행 결과에서 유추할 수 있듯, 각 macro의 마지막에 적은 string 이 곧 test case 이름이 됩니다. 
```rust
running 4 tests
test tests::multiplication_test::when_negative_negative_value_provided ... ok
test tests::multiplication_test::when_positive_positive_value_provided ... ok
test tests::multiplication_test::when_negative_positive_value_provided ... ok
test tests::multiplication_test::when_positive_negative_value_provided ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

test case 이름을 지정하지 않으면 test case name은 input, output 값을 가지고 test case를 생성해 냅니다. 여기서 i8 input의 positive/negative는 구별하지 못하는 것 같더라고요. (-2, 4 => 8)과 (2, 4 => 8)이 같은 이름으로 생성되 에러가 발생하니 참고하세요.
```rust
#[test_case(-2, -4 => 8)]
#[test_case(-3, 4 => 12)]
#[test_case(1, -4 => 4)]
#[test_case(6, 4 => 24)]
fn multiplication_test(x: i8, y: i8) -> i8{
    multiplication(x, y)
} 
```
```rust
running 4 tests
test tests::multiplication_test::_1_4_expects_4 ... ok
test tests::multiplication_test::_2_4_expects_8 ... ok
test tests::multiplication_test::_3_4_expects_12 ... ok
test tests::multiplication_test::_6_4_expects_24 ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s 
```


그 외에도 expect panic, assert function 지정 등 재미있는 syntax들이 많습니다. 더 자세한 사용법은 [link](https://github.com/frondeus/test-case/wiki/Syntax)를 참고해 주세요.

 
 


### rtest

[rstest](https://github.com/la10736/rstest)는 test fixture, parameterised test 등을 제공하는 crate 입니다. 앞에서 소개한 test-case 보다 다양한 기능이 있는 것 같습니다. 오늘은 그 중 parameterised test 작성 부분만 살펴보겠습니다.

위에서 작성한 코드를 그대로 rstest를 사용하여 작성하면 다음과 같습니다. 자동 생성 된 Test case 명이 아쉬운데, 테스트 명을 변경하는 방법은 찾지 못하였어요. 

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use rstest::rstest;

    #[rstest]
    #[case(-2, -4, 8)]
    #[case(2, -4, 8)]
    #[case(-2, 4, 8)]
    #[case(2, 4, 8)]
    fn multiplication_test(#[case] x: i8, #[case] y: i8, #[case] expected: i8) {
        assert_eq!(multiplication(x, y), expected);
    }
} 
```
```rust
running 4 tests
test tests::multiplication_test::case_1 ... ok
test tests::multiplication_test::case_2 ... ok
test tests::multiplication_test::case_3 ... ok
test tests::multiplication_test::case_4 ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```


사용법도 조금 더 귀찮고 아쉽운 부분이 있긴 한데, `rstest`는 Test fixture 제공, Test timeout 설정하기 등등 다른 유용한 기능들이 많습니다. 이 기능들은 좀 더 사용해 보고 공유하겠습니다 :) 


 
 



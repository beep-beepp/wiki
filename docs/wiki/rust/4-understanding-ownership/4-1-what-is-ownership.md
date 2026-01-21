---
title: "4-1. What is Ownership?"
parent: "4. Understanding Ownership"
grand_parent: Rust
permalink: /rust/understanding-ownership/what-is-ownership/
nav_order: 1
---

# 1. What is Ownership?
## 1. 개요
소유권은 Rust의 가장 독특한 기능이며, 언어 전반에 깊은 영향을 미칩니다.
이 개념 덕분에 Rust는 가비지 컬렉터 없이도 메모리 안전성 보장을 할 수 있습니다.
따라서 소유권이 어떻게 동작하는지를 이해하는 것은 매우 중요합니다.

이 장에서는 소유권을 중심으로, 그와 밀접하게 연관된 여러 기능들인
빌림(borrowing), 슬라이스(slices), 그리고 Rust가 메모리를 배치하는 방식에 대해 다룰 것입니다.

--

모든 프로그램은 실행 중 메모리를 관리해야 하며, 언어마다 접근 방식이 다름.

가비지 컬렉션(GC)을 사용하는 언어:
→ 사용되지 않는 메모리를 런타임에서 자동 정리함

수동 메모리 관리 언어:
→ 개발자가 직접 메모리 할당 및 해제를 수행함

Rust의 방식:
→ 컴파일러가 검사하는 소유권 규칙을 통해 메모리를 관리함

소유권 규칙을 위반할 경우 컴파일 자체가 불가능함.
해당 규칙들은 런타임 성능에 영향을 주지 않음.


## 2. Ownership Rules (소유권 규칙)
Rust의 소유권 시스템은 다음 **세 가지 핵심 규칙**으로 구성됨.

| 규칙 번호 | 내용                     |
| ------- | ----------------------- |
| Rule 1  | Rust의 모든 값은 반드시 하나의 소유자를 가짐 |
| Rule 2  | 어떤 시점에도 하나의 값은 하나의 소유자만 가질 수 있음 |
| Rule 3  | 소유자가 스코프를 벗어나면 값은 drop됨 |

이 규칙들은 **컴파일러에 의해 정적으로 검사됨.**


## 3. Variable Scope (변수 스코프)
스코프(scope)는 프로그램 내에서 어떤 값이 유효한 범위를 의미함.

변수는 다음 시점 사이에서만 유효함.
- 선언된 시점부터
- 해당 스코프가 종료되는 시점까지

```rust
    {                      // s is not valid here, since it's not yet declared
        let s = "hello";   // s is valid from this point forward

        // do stuff with s
    }                      // this scope is now over, and s is no longer valid
```

- s는 문자열 리터럴을 가리킴
- 문자열 리터럴은 프로그램 코드에 하드코딩됨
- s는 선언 이후부터 스코프 종료 시점까지 유효함

스코프와 변수 유효성의 개념 자체는
다른 언어들과 큰 차이가 없음.


## 4. String 타입
문자열 리터럴의 특징은 다음과 같음.
- 컴파일 시점에 크기가 결정됨 ( 변경 불가능; immutable )
- 스코프 종료 시 자동 정리됨

그러나 소유권을 설명하기 위해서는 다음 조건을 만족하는 타입이 필요함.
- 실행 중 크기가 결정됨 ( 변경 가능; mutable )
- 메모리를 직접 관리함
이를 설명하기 위한 대표적인 타입이 String임.

```rust
    let mut s = String::from("hello");

    s.push_str(", world!"); // push_str() appends a literal to a String

    println!("{s}"); // this will print `hello, world!`
```


## 5. Memory and Allocation
### 5-1. 개요
String 타입을 사용하기 위해서는 다음 두 가지가 필요함.

1. 실행 중 필요한 메모리를 요청함
2. 사용이 끝난 메모리를 적절한 시점에 반환함

첫 번째는 `String::from` 호출 시 내부적으로 수행됨.
두 번째가 기존 언어에서 어려운 문제였음

### 5-2. 기존 언어의 메모리 해제 문제
GC가 없는 언어에서 메모리 관리는 다음과 같은 문제를 가짐.

- 해제를 잊으면 메모리 누수 발생
- 너무 빨리 해제하면 잘못된 메모리 접근 발생
- 두 번 해제하면 심각한 버그 발생

즉, 다음 조건을 항상 만족해야 함.
- **1회 할당 ↔ 1회 해제**
이를 사람이 직접 관리하는 것은 매우 어려움.

### 5-3. rust의 접근 방식 요약
Rust는 다음 원칙을 채택함.

- 메모리는 소유자에 의해 관리됨
- 소유자가 스코프를 벗어나는 시점에 자동 정리됨
- 이 동작은 컴파일 타임에 보장됨

이를 통해 Rust는 다음을 동시에 달성함.

- 메모리 안전성 확보
- 런타임 성능 손실 없음
- 명시적 GC 불필요

## 6. Copy & Move
### 6-1. 개요 
- Rust에서 heap 데이터를 다루는 타입은 기본적으로 **복사(copy)**가 아닌 **이동(move)** 발생
- Move는 메모리 안전성을 보장하기 위한 핵심 규칙 중 하나임
- `String` 타입을 통해 Move 동작 방식 이해 가능

### 6-2. Stack 데이터(예: 정수타입)와 Heap 데이터(예: String) 타입의 차이
#### Stack 데이터 >> Copy 발생
- 크기가 컴파일 타임에 고정됨
- 스택(stack)에만 저장됨
- 값 자체가 복사됨
- 원본 변수 사용 가능

```rust
let x = 5;
let y = x; // copy 발생
```

#### Heap 데이터 >> Move 발생
- 힙(heap)에 실제 데이터 저장
- 스택에는 포인터, 길이, 용량만 저장
- 기본 동작은 move
- 원본 변수 무효화됨

```rust
let s1 = String::from("hello");
let s2 = s1; // move 발생
```

**[ String 메모리 구조 ]**

| 위치    | 저장 내용                                  |
| ----- | -------------------------------------- |
| Stack | 포인터(pointer), 길이(length), 용량(capacity) |
| Heap  | 문자열 실제 데이터                             |

- s1 → s2 할당 시 스택 데이터만 복사됨
- 힙 데이터는 복사되지 않음
- 두 변수는 동일한 힙 데이터를 가리킴

![trpl04-02.svg](img/trpl04-02.svg)

![trpl04-04.svg](img/trpl04-04.svg)

**[ Move가 필요한 이유 ]**
- 문제 상황
    - 두 변수가 동일한 힙 메모리를 소유할 경우
    - 스코프 종료 시 동일 메모리를 두 번 해제 시도 가능
    - Double free error 발생 가능성
    - 메모리 손상 및 보안 취약점 위험

- Rust의 해결 방식
    - Move 발생 시 기존 변수 무효화
    - 소유권은 항상 하나의 변수만 가짐
    - 스코프 종료 시 단 한 번만 메모리 해제됨

- Move 이후 원본 변수 상태
    - s1은 move 이후 더 이상 유효하지 않음
    - 컴파일 타임에 오류 발생
    - 런타임 에러 사전 차단

```rust
let s1 = String::from("hello");
let s2 = s1;

println!("{s1}"); // 컴파일 에러
```

- Move와 Shallow Copy의 차이
    - Rust에서는 shallow copy라는 용어 대신 move 사용
    - 자동 deep copy는 절대 발생하지 않음
    - 성능 비용 없는 기본 동작 보장

| 구분     | Shallow Copy | Move (Rust) |
| ------ | ------------ | ----------- |
| 힙 데이터  | 공유           | 공유          |
| 원본 변수  | 유효           | 무효화         |
| 메모리 해제 | 위험 가능성       | 안전 보장       |
| 용어     | 개념적 분류       | Rust 핵심 규칙  |


## 7. Scope & Assignment
스코프 종료가 아닌 **재할당 시점**에서도 `drop` (메모리 해제 함수; rust 가 자동 호출) 발생 가능

```rust
let mut s = String::from("hello");
s = String::from("ahoy");
```

동작 단계별 정리
1. 변수 s 선언 및 "hello" 값 소유
2. 새로운 String("ahoy") 생성
3. s에 새로운 값 할당 ( 기존 변수가 소유한 값 위에 새로운 값 할당 가능 )
4. 기존 "hello" 값은 더 이상 참조되지 않음 ( 새로운 값 할당 시 기존 값은 즉시 소유권 상실 )
5. 기존 값에 대해 drop 즉시 호출 ( 기존 값은 스코프 종료를 기다리지 않고 즉시 해제 )
6. 힙 메모리 즉시 해제 ( Rust 컴파일러에 의해 자동 처리됨 )

![trpl04-05.svg](img/trpl04-05.svg)

## 8. Ownership and Functions

- 함수를 호출하면서 값을 전달하면 소유권 규칙 적용
- 전달되는 타입에 따라 move 또는 copy 발생
- heap 데이터를 소유하는 타입은 기본적으로 move 발생
- Copy 트레이트를 구현한 타입은 copy 발생

```rust
fn main() {
    let s = String::from("hello"); // s가 스코프에 들어옴
    takes_ownership(s);            // s의 값이 함수로 이동됨
                                  // 이후 s는 더 이상 유효하지 않음

    let x = 5;                     // x가 스코프에 들어옴
    makes_copy(x);                 // i32는 Copy 트레이트 구현
                                  // x는 이동되지 않음
} // x는 스코프 종료
  // s는 이미 이동되었으므로 특별한 동작 없음

fn takes_ownership(some_string: String) {
    println!("{some_string}");
} // some_string 스코프 종료 시 drop 호출
  // 힙 메모리 해제

fn makes_copy(some_integer: i32) {
    println!("{some_integer}");
} // some_integer 스코프 종료
  // 특별한 동작 없음
```

| 타입       | 함수 전달 시 동작 | 함수 종료 시   |
| -------- | ---------- | --------- |
| `String` | move 발생    | drop 호출   |
| `i32`    | copy 발생    | 단순 스코프 종료 |

## 9. Return Values and Scope

- 함수가 값을 반환하면 소유권이 호출자에게 이동
- 반환 값은 새로운 변수에 바인딩됨
- 반환된 값의 소유자는 호출한 쪽 변수임

```rust
fn main() {
    let s1 = gives_ownership();        // 반환 값이 s1으로 이동

    let s2 = String::from("hello");    // s2 스코프 진입

    let s3 = takes_and_gives_back(s2); // s2는 함수로 이동
                                       // 반환 값은 s3으로 이동
} // s3 drop 호출
  // s2는 이미 이동됨
  // s1 drop 호출

fn gives_ownership() -> String {
    let some_string = String::from("yours"); // 스코프 진입
    some_string                              // 반환과 함께 이동
}

fn takes_and_gives_back(a_string: String) -> String {
    a_string // 반환과 함께 이동
}
```

- 튜플을 이용한 다중 반환 ( 값 재사용을 위한 반환값 추가 )

```rust
fn main() {
    let s1 = String::from("hello");

    let (s2, len) = calculate_length(s1);

    println!("The length of '{s2}' is {len}.");
}

fn calculate_length(s: String) -> (String, usize) {
    let length = s.len();
    (s, length)
}
```

- 한계: 너무 많은 절차와 코드를 요구함
    -> 소유권을 이전하지 않고 값을 사용하는 기능이 있음! -> References and Borrowing
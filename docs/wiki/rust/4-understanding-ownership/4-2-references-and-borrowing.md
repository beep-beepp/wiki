---
title: "4-2. References and Borrowing"
parent: "4. Understanding Ownership"
grand_parent: Rust
permalink: /rust/understanding-ownership/references-and-borrowing/
nav_order: 2
---

## 1. Reference (참조) 란?

- ownership 이동 없이 값 접근 필요 상황 존재
- 값 자체가 아니라 주소를 빌려오는 방식

```rust
fn main() {
    let s1 = String::from("hello");
    let len = calculate_length(&s1); // &s1 : s1의 참조 생성

    println!("s1 사용 가능: {s1}");
}

fn calculate_length(s: &String) -> usize {
    // s는 String에 대한 참조(reference)임
    s.len()
}
// 여기서 s는 scope를 벗어나며 사라짐
// 하지만 s는 가리키는 값의 ownership을 가지고 있지 않기 때문에  
// 참조 대상인 String 자체는 drop되지 않음
```


## 2. Borrowing 란?

- reference를 만드는 행위 = borrowing 의미
- 값 소유권은 원래 변수에 남아있는 상태
- 참조는 임시 접근 권한만 가지는 구조

## 3. Immutable Reference 규칙

- 기본 reference는 immutable 상태 적용
- 빌린 값 수정 불가

```rust
fn change(some_string: &String) { // 수정 불가 상태
    some_string.push_str(", world"); // 컴파일 실패 ( mutable borrow 필요 )
}
```

- 값 변경 필요 시 `&mut`사용

```rust
fn main() {
    let mut s = String::from("hello"); // 원본 변수 mut 선언
    change(&mut s);
}

fn change(some_string: &mut String) { // 참조도 &mut 사용
    some_string.push_str(", world");
}
```

- mutable reference 제한 규칙
    - 규칙 1. 동시에 mutable reference 는 하나만 가능

    ```rust
    let mut s = String::from("hello");

    let r1 = &mut s;
    let r2 = &mut s; // 불가
    // 동시에 두 개가 수정 접근하면 데이터 레이스 위험 발생 -> rust 컴파일 단계에서 차단
    // (참고) 데이터 레이스란
    // - 여러 실행 흐름이 같은 데이터에 동시에 접근하는 상황
    // - 접근 순서가 예측 불가능해지는 문제 발생
    // - 결과가 매번 달라질 수 있는 위험 상태
    ```

    - 규칙 2. scope 분리 시 연속 borrow 가능

    ```rust
    let mut s = String::from("hello");

    {
        let r1 = &mut s;
    } // scope 종료 시 r1 borrow 해제

    let r2 = &mut s; // 이후 새로운 mutable reference 생성 가능
    ```

- immutable + mutable 혼용 제한
    
    ```rust
    let mut s = String::from("hello");

    let r1 = &s;
    let r2 = &s;

    let r3 = &mut s; // 불가
    // - 읽는 중에 값이 바뀌는 상황 방지
    // - 안전한 동시 접근 보장 ( 컴파일 단계에서 에러 )
    ```

## 4. reference scope 규칙
    - reference scope는 생성 시점부터 마지막 사용까지 지속
    - 단순히 블록 끝까지가 아니라 "마지막 사용 지점" 기준 적용

    ```rust
    let mut s = String::from("hello");

    let r1 = &s;
    let r2 = &s;

    println!("{r1} and {r2}"); // 마지막 사용

    let r3 = &mut s; // 가능
    println!("{r3}");
    ```

## 5. Dangling References ( 해제된 메모리를 참조 )
- rust 에서는 컴파일러가 dangling reference 가 절대 발생하지 않도록 보장
- 어떤 데이터에 대한 reference 가 존재한다면
- 그 데이터가 reference 보다 먼저 scope 를 벗어나 drop 되지 않도록 컴파일러가 강제

```rust
fn main() {
    let reference_to_nothing = dangle(); // &s 만 반환되고, 이미 사라진 메모리를 가리키게 됨 -> 컴파일 에러

    let solution = no_dangle(); // ownership 이 호출자에게 move
    // drop 발생하지 않고 값이 안전하게 유지됨
}

fn dangle() -> &String { // String reference 반환 시도
    let s = String::from("hello"); // 함수 내부에서 String 생성
    &s // s에 대한 reference 반환
} // 함수 종료 → s drop 발생

// 해결방법: ownership을 직접 반환
fn do_dangle() -> String {
    let s = String::from("hello");
    s
}
```
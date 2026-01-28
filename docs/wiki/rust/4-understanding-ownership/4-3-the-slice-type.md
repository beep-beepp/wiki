---
title: "4-3. The Slice Type"
parent: "4. Understanding Ownership"
grand_parent: Rust
permalink: /rust/understanding-ownership/the-slice-type/
nav_order: 3
---


## Slice 개념

- slice는 컬렉션(collection) 안에서 연속된 요소 구간을 참조하는 방식 의미  
- slice는 reference의 한 종류이므로 ownership을 가지지 않는 구조 존재  

정리

- slice = 부분 구간 참조  
- ownership 없음  
- 데이터는 원래 변수에 남아있는 상태  

---

## 문제 상황: 첫 단어 찾기 함수

문제 정의

- 공백으로 구분된 문자열 입력  
- 첫 번째 단어를 찾아 반환해야 하는 상황  

조건

- 공백이 없다면 전체 문자열이 한 단어 상태  

---

## Slice 없이 구현 시도

함수 시그니처 고민 발생

```rust
fn first_word(s: &String) -> ?
```

- ownership은 필요 없으므로 `&String` 사용 가능  
- 하지만 문자열의 "부분"을 반환할 타입 표현 어려움 존재  

대안

- 첫 단어 끝 위치 index를 반환하는 방식 시도 가능  

---

## byte index 반환 방식 구현

예시 코드

```rust
fn first_word(s: &String) -> usize {
    let bytes = s.as_bytes();

    for (i, &item) in bytes.iter().enumerate() {
        if item == b' ' {
            return i;
        }
    }

    s.len()
}
```

구조 설명

- 문자열을 byte 배열로 변환  
- 공백(`b' '`) 탐색  
- 공백 발견 시 index 반환  
- 공백 없으면 문자열 길이 반환  

---

## 문제점 발생

- 반환값이 단순 usize 숫자 상태  
- String과 분리된 값이라 future validity 보장 불가  

예시 코드

```rust
let mut s = String::from("hello world");
let word = first_word(&s); // word = 5

s.clear(); // String 내용 제거

// word는 여전히 5지만 의미 없는 값 상태
```

문제 핵심

- String 내용이 변했는데 index는 그대로 남아있는 상태  
- index와 실제 데이터 불일치 버그 발생 가능  

---

## Slice의 해결책: String Slice

- Rust는 string slice라는 해결책 제공  
- string slice는 String의 연속된 부분을 참조하는 구조  

예시

```rust
let s = String::from("hello world");

let hello = &s[0..5];
let world = &s[6..11];
```

![trpl04-07.svg](img/trpl04-07.svg)

정리

- `hello`는 전체 String이 아니라 일부 구간 참조  
- `[start..end]` 범위 지정 방식 사용  

slice 내부 정보

| 저장 값 | 의미 |
|--------|------|
| 시작 위치 pointer | 첫 요소 위치 |
| length | slice 길이 |

---

## Range 문법 축약 가능

### 시작이 0이면 생략 가능

```rust
&s[0..2]
&s[..2]
```

### 끝이 마지막이면 생략 가능

```rust
&s[3..len]
&s[3..]
```

### 전체 문자열 slice

```rust
&s[0..len]
&s[..]
```

---

## UTF-8 boundary 주의

- slice index는 UTF-8 문자 경계에 맞아야 하는 규칙 존재  
- 멀티바이트 문자 중간 slicing 시 runtime error 발생 가능  

---

## first_word 함수 slice 버전

반환 타입: `&str`

```rust
fn first_word(s: &String) -> &str {
    let bytes = s.as_bytes();

    for (i, &item) in bytes.iter().enumerate() {
        if item == b' ' {
            return &s[0..i];
        }
    }

    &s[..]
}
```

핵심 변화

- index 반환 대신 slice 반환  
- 반환값이 underlying data와 연결된 상태 유지  

---

## Slice의 장점

- String 상태 변화 시 invalid slice 발생을 컴파일러가 차단  
- index 기반 버그 원천 제거 가능  

예시

```rust
let mut s = String::from("hello world");
let word = first_word(&s);

s.clear(); // 컴파일 에러 발생
```

---

## String Literal도 Slice

```rust
let s = "Hello, world!";
```

- 타입은 `&str` 상태  
- 바이너리 안의 문자열 구간을 slice로 참조하는 구조  

특징

- immutable reference만 가능  

---

## String Slice를 파라미터로 받는 개선

기존

```rust
fn first_word(s: &String) -> &str
```

개선

```rust
fn first_word(s: &str) -> &str
```

장점

- `String`과 string literal 둘 다 입력 가능  
- 더 일반적이고 유연한 API 제공  

---

## Other Slices (배열 slice)

slice는 문자열 전용이 아닌 구조  

예시

```rust
let a = [1, 2, 3, 4, 5];
let slice = &a[1..3];

assert_eq!(slice, &[2, 3]);
```

타입

- `&[i32]` 형태 slice 가능  

구조 동일

- pointer + length 저장  
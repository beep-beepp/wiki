---
title: "3. Common Programming Concepts"
parent: "Rust"
permalink: /rust/common-programming-concepts
nav_order: 3
---

## 1. Variables and Mutability

### 1. 기본 개념

| 개념    | 설명                                |
| ----- | --------------------------------- |
| 변수    | 값을 저장하는 이름                        |
| 기본 성질 | Rust의 변수는 기본적으로 **불변(immutable)** |
| 변경 허용 | `mut` 키워드 사용                      |
| 안정성   | 불변성으로 버그 예방 가능                    |
| 병렬성   | 안전한 동시성에 유리함                      |

---

### 2. 불변 변수

* `let x = 5;` 형태
* 값 변경 불가
* 다시 대입하면 컴파일 에러 발생
* 코드 안정성 증가
* 추론 쉬움

예시:

```rust
let x = 5;
x = 6; // 에러 발생
```

의미:

* 한 번 정한 값은 유지됨
* 예측 가능함
* 안전함

---

### 3. 가변 변수 (`mut`)

* `let mut x = 5;` 형태
* 값 변경 가능
* 의도가 코드에 명확히 드러남
* 상태 변경이 필요한 경우 사용

예시:

```rust
let mut x = 5;
x = 6; // 정상 작동
```

효과:

* 값 업데이트 가능
* 유연함
* 하지만 변경 가능성 증가

---

### 4. 상수 (`const`)

| 특징     | 설명            |
| ------ | ------------- |
| 선언 키워드 | `const`       |
| mut 사용 | 불가            |
| 타입 명시  | 필수            |
| 범위     | 전역 또는 지역      |
| 값      | 컴파일 타임 상수만 가능 |

예시:

```rust
const THREE_HOURS_IN_SECONDS: u32 = 60 * 60 * 3;
```

장점:

* 의미가 명확함
* 한 곳에서 관리 가능
* 유지보수 쉬움

---

### 5. Shadowing (변수 가리기)

* 같은 이름의 새 변수를 다시 선언
* `let` 키워드 다시 사용
* 이전 변수는 가려짐
* 새로운 값과 타입 가능

예시:

```rust
fn main() {
    let x = 5;
    let x = x + 1;   // x = 6

    {
        let x = x * 2;  // x = 12
        println!("The value of x in the inner scope is: {x}"); // 12
    }

    println!("The value of x is: {x}"); // 6
}
```

특징:

* 타입 변경 가능
* 값 변환에 유리함
* 불변성 유지 가능

---

### 6. mut vs shadowing 비교

| 구분      | mut      | shadowing |
| ------- | -------- | --------- |
| 값 변경    | 가능       | 가능        |
| 타입 변경   | 불가능      | 가능        |
| let 재사용 | 불필요      | 필요        |
| 안전성     | 낮아질 수 있음 | 높음        |
| 용도      | 상태 변경    | 값 변환      |

---

### 7. 실전 예시

공백 개수 세기:

```rust
let spaces = "   ";
let spaces = spaces.len();
```

효과:

* 문자열 → 숫자로 변환
* 같은 이름 유지
* 코드 간결함

`mut`로 하면 실패:

```rust
let mut spaces = "   ";
spaces = spaces.len(); // 타입 불일치 에러
```

---

## 2. Data Types

### 1. 개요

* Rust는 **정적 타입 언어**임.
* 컴파일 시점에 **모든 변수의 타입이 확정**되어야 함.
* 필요한 경우 `: 타입`으로 **명시적 타입 어노테이션** 사용함.
* 예: `let guess: u32 = "42".parse().unwrap();` 형태로 지정함.
* 타입 명확화로 오류 감소됨. 편리함.

---

### 2. 타입 분류

| 구분       | 설명            |
| -------- | ------------- |
| Scalar   | 단일 값 표현함.     |
| Compound | 여러 값을 하나로 묶음. |

정확한 데이터 표현 가능함. 편리함.

---

### 3. Scalar Types

#### 3-1. Integer (정수)

##### [ 타입 표 ]

| Bit  | Signed | Unsigned |
| ---- | ------ | -------- |
| 8    | i8     | u8       |
| 16   | i16    | u16      |
| 32   | i32    | u32      |
| 64   | i64    | u64      |
| 128  | i128   | u128     |
| arch | isize  | usize    |

* `i` = signed(음수~양수 가능)
* `u` = unsigned(0 이상만)
* 기본 정수 타입은 `i32`임.
* 인덱스는 보통 `usize` 사용함. 편리함.

##### [ 정수 리터럴 ]

| 형태      | 예시            |
| ------- | ------------- |
| Decimal | `98_222`      |
| Hex     | `0xff`        |
| Octal   | `0o77`        |
| Binary  | `0b1111_0000` |
| Byte    | `b'A'`        |

가독성 위해 `_` 사용 가능함. 편리함.

##### [ Integer Overflow (정수 오버플로) ]

```
let x: u8 = 255;
x = 256; // ❌
```

* Debug 빌드 → overflow 시 panic 발생함.
* Release 빌드 → 값이 wrap-around 됨.
* 오버플로를 안전하게 다루기 위한 메서들

| 종류              | 동작               |
| --------------- | ---------------- |
| `wrapping_*`    | 항상 wrap          |
| `checked_*`     | overflow 시 None  |
| `overflowing_*` | (값, overflow 여부) |
| `saturating_*`  | 최대/최소로 고정        |

```
let x: u8 = 255;

x.wrapping_add(1);     // 0
x.checked_add(1);     // None
x.overflowing_add(1); // (0, true)
x.saturating_add(1);  // 255
```

---

#### 3-2. Floating Point (부동소수점)

| 타입  | 비트      |
| --- | ------- |
| f32 | 32      |
| f64 | 64 (기본) |

* IEEE-754 표준 사용함.
* 기본은 `f64`로 정확도 높음. 편리함. (요즘 CPU성능이 좋아서 f64가 더 효과적)

---

#### 3-3. Numeric Operations

| 연산 | 예       |
| -- | ------- |
| +  | `a + b` |
| -  | `a - b` |
| *  | `a * b` |
| /  | `a / b` |
| %  | `a % b` |

* 정수 나눗셈은 **0 방향으로 버림**됨.

| 계산      | 실제 값 | Rust 정수 나눗셈 결과 |
| ------- | ---- | -------------- |
| 5 / 2   | 2.5  | **2**          |
| -5 / 2  | -2.5 | **-2**         |


---

#### 3-4. Boolean

| 타입   | 값            |
| ---- | ------------ |
| bool | true / false |

* 조건문에 주로 사용함.
* 크기 1바이트임. 편리함.

---

#### 3-5. Character

* 타입: `char`
* 크기: 4바이트
* Unicode scalar value 지원함.
* 이모지, 한글, 특수문자 가능함. 편리함.

예:

```rust
let c = 'z';
let z = 'ℤ';
let cat = '😻';
```

---

### 4. Compound Types

#### 4-1. Tuple

* 서로 다른 타입 묶기 가능함.
* 크기 고정됨.

```rust
let tup: (i32, f64, u8) = (500, 6.4, 1);
```

##### [ 분해(destructuring) ]

```rust
let (x, y, z) = tup;
```

##### [ 인덱스 접근 ]

```rust
let a = tup.0;
let b = tup.1;
```

* 인덱스는 0부터 시작함. 편리함.

##### [ Unit Type ]

* `()` = 값 없음.
* 반환값 없는 함수의 기본 타입임. 편리함.

---

#### 4-2. Array

* 모든 요소가 **같은 타입**임.
* **길이 고정**됨.
* Stack에 저장됨.

```rust
let a = [1, 2, 3, 4, 5];
```

##### [ 타입 명시 ]

```rust
let a: [i32; 5] = [1, 2, 3, 4, 5];
```

##### [ 동일 값 초기화 ]

```rust
let a = [3; 5]; // [3,3,3,3,3]
```

##### [ 인덱스 접근 ]

```rust
let first = a[0];
```

* 범위 초과 접근 시 **panic 발생**함.
* 메모리 안전성 보장됨. 편리함.

---

### 5. Array vs Vector

| 구분  | Array     | Vector   |
| --- | --------- | -------- |
| 크기  | 고정 (컴파일 시점에 고정)        | 가변 (프로그램 실행 중에 요소 추가/제거 가능)       |
| 메모리 | Stack     | Heap     |
| 사용  | 요소 수 고정 시 | 일반적인 컬렉션 (예: 서버에서 데이터 받기) |

* 대부분 상황에서 `Vec<T>` 사용 권장됨.
* 고정 크기일 때만 Array 사용함.

<details>
<summary>Stack(스택) VS. Heap(힙)</summary>

| 차이는 "누가 언제 메모리를 해제하느냐" <br>

1️⃣ 스택: 컴파일러가 100% 관리 <br>

<pre><code class="language-rust">fn main() {
    let x = 10;
    let y = 20;
}
</code></pre>

이 코드는 이렇게 동작한다: <br>
main 시작 <br>
  → 스택에 x 자리 확보 <br>
  → 스택에 y 자리 확보 <br>
main 끝 <br>
  → x, y 자동 제거 <br>


특징: <br>
- 크기가 컴파일 타임에 정해짐 <br>
- 메모리는 함수 스코프와 완전히 일치 <br>
- drop을 부를 필요도 없음 <br>
- CPU는 스택 포인터만 움직이면 됨 → 극도로 빠름 <br>

Rust에서 스택 값: <br>
<pre><code class="language-rust">i32, f64, bool, (i32, i32), struct(힙 없는)
</code></pre>

2️⃣ 힙: 런타임이 관리, Rust는 "소유권"으로 통제 <br>
<pre><code class="language-rust">fn main() {
    let s = String::from("hello");
}
</code></pre>

실제 구조: <br>
<pre><code>스택:
s → (ptr, len, cap)

힙:
"hello"
</code></pre>

즉: <br>
- 스택에는 주소만 <br>
- 실제 데이터는 힙에 있음 <br>

이때 중요한 점: <br>
| Rust는 GC가 없기 때문에, 누가 이 힙 메모리를 해제할지 컴파일러가 추적해야 한다. <br>
그래서 등장하는 게 소유권(ownership). <br>

3️⃣ 힙 메모리 해제 규칙 <br>
<pre><code class="language-rust">{
    let s = String::from("hello"); // 힙 할당
} // ← 여기서 s가 drop됨 → 힙 메모리 free
</code></pre>

이 규칙: <br>
| "힙 데이터는 소유자가 스코프를 벗어나면 자동 해제된다." <br>
이게 Rust의 RAII 모델.

C: <br>
<pre><code class="language-c">malloc → free 안 하면 leak
</code></pre>

Java: <br>
<pre><code class="language-java">GC가 나중에 정리
</code></pre>

Rust: <br>
<pre><code class="language-rust">스코프 종료 = free
</code></pre>

4️⃣ 그래서 스택 vs 힙 관리 방식의 진짜 차이 <br>

<table>
<tr><th>구분</th><th>스택</th><th>힙</th></tr>
<tr><td>누가 관리?</td><td>CPU + 컴파일러</td><td>Rust 소유권 시스템</td></tr>
<tr><td>할당</td><td>스코프 시작</td><td>런타임 (<code>Box</code>, <code>String</code>, <code>Vec</code>)</td></tr>
<tr><td>해제</td><td>스코프 종료</td><td>소유자가 drop될 때</td></tr>
<tr><td>비용</td><td>거의 0</td><td>할당 + 해제 비용 있음</td></tr>
<tr><td>버그</td><td>거의 없음</td><td>double free, use-after-free → Rust가 차단</td></tr>
</table>

5️⃣ borrow가 왜 필요한가? <br>

<pre><code class="language-rust">let s = String::from("hello");
let r = &s;
</code></pre>

r은 힙 데이터를 가리키지만 소유자가 아님 <br>
그래서 Rust는 이렇게 규칙을 둔다: <br>
| "힙 메모리는 소유자만 해제할 수 있고, 참조자는 살아 있는 동안 소유자가 죽지 못한다." <br>
이게 borrow checker의 본질이다. <br>

6️⃣ 한 줄로 정리 <br>
스택은 '자동으로 쌓이고 사라지는 메모리'이고, <br>
힙은 '누가 소유하는지 추적해야 하는 메모리'다. <br>
Rust는 그 소유 추적을 컴파일 타임에 한다. <br>

이제 Box<T>, Rc<T>, Arc<T> 들어가면 <br>
"힙을 여러 명이 소유하는 방법"으로 확장된다. <br>
그게 Rust의 진짜 재미 구간 😄 <br>
- by. chatGPT <br>

</details>

## 3. Functions
### 1. 함수 기본 구조
- Rust에서 함수는 `fn` 키워드로 정의됨
- 함수 이름은 snake_case로 작성됨
- 함수 본문은 `{}` 로 감싸짐
- 함수는 정의 위치와 무관하게, 호출 스코프에서 보이면 사용 가능함

```rust
fn main() {
    another_function();
}

fn another_function() {
    println!("Another function.");
}
```

- `main` 함수가 프로그램의 시작점임
- `another_function()` 처럼 이름 뒤에 `()`로 호출함

### 2. 매개변수 (Parameters)
- 함수는 0개 이상의 매개변수를 가질 수 있음
- 모든 매개변수는 반드시 타입을 명시해야 함
- 호출 시 전달하는 값은 argument, 정의부의 변수는 parameter로 구분됨

```rust
fn another_function(x: i32) {
    println!("The value of x is: {x}");
}
```

- `x`는 매개변수, `i32`는 타입임
- `another_function(5)`에서 `5`는 argument임

여러 개도 가능함:

```rust
fn print_labeled_measurement(value: i32, unit_label: char) {
    println!("The measurement is: {value}{unit_label}");
}
```

- `value는` i32
- `unit_label`은 char
- 쉼표로 매개변수 구분함

### 3. Statements vs Expressions (매우 중요)

Rust는 Expression 기반 언어임
→ 코드의 대부분이 “값을 만들어내는 식(Expression)”으로 구성됨

####3.1 Statement (문)
- 어떤 동작을 수행하지만 값을 반환하지 않음
- 대표적인 예: let 문, 함수 정의

```rust
let y = 6;
```

- `let y = 6;` 은 statement임
- 이 문 자체는 값이 없음

따라서 아래 코드는 에러 발생함:

```rust
let x = (let y = 6);   // ❌
```

- `let y = 6`은 값이 없기 때문에
- x에 할당할 수 없음

C, Python, Ruby에서는 `x = y = 6` 이 가능하지만
Rust에서는 불가능함 → assignment는 값이 아님

#### 3.2 Expression (식)
- 계산되어 하나의 값을 만들어냄
- Rust 코드의 핵심 구성요소임

```rust
5 + 6        // 11을 반환하는 expression
```

다음은 모두 expression임:
- 숫자 리터럴 (`5`)
- 연산 (`x + 1`)
- 함수 호출 (`five()`)
- 매크로 호출 (`println!()`)
- 블록 `{ ... }`

#### 3.3 블록도 Expression임

```rust
let y = {
    let x = 3;
    x + 1
};
```

- `{ ... }` 전체가 하나의 expression임
- 마지막 줄 `x + 1` 이 4를 반환함
- 그 값이 `y`에 바인딩됨

즉,

```rust
{
    let x = 3;
    x + 1
}
```

→ 이 블록의 값은 4임

#### 3.4 세미콜론의 의미 (매우 중요)
- Expression 뒤에 세미콜론을 붙이면 Statement로 바뀜
- Statement는 값을 반환하지 않음

```rust
x + 1      // expression → 값 4 반환
x + 1;     // statement → 값 반환 안 함
```

→ 이 차이가 함수 반환과 직결됨

### 4. 반환값을 갖는 함수
- 함수 반환 타입은 -> 타입 으로 명시함
- Rust에서 함수의 반환값은 마지막 expression의 값임

```rust
fn five() -> i32 {
    5
}
```

- 5는 expression
- 세미콜론이 없으므로 반환됨
- 함수 전체의 값이 5가 됨

```rust
let x = five();   // x = 5
```

#### 4.1 또 다른 예

```rust
fn plus_one(x: i32) -> i32 {
    x + 1
}
```

- `x + 1`이 마지막 expression임
- 따라서 그 값이 자동으로 반환됨

#### 4.2 세미콜론을 붙이면 망가짐

```rust
fn plus_one(x: i32) -> i32 {
    x + 1;
}
```

- `x + 1;` 은 statement가 됨
- statement는 값이 없으므로 `()` 반환됨
- 함수 시그니처는 `i32`를 기대함
- 타입 불일치로 컴파일 에러 발생함

Rust 컴파일러는 친절하게
→ “세미콜론을 제거하라” 라고 알려줌

## 4. Control Flow
### 1. 제어 흐름 개요
- 프로그램은 조건에 따라 서로 다른 코드를 실행할 수 있어야 함
- 특정 조건이 유지되는 동안 코드를 반복 실행할 수 있어야 함
- Rust에서 제어 흐름의 핵심 구성요소는 `if` 와 `loop`, `while`, `for` 임

### 2. `if` 표현식
- `if`는 조건에 따라 코드 블록을 분기 실행함
- Rust에서 `if`는 **expression(식)** 임
- 조건은 반드시 `bool` 타입이어야 함

```rust
if number < 5 {
    println!("condition was true");
} else {
    println!("condition was false");
}
```

- 조건이 참이면 if 블록 실행됨
- 거짓이면 else 블록 실행됨
- else가 없으면 그냥 넘어감

#### 2.1 조건은 반드시 bool 이어야 함

```rust
if number { ... }   // ❌
```

- Rust는 숫자를 자동으로 true/false로 변환하지 않음
- 반드시 명시적으로 비교해야 함

```rust
if number != 0 { ... }   // ✅
```

### 3. `else if`를 이용한 다중 조건

```rust
if number % 4 == 0 { ... }
else if number % 3 == 0 { ... }
else if number % 2 == 0 { ... }
else { ... }
```

- 위에서부터 순서대로 조건 검사됨
- 첫 번째로 true가 되는 블록만 실행됨
- 나머지 조건은 검사되지 않음
- 너무 많은 else if 는 `match` 사용이 권장됨

### 4. `if`는 값(Expression)을 반환함
- Rust에서 `if`는 statement가 아니라 expression임
- 따라서 변수에 할당 가능함

```rust
let number = if condition { 5 } else { 6 };
```

- if 전체가 하나의 expression이 됨
- 실행된 블록의 마지막 값이 반환됨
- `number`에 그 값이 바인딩됨

#### 4.1 모든 분기 결과는 같은 타입이어야 함

```rust
let number = if condition { 5 } else { "six" };   // ❌
```

- if 블록은 i32
- else 블록은 &str
- Rust는 컴파일 시 타입이 확정되어야 하므로 에러 발생함

### 5. loop 반복문
- loop는 무한 반복을 수행함
- break로 탈출 가능함

```rust
loop {
    println!("again!");
}
```

- ctrl-C로 수동 종료 가능함

#### 5.1 `break`와 `continue`

- `break`는 반복문 종료함
- `continue`는 현재 반복을 건너뛰고 다음 반복으로 이동함

#### 5.2 loop는 값을 반환할 수 있음

```rust
let result = loop {
    if counter == 10 {
        break counter * 2;
    }
};
```

- `break 값` 형태로 루프에서 값 반환 가능함
- `result`에 해당 값이 바인딩됨
- 이 경우 `result = 20`이 됨

### 6. 루프 라벨 (Loop Labels)
- 중첩된 루프에서 바깥 루프를 종료하려면 라벨이 필요함

```rust
'counting_up: loop {
    loop {
        break 'counting_up;
    }
}
```

- 라벨은 `'이름` 형태로 정의됨
- `break 'label` 로 특정 루프 종료 가능함

### 7. while 반복문
- 조건이 true인 동안 반복 실행됨

```rust
while number != 0 {
    number -= 1;
}
```

- 조건이 false가 되면 자동 종료됨
- `loop + if + break` 패턴을 단순화한 문법임

### 8. 컬렉션 반복: `for` 루프
#### 8.1 while로 배열 순회하는 방식

```rust
while index < 5 {
    println!("{}", a[index]);
}
```

- 인덱스 관리가 필요함
- 배열 크기 변경 시 버그 발생 가능함
- 런타임 범위 검사로 느려질 수 있음

#### 8.2 `for`를 이용한 안전한 반복

```rust
for element in a {
    println!("{element}");
}
```

- 배열 크기를 자동으로 처리함
- 범위 초과 버그가 발생하지 않음
- 컴파일러가 더 최적화 가능함
- Rust에서 가장 권장되는 반복 방식임

### 9. 범위(Range)와 rev()

```rust
for number in (1..4).rev() {
    println!("{number}");
}
```

- `1..4`는 1,2,3 생성함
- `rev()`는 역순으로 순회함
- countdown 같은 로직에 자주 사용됨
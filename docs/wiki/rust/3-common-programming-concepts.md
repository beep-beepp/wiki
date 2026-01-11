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

[ 타입 표 ]

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

[ 정수 리터럴 ]

| 형태      | 예시            |
| ------- | ------------- |
| Decimal | `98_222`      |
| Hex     | `0xff`        |
| Octal   | `0o77`        |
| Binary  | `0b1111_0000` |
| Byte    | `b'A'`        |

가독성 위해 `_` 사용 가능함. 편리함.

[ Integer Overflow (정수 오버플로) ]

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

[ 분해(destructuring) ]

```rust
let (x, y, z) = tup;
```

[ 인덱스 접근 ]

```rust
let a = tup.0;
let b = tup.1;
```

* 인덱스는 0부터 시작함. 편리함.

[ Unit Type ]

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

[ 타입 명시 ]

```rust
let a: [i32; 5] = [1, 2, 3, 4, 5];
```

[ 동일 값 초기화 ]

```rust
let a = [3; 5]; // [3,3,3,3,3]
```

[ 인덱스 접근 ]

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

| 구분     | 스택         | 힙                                      |
| ------ | ---------- | -------------------------------------- |
| 누가 관리? | CPU + 컴파일러 | Rust 소유권 시스템                           |
| 할당     | 스코프 시작     | 런타임 (`Box`, `String`, `Vec`)           |
| 해제     | 스코프 종료     | 소유자가 drop될 때                           |
| 비용     | 거의 0       | 할당 + 해제 비용 있음                          |
| 버그     | 거의 없음      | double free, use-after-free → Rust가 차단 |

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
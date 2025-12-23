---
title: "2. Programming a Guessing Game"
parent: "Rust"
permalink: /rust/guessing-game
nav_order: 2
---

```rust
use std::io; // 표준 입출력 라이브러리 가져오기

fn main() {
    println!("Guess the number!"); // 문자열 출력 매크로
    println!("Please input your guess.");

    /* 
     * rust 에서 변수는 기본적으로 불변(immutable)임
     * 값을 변경하려면 mut 키워드를 사용해야 함
     */
    let mut guess = String::new(); // 빈 String으로 초기화된 가변 변수

    io::stdin()
        /* 
         * 참조( & )는 데이터를 복사하지 않고 접근하게 함
         * 기본적으로 불변이기 때문에 여기서는 &mut 사용
         */
        .read_line(&mut guess) // 사용자가 입력한 내용을 문자열에 추가
        /* 
         * read_line은 Result 타입을 반환
         * - Ok : 성공
         * - Err: 실패
         */
        .expect("Failed to read line"); // 오류가 발생하면 프로그램을 종료하고 메시지를 출력


    println!("You guessed: {guess}"); // {}로 값을 끼워넣음 (플레이스홀더)
}
```

- Cargo는 SemVer(시맨틱 비저닝)를 사용하여 호환 가능한 최신 버전을 자동으로 관리

```toml
[dependencies]
rand = "0.8.5"
```

  - 지정자 0.8.5는 실제로는 ^0.8.5의 축약형으로, 최소 0.8.5 이상이면서 0.9.0 미만인 모든 버전을 의미
  - `cargo update`를 통해 Cargo.lock 파일을 무시하고, Cargo.toml에 지정된 조건에 맞는 가장 최신 버전을 적용
    - 예시
    ```
    $ cargo update
    Updating crates.io index
        Locking 1 package to latest Rust 1.85.0 compatible version
        Updating rand v0.8.5 -> v0.8.6 (available: v0.9.0)
    ```

- `cargo doc --open`
    - 프로젝트가 의존하고 있는 모든 크레이트의 문서를 로컬에서 빌드한 뒤, 웹 브라우저에서 바로 열어줌
---
title: "2. Programming a Guessing Game"
parent: "Rust"
permalink: /rust/guessing-game
nav_order: 2
---

```rust
use std::io; // 표준 입출력 라이브러리 가져오기
use rand::Rng;
use std::cmp::Ordering; // cmp: compare

fn main() { // main함수: 모든 Rust 프로그램의 시작점
    println!("Guess the number!"); // 문자열 출력 매크로

    // 현재 스레드용 난수 생성기를 통한 비밀 숫자 생성
    let secret_number = rand::thread_rng().gen_range(1..=100);

    println!("The secret number is: {secret_number}");  // {}로 값을 끼워넣음 (플레이스홀더)

    loop { // 무한 반복문
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

        /*
        * trim(): 공백, 개행 제거 - 사용자가 숫자 입력하고 엔터 누르면 항상 \n이 포함됨
        * parse(): 문자열 > 숫자 변환
        * 변수 섀도잉(shadowing)을 사용해 guess 덮어씀
        */
        // 방법1 - 숫자 외 입력 시 에러 발생시키며 프로그램 종료
        let guess: u32 = guess.trim().parse().expect("Please type a number!");
        // 방법2 - 숫자 외 입력 시 그 입력을 무시하고 계속 추측할 수 있도록 함
        // parse가 Result 타입을 반환하고, 이를 직접 처리하는 방식
        let guess: u32 = guess.trim().parse() {
            Ok(num) => num,
            Err(_) => { // _ : 모든 에러
                println!("Please type a number!");
                continue;
            },
        };

        println!("You guessed: {guess}");

        match guess.cmp(&secret_number) {
            Ordering::Less => println!("Too small!"),
            Ordering::Greater => println!("Too big!"),
            Ordering::Equal => {
                println!("You win!");
                break; // 반복문 종료
            }
        }
    }
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
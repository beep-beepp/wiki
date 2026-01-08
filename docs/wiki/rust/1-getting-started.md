---
title: "1. Getting Started"
parent: "Rust"
permalink: /rust/getting-started
nav_order: 1
---

<details>
<summary>최근 Rust에 대한 반응 조사 결과</summary>

GitHub, Rust 공식 포럼, 검색 결과를 통해 수집한 최근 Rust 커뮤니티의 반응을 정리했습니다:<br>
<br>
<b>긍정적 반응</b><br>
<br>
1. 지속적인 성장과 인기<br>
- Stack Overflow 2025 조사에서 Rust는 가장 존경받는 프로그래밍 언어 1위 유지<br>
- GitHub 트렌딩에서 AI 및 고성능 도구들이 Rust로 개발되고 있음 (Deno, Turbo, Bevy 게임 엔진 등)<br>
- 일일 187-353개의 새로운 Star를 받는 프로젝트들이 계속 등장<br>
<br>
2. 실무 채택 확대<br>
- $400K 수준의 높은 개발자 급여 제시<br>
- 기업들이 성능 최적화와 안전성이 필요한 분야(시스템 소프트웨어, CLI 도구, 웹 서버)에 Rust 채택 중<br>
- 2025 State of Rust 조사에서 기초 소프트웨어 개발의 핵심 언어로 위치 강화<br>
<br>
3. 커뮤니티 활동<br>
- Rust 공식 포럼에서 일일 수십 개의 활발한 토론 (삶의 주기, 제네릭, 비동기 프로그래밍 등)<br>
- Google Summer of Code 2025에 많은 참가 프로젝트<br>
- 정기적인 버전 업데이트 (1.90, 1.91, 1.92)와 기능 추가<br>
<br>
<b>부정적 반응 및 우려사항</b><br>
<br>
1. 주류화 진입 지연 논쟁<br>
- Medium의 "Rust in 2025: Dead or Dormant?" 같은 기사에서 주류화 속도의 둔화 지적<br>
- Quora에서 "Rust가 주류로 가지 못했다"는 주장<br>
<br>
2. 주요 과제<br>
- 컴파일 시간: 여전히 개발 생산성의 걸림돌로 지적<br>
- 언어 복잡도: 높은 학습곡선으로 인한 초심자 진입장벽<br>
- 풀-스택 솔루션 부족: 웹 개발 분야에서 Python/JavaScript와의 경쟁에서 약한 평가<br>
<br>
3. 생태계 이슈<br>
- crates.io에서 악성 크레이트(phishing, malware) 지속 발견으로 보안 문제 지적<br>
- 일부 개발자들의 Rust 기능(예: 복잡한 lifetime 규칙)에 대한 불만<br>
<br>
<b>현재 트렌드 (2025년)</b><br>
- 기초 소프트웨어 포지셔닝: 10주년을 맞이하며 기초 소프트웨어 개발 언어로의 입지 강화<br>
- AI/고성능 도구 개발: LLM, 데이터 처리, 성능 최적화 프로젝트에 선호<br>
- 산업 채택 확대: 대형 기술 기업들의 Rust 투자 증가<br>
- 개발자 경험 개선: 컴파일 속도, 에러 메시지, 디버거 개선에 집중<br>
<br>
<b>종합 평가</b><br>
Rust는 "매우 존경받지만 제한적 채택" 상태입니다. 성능과 안전성이 필수적인 시스템 프로그래밍, CLI 도구, 고성능 서버/데이터 처리에서는 강세지만, 웹 개발이나 일반 애플리케이션 개발에서는 여전히 한계를 보이고 있습니다. 커뮤니티는 활발하고 긍정적이지만, 주류 언어로의 도약을 위해서는 개발 생산성 개선과 더 높은 추상화 레벨의 도구 필요성을 인식하고 있습니다.

</details>

### main
- 모든 실행 가능한 Rust 프로그램에서 가장 먼저 실행되는 코드

```
fn main() {
    println!("Hello, world!");
}
```

- 매크로 호출

println!은 Rust 매크로를 호출합니다.
만약 함수였다면 println처럼 ! 없이 작성되었을 것입니다.

Rust 매크로는 코드를 생성하는 코드로, Rust의 문법을 확장하는 역할을 합니다.
이에 대해서는 20장에서 자세히 다룹니다.

지금은 !가 붙어 있으면 함수가 아니라 매크로를 호출한다는 점만 기억하면 됩니다.


### 컴파일과 실행

Rust 프로그램을 실행하기 전에 반드시 컴파일해야 합니다.
다음과 같이 Rust 컴파일러인 rustc에 소스 파일 이름을 전달합니다.

$ rustc main.rs

컴파일이 성공하면 Rust는 실행 가능한 바이너리 파일을 생성합니다.
```
$ ls
main main.rs
```


<details>
<summary>컴파일 언어 vs 인터프리터 언어</summary>

Rust는 사전 컴파일(ahead-of-time) 언어입니다.
즉, 프로그램을 컴파일한 뒤 실행 파일을 다른 사람에게 전달하면,
상대방은 Rust가 설치되어 있지 않아도 프로그램을 실행할 수 있습니다.

</details>


### Cargo
- Rust의 빌드 시스템이자 패키지 관리자
  - 코드 빌드, 코드가 의존하는 라이브러리 다운로드, 그리고 해당 라이브러리 빌드와 같은 많은 작업을 대신 처리

```
$ cargo new hello_cargo
Cargo.toml
src/ 디렉터리
main.rs
````

- Cargo.toml

````
[package]
name = "hello_cargo"
version = "0.1.0"
edition = "2024"

[dependencies]
```


<details>
<summary>Cargo 명령어 모음</summary>

- `cargo new`: 프로젝트 생성<br>
- `cargo build`: 프로젝트 빌드<br>
- `cargo run`: 빌드와 실행을 한번에<br>
- `cargo check`: 실행 파일 없이 컴파일 오류를 확인<br>
- `cargo build --release`: 최적화를 적용하여 실행파일 생성<br>
    - 컴파일 시간은 더 오래 걸리지만, 실행 속도는 빨라짐<br>

</details>
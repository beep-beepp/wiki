
```rust
use std::{
    io::{BufReader, prelude::*},
    net::{TcpListener, TcpStream},
};

fn main() {
    let listner = TcpListener::bind("127.0.0.1:7878").unwrap();
    // TcpListener: TCP 연결 수신
    // bind: new 함수처럼 새로운 TcpListener 인스턴스 반환 ( 특정 포트에 연결해서 수신 대기하는 걸 보통 포트에 바인딩한다 라고 함 )
    // unwrap: 오류 발생 시 프로그램 종료

    for stream in listner.incoming() { // 스트림들의 순서를 제공하는 iterator 를 반환, 하나의 스트림: 클라이언트와 서버 사이의 연린 연결
        let stream = stream.unwrap();

        handle_connection(stream);
    }
}

fn handle_connection(mut stream: TcpStream) {
    let buf_reader = BufReader::new(&mut stream);
    let http_request: Vec<_> = buf_reader // 브라우저가 서버로 보내는 요청의 각 줄을 벡터에 모음
        .lines() // 데이터 스트림에서 줄바꿈 문자를 만날 때마다 분리하여 Result<String, std::io::Error> 형태의 iterator 를 반환
        .map(|result| result.unwrap()) // 각 String을 얻기 위해 map 으로 각 Result를 unwrap
        .take_while(|line| !line.is_empty()) // 브라우저는 HTTP 요청의 끝을 알리기 위해 연속된 두 개의 줄바꿈 문자를 보냄 -> 빈 줄이 나올 떄까지 줄을 가져오면 됨
        .collect();
    
    println!("Request: {http_request:#?}");

    let response = "HTTP/1.1 200 OK\r\n\r\n";
    
    stream.write_all(response.as_bytes()).unwrap();
}

```
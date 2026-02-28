# 15장. Tokio 심화

> **목표**: Tokio의 주요 기능을 활용하여 실전 비동기 프로그래밍을 할 수 있다.

---

## 1. Tokio 태스크 (spawn)

14장에서 `async/await`와 Tokio 런타임을 배웠습니다. 이번에는 **여러 비동기 작업을 동시에 실행**하는 방법을 알아봅시다.

### tokio::spawn()으로 태스크 생성

`tokio::spawn()`은 비동기 작업을 **별도의 태스크로 분리**해서 실행합니다.

```rust
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    // 태스크 생성: 백그라운드에서 실행됨
    let handle = tokio::spawn(async {
        sleep(Duration::from_secs(1)).await;
        println!("태스크 완료!");
        42 // 반환값
    });

    println!("태스크를 기다리는 중...");

    // JoinHandle로 결과 받기
    let result = handle.await.unwrap();
    println!("태스크 결과: {}", result);
}
```

```
태스크를 기다리는 중...
태스크 완료!
태스크 결과: 42
```

### 스레드와 태스크의 차이

13장에서 배운 스레드와 Tokio 태스크는 비슷해 보이지만 큰 차이가 있습니다.

| 구분 | std::thread::spawn | tokio::spawn |
|------|-------------------|-------------|
| 종류 | OS 스레드 (무거움) | 경량 태스크 (가벼움) |
| 메모리 | 약 8MB 스택 | 약 몇 KB |
| 생성 비용 | 높음 | 낮음 |
| 동시 실행 수 | 수백~수천 개 | **수만~수십만 개** |
| 용도 | CPU 집약적 작업 | I/O 대기가 많은 작업 |

> 웹서버는 수천 개의 동시 요청을 처리해야 합니다. OS 스레드로는 감당이 안 되지만, Tokio 태스크는 가볍기 때문에 가능합니다.

### JoinHandle로 결과 받기

`tokio::spawn()`은 `JoinHandle`을 반환합니다. `.await`하면 태스크의 결과를 받을 수 있습니다.

```rust
use tokio::time::{sleep, Duration};

async fn fetch_data(id: u32) -> String {
    sleep(Duration::from_millis(100)).await;
    format!("데이터-{}", id)
}

#[tokio::main]
async fn main() {
    let handle1 = tokio::spawn(fetch_data(1));
    let handle2 = tokio::spawn(fetch_data(2));

    // 각각의 결과를 받기
    let result1 = handle1.await.unwrap();
    let result2 = handle2.await.unwrap();

    println!("{}, {}", result1, result2);
    // 데이터-1, 데이터-2
}
```

> `JoinHandle::await`는 `Result`를 반환합니다. 태스크 내부에서 패닉이 발생하면 `Err`가 됩니다. `unwrap()`은 연습용이고, 실전에서는 에러를 적절히 처리하세요.

<details>
<summary><b>🔍 태스크 스케줄링 원리 (M:N 모델)</b></summary>

### M:N 스레딩 모델

Tokio는 **M:N 모델**을 사용합니다. M개의 태스크를 N개의 OS 스레드 위에서 실행합니다.

```
태스크 (M개, 경량)         OS 스레드 (N개, 보통 CPU 코어 수)
┌────────┐
│ 태스크1 │ ─┐
├────────┤   │         ┌──────────┐
│ 태스크2 │ ──┼────────→│ 스레드 1  │
├────────┤   │         └──────────┘
│ 태스크3 │ ─┘
├────────┤             ┌──────────┐
│ 태스크4 │ ──────────→│ 스레드 2  │
├────────┤             └──────────┘
│ 태스크5 │ ─┐
├────────┤   │         ┌──────────┐
│ 태스크6 │ ──┼────────→│ 스레드 3  │
├────────┤   │         └──────────┘
│ ...    │ ─┘
└────────┘
```

### 동작 방식

1. Tokio 런타임이 시작되면 **워커 스레드 풀**을 만듭니다 (기본: CPU 코어 수)
2. `tokio::spawn()`으로 태스크를 만들면 **큐에 등록**됩니다
3. 태스크가 `.await`에서 멈추면 (I/O 대기 등) 스레드를 **다른 태스크에게 양보**합니다
4. I/O가 완료되면 태스크가 다시 **큐에 들어가서** 실행을 재개합니다

### 왜 효율적인가?

```
전통적 스레드 모델 (1:1):
스레드1: [작업] [====I/O 대기====] [작업]     ← 대기 중 스레드 낭비
스레드2: [작업] [====I/O 대기====] [작업]     ← 대기 중 스레드 낭비

Tokio M:N 모델:
스레드1: [태스크A] [태스크B] [태스크C] [태스크A계속] ← 쉬지 않고 일함
스레드2: [태스크D] [태스크E] [태스크F] [태스크D계속] ← 쉬지 않고 일함
```

태스크가 I/O를 기다리는 동안 스레드가 놀지 않고 다른 태스크를 처리합니다. 이것이 비동기 프로그래밍이 웹서버에서 효율적인 핵심 이유입니다.

### 주의사항

태스크 안에서 **오래 걸리는 동기 작업**(CPU 집약적 계산 등)을 하면, 그 스레드를 독점해서 다른 태스크가 실행되지 못합니다. 이런 경우에는 `tokio::task::spawn_blocking()`을 사용합니다.

```rust
// CPU 집약적 작업은 spawn_blocking으로!
let result = tokio::task::spawn_blocking(|| {
    // 무거운 계산 (별도의 블로킹 스레드에서 실행)
    heavy_computation()
}).await.unwrap();
```

</details>

---

## 2. 여러 태스크 동시 실행

### tokio::join! - 모두 완료될 때까지 대기

`tokio::join!`은 여러 비동기 작업을 **동시에 실행**하고, **모두 끝날 때까지** 기다립니다.

```rust
use tokio::time::{sleep, Duration};

async fn fetch_user() -> String {
    sleep(Duration::from_secs(2)).await;
    String::from("홍길동")
}

async fn fetch_orders() -> Vec<String> {
    sleep(Duration::from_secs(1)).await;
    vec![String::from("주문1"), String::from("주문2")]
}

async fn fetch_settings() -> String {
    sleep(Duration::from_secs(1)).await;
    String::from("다크모드")
}

#[tokio::main]
async fn main() {
    // 순차 실행이면 2 + 1 + 1 = 4초
    // join!으로 동시 실행하면 가장 오래 걸리는 것 = 2초
    let (user, orders, settings) = tokio::join!(
        fetch_user(),
        fetch_orders(),
        fetch_settings(),
    );

    println!("사용자: {}", user);
    println!("주문: {:?}", orders);
    println!("설정: {}", settings);
}
```

```
사용자: 홍길동
주문: ["주문1", "주문2"]
설정: 다크모드
```

> `join!`은 모든 작업이 끝나야 다음으로 넘어갑니다. 하나가 오래 걸리면 전체가 기다립니다.

### 순차 vs 동시 실행 비교

```rust
use tokio::time::{sleep, Duration, Instant};

async fn task_a() -> &'static str {
    sleep(Duration::from_secs(1)).await;
    "A 완료"
}

async fn task_b() -> &'static str {
    sleep(Duration::from_secs(2)).await;
    "B 완료"
}

#[tokio::main]
async fn main() {
    // 순차 실행: 1초 + 2초 = 3초
    let start = Instant::now();
    let a = task_a().await;
    let b = task_b().await;
    println!("[순차] {}, {} ({}초)", a, b, start.elapsed().as_secs());

    // 동시 실행: max(1초, 2초) = 2초
    let start = Instant::now();
    let (a, b) = tokio::join!(task_a(), task_b());
    println!("[동시] {}, {} ({}초)", a, b, start.elapsed().as_secs());
}
```

```
[순차] A 완료, B 완료 (3초)
[동시] A 완료, B 완료 (2초)
```

### tokio::select! - 가장 빨리 끝나는 것 선택

`tokio::select!`는 여러 비동기 작업 중 **가장 먼저 완료되는 하나만** 선택합니다. 나머지는 취소됩니다.

```rust
use tokio::time::{sleep, Duration};

async fn fast_server() -> String {
    sleep(Duration::from_millis(100)).await;
    String::from("빠른 서버 응답")
}

async fn slow_server() -> String {
    sleep(Duration::from_secs(5)).await;
    String::from("느린 서버 응답")
}

#[tokio::main]
async fn main() {
    // 두 서버 중 먼저 응답하는 쪽을 사용
    tokio::select! {
        result = fast_server() => {
            println!("선택됨: {}", result);
        }
        result = slow_server() => {
            println!("선택됨: {}", result);
        }
    }
    // 선택됨: 빠른 서버 응답
    // (slow_server는 자동으로 취소됨)
}
```

### join! vs select! 비교

| 매크로 | 동작 | 비유 |
|--------|------|------|
| `join!` | 모든 작업 완료 대기 | 모든 선수가 결승선에 도착할 때까지 대기 |
| `select!` | 가장 빠른 하나만 선택 | 첫 번째 선수가 결승선에 도착하면 경기 종료 |

### 실용 예제: 여러 API를 동시에 호출하는 상황

웹서버에서 사용자 정보를 보여줄 때, 여러 데이터를 동시에 가져오는 패턴입니다.

```rust
use tokio::time::{sleep, Duration};

// 각각 다른 서비스에서 데이터를 가져온다고 가정
async fn get_profile(user_id: u32) -> String {
    sleep(Duration::from_millis(200)).await;
    format!("프로필(ID:{})", user_id)
}

async fn get_recent_posts(user_id: u32) -> Vec<String> {
    sleep(Duration::from_millis(300)).await;
    vec![
        format!("글1(작성자:{})", user_id),
        format!("글2(작성자:{})", user_id),
    ]
}

async fn get_notifications(user_id: u32) -> u32 {
    sleep(Duration::from_millis(100)).await;
    5 // 알림 5개
}

#[tokio::main]
async fn main() {
    let user_id = 42;

    // 3개의 API를 동시에 호출
    let (profile, posts, notification_count) = tokio::join!(
        get_profile(user_id),
        get_recent_posts(user_id),
        get_notifications(user_id),
    );

    println!("=== 사용자 대시보드 ===");
    println!("{}", profile);
    println!("최근 글: {:?}", posts);
    println!("알림: {}개", notification_count);
}
```

```
=== 사용자 대시보드 ===
프로필(ID:42)
최근 글: ["글1(작성자:42)", "글2(작성자:42)"]
알림: 5개
```

> 순차 실행이면 200 + 300 + 100 = 600ms 걸리지만, `join!`으로 동시 실행하면 300ms면 됩니다. 웹서버에서 응답 속도를 줄이는 핵심 기법입니다.

---

## 3. 비동기 I/O

### tokio::fs - 비동기 파일 읽기/쓰기

Rust 표준 라이브러리의 `std::fs`는 동기 방식입니다. Tokio는 비동기 버전인 `tokio::fs`를 제공합니다.

### 비동기 파일 쓰기와 읽기

```rust
use tokio::fs;

#[tokio::main]
async fn main() {
    // 파일 쓰기
    fs::write("hello.txt", "안녕하세요, Tokio!").await.unwrap();
    println!("파일 쓰기 완료");

    // 파일 읽기
    let content = fs::read_to_string("hello.txt").await.unwrap();
    println!("파일 내용: {}", content);

    // 파일 삭제
    fs::remove_file("hello.txt").await.unwrap();
    println!("파일 삭제 완료");
}
```

```
파일 쓰기 완료
파일 내용: 안녕하세요, Tokio!
파일 삭제 완료
```

### 동기 I/O와 비동기 I/O 비교

```rust
use tokio::fs;
use tokio::time::Instant;

async fn read_files_sequential() {
    // 순차로 3개 파일 읽기
    let _a = fs::read_to_string("file_a.txt").await.unwrap();
    let _b = fs::read_to_string("file_b.txt").await.unwrap();
    let _c = fs::read_to_string("file_c.txt").await.unwrap();
}

async fn read_files_concurrent() {
    // 동시에 3개 파일 읽기
    let (_a, _b, _c) = tokio::join!(
        fs::read_to_string("file_a.txt"),
        fs::read_to_string("file_b.txt"),
        fs::read_to_string("file_c.txt"),
    );
}
```

| 방식 | 코드 | 설명 |
|------|------|------|
| 동기 (`std::fs`) | `std::fs::read_to_string()` | 파일을 읽는 동안 스레드가 멈춤 |
| 비동기 순차 | `tokio::fs::read_to_string().await` | 파일을 읽는 동안 다른 태스크 실행 가능 |
| 비동기 동시 | `tokio::join!`으로 여러 파일 | 여러 파일을 동시에 읽기 |

> 웹서버에서는 `tokio::fs`를 사용해야 합니다. `std::fs`를 쓰면 파일을 읽는 동안 다른 요청 처리가 멈춥니다.

<details>
<summary><b>🔍 비동기 I/O가 왜 빠른가? (원리)</b></summary>

### 동기 I/O의 문제

동기 I/O에서는 파일을 읽거나 네트워크 응답을 기다리는 동안 **스레드가 아무것도 하지 못합니다**.

```
동기 방식 (스레드 1개):
[파일A 읽기 시작]---[대기중...]---[완료][파일B 읽기 시작]---[대기중...]---[완료]
                    ↑ 이 시간 낭비!
```

### 비동기 I/O의 해결

비동기 I/O에서는 OS에게 "읽기 시작해줘, 끝나면 알려줘"라고 요청한 뒤, **다른 작업을 먼저 처리**합니다.

```
비동기 방식 (스레드 1개):
[파일A 요청][파일B 요청][다른 작업...][파일A 완료 처리][파일B 완료 처리]
            ↑ 대기 없이 바로 다음 요청!
```

### OS 수준의 동작

Tokio는 OS가 제공하는 비동기 I/O 메커니즘을 활용합니다:
- **Linux**: epoll
- **macOS**: kqueue
- **Windows**: IOCP (I/O Completion Ports)

이들은 모두 "여러 I/O 작업을 등록해두고, 완료된 것만 알려주는" 방식입니다. 덕분에 하나의 스레드로 수천 개의 동시 연결을 처리할 수 있습니다.

</details>

---

## 4. 타이머와 타임아웃

### tokio::time::sleep - 비동기 대기

`std::thread::sleep()`은 스레드를 멈추지만, `tokio::time::sleep()`은 **태스크만 멈추고 스레드는 다른 작업을 할 수 있습니다**.

```rust
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    println!("시작!");
    sleep(Duration::from_secs(2)).await; // 2초 대기 (스레드는 자유)
    println!("2초 후!");
}
```

> **주의**: 비동기 코드 안에서 `std::thread::sleep()`을 쓰면 스레드 전체가 멈춰서 다른 태스크도 실행되지 못합니다. 반드시 `tokio::time::sleep()`을 쓰세요.

### tokio::time::timeout - 시간 제한

느린 작업에 **시간 제한**을 걸 수 있습니다. 시간 내에 끝나지 않으면 에러를 반환합니다.

```rust
use tokio::time::{timeout, Duration};

async fn slow_operation() -> String {
    tokio::time::sleep(Duration::from_secs(10)).await;
    String::from("완료!")
}

#[tokio::main]
async fn main() {
    // 5초 안에 끝나지 않으면 타임아웃
    let result = timeout(Duration::from_secs(5), slow_operation()).await;

    match result {
        Ok(value) => println!("성공: {}", value),
        Err(_) => println!("시간 초과!"),
    }
    // 시간 초과! (slow_operation은 10초가 걸리니까)
}
```

### 실용 예제: API 호출에 타임아웃 적용

```rust
use tokio::time::{timeout, sleep, Duration};

async fn call_external_api() -> String {
    // 외부 API 호출을 시뮬레이션
    sleep(Duration::from_secs(3)).await;
    String::from("API 응답 데이터")
}

async fn fetch_with_timeout() -> Result<String, String> {
    let result = timeout(
        Duration::from_secs(5),
        call_external_api()
    ).await;

    match result {
        Ok(data) => Ok(data),
        Err(_) => Err(String::from("API 호출 시간 초과")),
    }
}

#[tokio::main]
async fn main() {
    match fetch_with_timeout().await {
        Ok(data) => println!("받은 데이터: {}", data),
        Err(err) => println!("에러: {}", err),
    }
}
```

### tokio::time::interval - 주기적 실행

일정 간격으로 작업을 반복 실행합니다.

```rust
use tokio::time::{interval, Duration};

#[tokio::main]
async fn main() {
    let mut ticker = interval(Duration::from_secs(1));
    let mut count = 0;

    loop {
        ticker.tick().await; // 1초마다 실행
        count += 1;
        println!("[{}초] 상태 점검 중...", count);

        if count >= 5 {
            println!("점검 완료!");
            break;
        }
    }
}
```

```
[1초] 상태 점검 중...
[2초] 상태 점검 중...
[3초] 상태 점검 중...
[4초] 상태 점검 중...
[5초] 상태 점검 중...
점검 완료!
```

> `interval`은 웹서버에서 헬스체크, 캐시 갱신, 로그 주기적 기록 등에 활용됩니다.

---

## 5. 비동기 채널

13장에서 `std::sync::mpsc` 채널을 배웠습니다. Tokio는 비동기 환경에 맞는 채널들을 제공합니다.

### tokio::sync::mpsc - 다중 생산자, 단일 소비자

가장 많이 쓰이는 채널입니다. 여러 곳에서 메시지를 보내고, 한 곳에서 받습니다.

```rust
use tokio::sync::mpsc;

#[tokio::main]
async fn main() {
    // 버퍼 크기 32인 채널 생성
    let (tx, mut rx) = mpsc::channel::<String>(32);

    // 생산자 1
    let tx1 = tx.clone();
    tokio::spawn(async move {
        tx1.send(String::from("안녕 from 태스크1")).await.unwrap();
    });

    // 생산자 2
    let tx2 = tx.clone();
    tokio::spawn(async move {
        tx2.send(String::from("안녕 from 태스크2")).await.unwrap();
    });

    // 원본 tx를 버려야 채널이 닫힘
    drop(tx);

    // 소비자: 메시지 받기
    while let Some(message) = rx.recv().await {
        println!("받음: {}", message);
    }
    println!("채널 닫힘, 모든 메시지 처리 완료");
}
```

```
받음: 안녕 from 태스크1
받음: 안녕 from 태스크2
채널 닫힘, 모든 메시지 처리 완료
```

### tokio::sync::oneshot - 일회용 채널

딱 **한 번만** 메시지를 보내는 채널입니다. 태스크의 결과를 돌려받을 때 유용합니다.

```rust
use tokio::sync::oneshot;

#[tokio::main]
async fn main() {
    let (tx, rx) = oneshot::channel::<String>();

    // 태스크가 작업을 끝내면 결과를 보냄
    tokio::spawn(async move {
        let result = String::from("계산 결과: 42");
        tx.send(result).unwrap();
    });

    // 결과를 기다림
    let value = rx.await.unwrap();
    println!("{}", value); // 계산 결과: 42
}
```

### tokio::sync::broadcast - 다중 구독

하나의 메시지를 **여러 수신자에게 동시에** 보냅니다.

```rust
use tokio::sync::broadcast;

#[tokio::main]
async fn main() {
    let (tx, _) = broadcast::channel::<String>(16);

    // 구독자 2명 생성
    let mut rx1 = tx.subscribe();
    let mut rx2 = tx.subscribe();

    // 메시지 발행
    tx.send(String::from("긴급 공지!")).unwrap();

    // 두 구독자 모두 같은 메시지를 받음
    println!("구독자1: {}", rx1.recv().await.unwrap());
    println!("구독자2: {}", rx2.recv().await.unwrap());
}
```

```
구독자1: 긴급 공지!
구독자2: 긴급 공지!
```

### 채널 종류 비교

| 채널 | 생산자 | 소비자 | 메시지 수 | 용도 |
|------|--------|--------|----------|------|
| `mpsc` | 여러 개 | 1개 | 여러 번 | 작업 큐, 이벤트 전달 |
| `oneshot` | 1개 | 1개 | 1번 | 결과 반환, 응답 대기 |
| `broadcast` | 1개 | 여러 개 | 여러 번 | 공지, 이벤트 브로드캐스트 |

### 웹서버에서의 활용 예시

```rust
use tokio::sync::mpsc;

// 로그를 비동기로 처리하는 패턴
struct LogMessage {
    level: String,
    message: String,
}

async fn log_worker(mut rx: mpsc::Receiver<LogMessage>) {
    while let Some(log) = rx.recv().await {
        // 실제로는 파일에 쓰거나, 외부 서비스로 전송
        println!("[{}] {}", log.level, log.message);
    }
}

#[tokio::main]
async fn main() {
    let (tx, rx) = mpsc::channel::<LogMessage>(100);

    // 로그 워커를 백그라운드에서 실행
    tokio::spawn(log_worker(rx));

    // 여러 태스크에서 로그 전송
    let tx1 = tx.clone();
    tokio::spawn(async move {
        tx1.send(LogMessage {
            level: String::from("INFO"),
            message: String::from("서버 시작됨"),
        }).await.unwrap();
    });

    let tx2 = tx.clone();
    tokio::spawn(async move {
        tx2.send(LogMessage {
            level: String::from("WARN"),
            message: String::from("메모리 사용량 높음"),
        }).await.unwrap();
    });

    // 잠시 대기 (로그가 처리될 시간)
    tokio::time::sleep(tokio::time::Duration::from_millis(100)).await;
}
```

```
[INFO] 서버 시작됨
[WARN] 메모리 사용량 높음
```

---

## 6. 실전 패턴 정리

### 패턴 1: spawn + channel 조합

태스크에서 작업을 수행하고, 채널로 결과를 전달하는 패턴입니다.

```rust
use tokio::sync::mpsc;

async fn process_item(item: u32) -> u32 {
    tokio::time::sleep(tokio::time::Duration::from_millis(100)).await;
    item * 2
}

#[tokio::main]
async fn main() {
    let items = vec![1, 2, 3, 4, 5];
    let (tx, mut rx) = mpsc::channel::<u32>(10);

    // 각 아이템을 별도 태스크에서 처리
    for item in items {
        let tx = tx.clone();
        tokio::spawn(async move {
            let result = process_item(item).await;
            tx.send(result).await.unwrap();
        });
    }

    // 원본 tx 제거
    drop(tx);

    // 결과 수집
    let mut results = Vec::new();
    while let Some(result) = rx.recv().await {
        results.push(result);
    }

    results.sort();
    println!("처리 결과: {:?}", results); // [2, 4, 6, 8, 10]
}
```

### 패턴 2: timeout으로 안전하게 감싸기

외부 호출에는 항상 타임아웃을 걸어서, 서버가 무한 대기하는 것을 방지합니다.

```rust
use tokio::time::{timeout, Duration};

async fn safe_api_call(url: &str) -> Result<String, String> {
    let result = timeout(
        Duration::from_secs(10),
        simulate_api_call(url),
    ).await;

    match result {
        Ok(Ok(data)) => Ok(data),                          // 성공
        Ok(Err(e)) => Err(format!("API 에러: {}", e)),      // API 에러
        Err(_) => Err(String::from("타임아웃: 10초 초과")),  // 시간 초과
    }
}

async fn simulate_api_call(url: &str) -> Result<String, String> {
    tokio::time::sleep(Duration::from_secs(1)).await;
    Ok(format!("{}에서 받은 데이터", url))
}

#[tokio::main]
async fn main() {
    match safe_api_call("https://api.example.com").await {
        Ok(data) => println!("성공: {}", data),
        Err(err) => println!("실패: {}", err),
    }
}
```

### 패턴 3: Graceful Shutdown (우아한 종료)

서버를 종료할 때, 진행 중인 작업을 마무리한 뒤 종료하는 패턴입니다.

```rust
use tokio::sync::broadcast;
use tokio::time::{sleep, Duration};

async fn worker(id: u32, mut shutdown: broadcast::Receiver<()>) {
    loop {
        tokio::select! {
            _ = shutdown.recv() => {
                println!("워커 {}: 종료 신호 수신, 정리 중...", id);
                // 정리 작업 수행
                sleep(Duration::from_millis(100)).await;
                println!("워커 {}: 정리 완료", id);
                return;
            }
            _ = sleep(Duration::from_secs(1)) => {
                println!("워커 {}: 작업 처리 중...", id);
            }
        }
    }
}

#[tokio::main]
async fn main() {
    let (shutdown_tx, _) = broadcast::channel::<()>(1);

    // 워커 3개 시작
    for id in 1..=3 {
        let rx = shutdown_tx.subscribe();
        tokio::spawn(worker(id, rx));
    }

    // 3초 동안 실행
    sleep(Duration::from_secs(3)).await;

    // 종료 신호 보내기
    println!("\n=== 종료 신호 전송 ===");
    shutdown_tx.send(()).unwrap();

    // 워커들이 정리할 시간
    sleep(Duration::from_millis(500)).await;
    println!("서버 종료 완료");
}
```

> Graceful Shutdown은 실제 웹서버에서 매우 중요합니다. 갑자기 종료하면 처리 중인 요청이 실패하기 때문입니다. Axum(17장)에서도 이 패턴을 사용합니다.

---

## 7. 실습

### 실습 1: 3개의 태스크를 동시에 실행하고 결과 모으기 (join!)

3개의 비동기 함수를 `tokio::join!`으로 동시에 실행하고, 모든 결과를 출력하세요.

```rust
use tokio::time::{sleep, Duration, Instant};

async fn download_file(name: &str, seconds: u64) -> String {
    sleep(Duration::from_secs(seconds)).await;
    format!("{} 다운로드 완료 ({}초)", name, seconds)
}

#[tokio::main]
async fn main() {
    // 여기에 코드를 작성하세요!
    // 1. download_file을 3번 호출하세요: ("음악.mp3", 2), ("영상.mp4", 3), ("문서.pdf", 1)
    // 2. tokio::join!으로 동시에 실행하세요
    // 3. 각 결과와 총 소요 시간을 출력하세요

    // 예상 출력:
    // 음악.mp3 다운로드 완료 (2초)
    // 영상.mp4 다운로드 완료 (3초)
    // 문서.pdf 다운로드 완료 (1초)
    // 총 소요 시간: 3초 (순차면 6초였을 것!)
}
```

<details>
<summary><b>정답 보기</b></summary>

```rust
use tokio::time::{sleep, Duration, Instant};

async fn download_file(name: &str, seconds: u64) -> String {
    sleep(Duration::from_secs(seconds)).await;
    format!("{} 다운로드 완료 ({}초)", name, seconds)
}

#[tokio::main]
async fn main() {
    let start = Instant::now();

    let (music, video, doc) = tokio::join!(
        download_file("음악.mp3", 2),
        download_file("영상.mp4", 3),
        download_file("문서.pdf", 1),
    );

    println!("{}", music);
    println!("{}", video);
    println!("{}", doc);
    println!("총 소요 시간: {}초 (순차면 6초였을 것!)", start.elapsed().as_secs());
}
```

</details>

---

### 실습 2: timeout으로 느린 작업에 시간 제한 걸기

비동기 작업에 타임아웃을 적용하고, 성공/시간 초과를 처리하세요.

```rust
use tokio::time::{timeout, sleep, Duration};

async fn slow_database_query() -> String {
    sleep(Duration::from_secs(3)).await;
    String::from("쿼리 결과: 42개의 레코드")
}

async fn fast_cache_lookup() -> String {
    sleep(Duration::from_millis(100)).await;
    String::from("캐시 결과: 42개의 레코드")
}

#[tokio::main]
async fn main() {
    // 여기에 코드를 작성하세요!
    // 1. slow_database_query()에 2초 타임아웃을 적용하세요 (시간 초과됨)
    // 2. fast_cache_lookup()에 2초 타임아웃을 적용하세요 (성공)
    // 3. 각 결과를 match로 처리하세요

    // 예상 출력:
    // [DB 쿼리] 시간 초과! 2초 안에 응답하지 않았습니다.
    // [캐시 조회] 성공: 캐시 결과: 42개의 레코드
}
```

<details>
<summary><b>정답 보기</b></summary>

```rust
use tokio::time::{timeout, sleep, Duration};

async fn slow_database_query() -> String {
    sleep(Duration::from_secs(3)).await;
    String::from("쿼리 결과: 42개의 레코드")
}

async fn fast_cache_lookup() -> String {
    sleep(Duration::from_millis(100)).await;
    String::from("캐시 결과: 42개의 레코드")
}

#[tokio::main]
async fn main() {
    // 1. DB 쿼리에 2초 타임아웃
    let db_result = timeout(
        Duration::from_secs(2),
        slow_database_query(),
    ).await;

    match db_result {
        Ok(data) => println!("[DB 쿼리] 성공: {}", data),
        Err(_) => println!("[DB 쿼리] 시간 초과! 2초 안에 응답하지 않았습니다."),
    }

    // 2. 캐시 조회에 2초 타임아웃
    let cache_result = timeout(
        Duration::from_secs(2),
        fast_cache_lookup(),
    ).await;

    match cache_result {
        Ok(data) => println!("[캐시 조회] 성공: {}", data),
        Err(_) => println!("[캐시 조회] 시간 초과! 2초 안에 응답하지 않았습니다."),
    }
}
```

</details>

---

### 실습 3: mpsc 채널로 생산자-소비자 패턴 구현

3개의 생산자 태스크가 메시지를 보내고, 소비자가 받아서 처리하세요.

```rust
use tokio::sync::mpsc;
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    // 여기에 코드를 작성하세요!
    // 1. mpsc 채널을 생성하세요 (버퍼 크기 10)
    // 2. 3개의 생산자 태스크를 spawn하세요
    //    - 각 생산자는 "생산자N: 메시지M" 형태의 문자열을 3개씩 보냅니다
    //    - 메시지 사이에 100ms 대기
    // 3. 원본 tx를 drop하세요
    // 4. 소비자에서 모든 메시지를 받아 출력하세요
    // 5. 받은 메시지 총 개수를 출력하세요

    // 예상 출력 (순서는 다를 수 있음):
    // 받음: 생산자1: 메시지1
    // 받음: 생산자2: 메시지1
    // ...
    // 총 9개의 메시지를 받았습니다.
}
```

<details>
<summary><b>정답 보기</b></summary>

```rust
use tokio::sync::mpsc;
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    let (tx, mut rx) = mpsc::channel::<String>(10);

    // 3개의 생산자 태스크 생성
    for producer_id in 1..=3 {
        let tx = tx.clone();
        tokio::spawn(async move {
            for msg_id in 1..=3 {
                let message = format!("생산자{}: 메시지{}", producer_id, msg_id);
                tx.send(message).await.unwrap();
                sleep(Duration::from_millis(100)).await;
            }
        });
    }

    // 원본 tx를 drop해야 채널이 닫힘
    drop(tx);

    // 소비자: 모든 메시지 받기
    let mut count = 0;
    while let Some(message) = rx.recv().await {
        println!("받음: {}", message);
        count += 1;
    }

    println!("총 {}개의 메시지를 받았습니다.", count);
}
```

</details>

---

## 8. 확인 문제

### 문제 1
`tokio::spawn()`으로 만든 태스크와 `std::thread::spawn()`으로 만든 스레드의 가장 큰 차이점은?

### 문제 2
아래 코드에서 총 소요 시간은 약 몇 초인가요?
```rust
async fn work_a() { tokio::time::sleep(Duration::from_secs(2)).await; }
async fn work_b() { tokio::time::sleep(Duration::from_secs(3)).await; }

#[tokio::main]
async fn main() {
    tokio::join!(work_a(), work_b());
}
```

### 문제 3
`tokio::select!`와 `tokio::join!`의 차이점을 설명하세요.

### 문제 4
아래 코드의 출력 결과는?
```rust
use tokio::time::{timeout, sleep, Duration};

#[tokio::main]
async fn main() {
    let result = timeout(
        Duration::from_secs(1),
        sleep(Duration::from_secs(3)),
    ).await;

    match result {
        Ok(_) => println!("완료"),
        Err(_) => println!("시간 초과"),
    }
}
```

### 문제 5
`mpsc` 채널에서 원본 `tx`(송신자)를 `drop`하지 않으면 어떤 문제가 발생하나요?

---

## 확인 문제 정답

<details>
<summary>정답 보기 (먼저 풀어본 후 클릭하세요!)</summary>

### 문제 1 정답
Tokio 태스크는 **경량 태스크**로 메모리를 수 KB만 사용하고, 수만 개를 동시에 실행할 수 있습니다. 반면 OS 스레드는 약 8MB의 스택을 차지하고, 수천 개 이상 만들기 어렵습니다. 태스크는 `.await` 시점에 스레드를 양보하여 다른 태스크가 실행될 수 있지만, 스레드는 OS 스케줄러가 관리합니다.

### 문제 2 정답
**약 3초**

`tokio::join!`은 두 작업을 **동시에** 실행합니다. `work_a()`는 2초, `work_b()`는 3초가 걸리므로, 전체 소요 시간은 둘 중 긴 쪽인 3초입니다. 순차 실행이었다면 5초가 걸렸을 것입니다.

### 문제 3 정답
- `tokio::join!`: 여러 작업을 동시에 실행하고, **모든 작업이 완료**될 때까지 기다립니다.
- `tokio::select!`: 여러 작업을 동시에 실행하고, **가장 먼저 완료되는 하나만** 선택합니다. 나머지 작업은 취소됩니다.

### 문제 4 정답
**`시간 초과`**

`timeout`은 1초의 제한을 걸었지만, `sleep`은 3초를 기다립니다. 1초가 지나면 타임아웃이 발생하여 `Err`가 반환됩니다.

### 문제 5 정답
수신자(`rx`)의 `recv()`가 **영원히 기다립니다**. `mpsc` 채널에서 `rx.recv()`는 모든 송신자(`tx`)가 드롭되어야 `None`을 반환하고 루프가 종료됩니다. 원본 `tx`를 드롭하지 않으면, 아직 송신자가 살아있다고 판단하여 메시지가 올 때까지 무한 대기합니다.

</details>

---

## 9. 15장 정리

| 배운 것 | 핵심 |
|---------|------|
| tokio::spawn | 경량 비동기 태스크 생성 |
| JoinHandle | `handle.await`로 태스크 결과 받기 |
| tokio::join! | 여러 작업을 동시에 실행, 모두 완료 대기 |
| tokio::select! | 여러 작업 중 가장 빨리 끝나는 하나만 선택 |
| tokio::fs | 비동기 파일 읽기/쓰기 |
| tokio::time::sleep | 비동기 대기 (스레드를 블로킹하지 않음) |
| tokio::time::timeout | 비동기 작업에 시간 제한 걸기 |
| tokio::time::interval | 주기적으로 반복 실행 |
| mpsc 채널 | 다중 생산자 -> 단일 소비자 |
| oneshot 채널 | 일회용 메시지 전달 |
| broadcast 채널 | 하나의 메시지를 여러 구독자에게 전달 |
| Graceful Shutdown | broadcast + select!로 안전하게 종료 |

---

## 다음 장 예고

> **16장. HTTP 기초 이론**에서는 웹의 핵심인 HTTP 프로토콜을 배웁니다.
> 요청과 응답의 구조, GET/POST/PUT/DELETE 메서드, 상태 코드, JSON 등을 이해하고, 17장에서 Axum으로 실제 웹서버를 만들 준비를 합니다!

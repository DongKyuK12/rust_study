# 17장. Axum으로 첫 웹서버

> **목표**: Axum 프레임워크로 첫 번째 웹서버를 만들고 실행한다.

---

드디어 이 순간이 왔습니다. 지금까지 배운 Rust 문법, 비동기, Tokio를 모두 합쳐서 **진짜 웹서버**를 만듭니다. 브라우저에서 주소를 입력하면 응답이 오는, 진짜 서버입니다!

---

## 1. Axum이란?

**Axum**은 Rust로 웹서버를 만들기 위한 프레임워크입니다. Tokio 팀이 직접 만들었습니다.

### Rust 웹 프레임워크 비교

| 프레임워크 | 특징 |
|-----------|------|
| **Axum** | Tokio 팀 제작, 타입 안전, 최신 설계 |
| Actix-web | 가장 오래됨, 높은 성능 |
| Rocket | 사용하기 쉬움, 매크로 많음 |
| Warp | 필터 기반 설계 |

### 왜 Axum을 선택하나?

1. **Tokio 생태계**: 앞서 배운 Tokio와 완벽하게 호환됩니다
2. **타입 안전**: 잘못된 핸들러를 작성하면 컴파일 단계에서 잡아줍니다
3. **인기**: 2024~2025년 기준 가장 빠르게 성장하는 Rust 웹 프레임워크입니다
4. **매크로 최소화**: 마법 같은 매크로 없이, Rust 문법 그대로 씁니다

---

## 2. 프로젝트 설정

### 새 프로젝트 만들기

터미널에서 다음 명령어를 실행합니다:

```bash
cargo new my_web_server
cd my_web_server
```

### 의존성 추가

`Cargo.toml` 파일을 열고 `[dependencies]` 부분을 수정합니다:

```toml
[dependencies]
axum = "0.8"
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
```

각 크레이트의 역할:

| 크레이트 | 역할 |
|---------|------|
| `axum` | 웹 프레임워크 (라우팅, 핸들러 등) |
| `tokio` | 비동기 런타임 (서버가 동시에 여러 요청을 처리) |
| `serde` | 데이터 직렬화/역직렬화 (JSON 변환) |
| `serde_json` | JSON 처리 |

<details>
<summary><b>🔍 features = ["full"]이 뭔가? (원리)</b></summary>

### Cargo의 Feature 시스템

크레이트는 **모든 기능을 한꺼번에 제공하지 않습니다**. 필요한 기능만 골라서 쓸 수 있게 **feature 플래그**를 제공합니다.

```toml
# "full" = 모든 기능 다 켜기
tokio = { version = "1", features = ["full"] }

# 필요한 것만 골라 쓸 수도 있음
tokio = { version = "1", features = ["rt-multi-thread", "macros", "net"] }
```

`serde`의 `derive` feature를 켜면 `#[derive(Serialize, Deserialize)]` 매크로를 사용할 수 있습니다. feature를 안 켜면 이 매크로를 쓸 수 없습니다.

이런 설계의 장점:
- **컴파일 시간 단축**: 안 쓰는 기능은 컴파일하지 않음
- **바이너리 크기 감소**: 필요한 코드만 포함

학습 단계에서는 `"full"`로 켜두고, 나중에 최적화할 때 필요한 것만 골라 쓰면 됩니다.

</details>

---

## 3. Hello World 서버

드디어 첫 번째 웹서버를 만듭니다. `src/main.rs`를 다음과 같이 작성하세요:

```rust
use axum::{Router, routing::get};

#[tokio::main]
async fn main() {
    let app = Router::new()
        .route("/", get(hello));

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000")
        .await
        .unwrap();

    println!("서버가 http://localhost:3000 에서 실행 중입니다!");

    axum::serve(listener, app).await.unwrap();
}

async fn hello() -> &'static str {
    "Hello, World!"
}
```

### 실행하기

```bash
cargo run
```

첫 빌드는 의존성을 다운로드하므로 시간이 걸립니다. `서버가 http://localhost:3000 에서 실행 중입니다!`가 나타나면 성공입니다.

브라우저를 열고 `http://localhost:3000`에 접속하세요. **Hello, World!** 가 보이면 축하합니다! 첫 웹서버를 만들었습니다!

> 서버를 종료하려면 터미널에서 `Ctrl + C`를 누르세요.

### 코드 한 줄씩 이해하기

```rust
use axum::{Router, routing::get};
// Router: 어떤 URL에 어떤 함수를 연결할지 정하는 라우터
// get: HTTP GET 요청을 처리하겠다는 뜻
```

```rust
#[tokio::main]
async fn main() {
// 비동기 main 함수. Tokio 런타임 위에서 실행됨 (15장에서 배웠습니다)
```

```rust
    let app = Router::new()
        .route("/", get(hello));
// "/" 경로로 GET 요청이 오면 hello 함수를 실행하라
```

```rust
    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000")
        .await
        .unwrap();
// TCP 리스너를 3000번 포트에 바인딩
// "0.0.0.0"은 모든 네트워크 인터페이스에서 접속 허용
```

```rust
    axum::serve(listener, app).await.unwrap();
// 리스너와 라우터를 연결하여 서버 시작
// .await로 서버가 계속 실행되며 요청을 기다림
```

```rust
async fn hello() -> &'static str {
    "Hello, World!"
}
// 핸들러 함수: 요청이 오면 "Hello, World!" 문자열을 응답으로 반환
// async fn이어야 합니다
```

<details>
<summary><b>🔍 Router, Handler, 서버가 내부적으로 어떻게 동작하나? (원리)</b></summary>

### 요청 처리 흐름

브라우저에서 `http://localhost:3000`을 입력하면 이런 일이 일어납니다:

```
브라우저                         서버 (우리 코드)
  │                                │
  │─── GET / HTTP/1.1 ──────────>│  1. TCP 연결 수락
  │                                │  2. HTTP 요청 파싱
  │                                │  3. Router에서 "/" 경로 찾기
  │                                │  4. hello() 핸들러 실행
  │                                │  5. 반환값을 HTTP 응답으로 변환
  │<── HTTP/1.1 200 OK ──────────│  6. 응답 전송
  │    "Hello, World!"            │
```

### Handler의 조건

Axum에서 핸들러 함수가 되려면:
1. **async fn**이어야 합니다
2. **반환 타입**이 `IntoResponse` 트레이트를 구현해야 합니다

`&'static str`, `String`, `Json<T>`, `(StatusCode, String)` 등이 모두 `IntoResponse`를 구현하고 있어서 핸들러의 반환값으로 사용할 수 있습니다.

### Router의 역할

Router는 **URL 패턴과 핸들러 함수의 매핑 테이블**입니다:

```
Router 내부 (개념적)
┌─────────────┬──────────┬──────────┐
│  경로        │ HTTP 메서드 │ 핸들러    │
├─────────────┼──────────┼──────────┤
│  /          │  GET     │  hello() │
│  /about     │  GET     │  about() │
│  /users     │  POST    │  create()│
└─────────────┴──────────┴──────────┘
```

요청이 들어오면 Router가 경로와 메서드를 비교해서 맞는 핸들러를 찾아 실행합니다.

</details>

---

## 4. 라우팅 기초

하나의 경로만으로는 쓸모가 없습니다. 여러 경로를 등록해봅시다.

### 여러 경로 등록

```rust
use axum::{Router, routing::get};

#[tokio::main]
async fn main() {
    let app = Router::new()
        .route("/", get(home))
        .route("/hello", get(hello))
        .route("/about", get(about));

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000")
        .await
        .unwrap();

    println!("서버가 http://localhost:3000 에서 실행 중입니다!");

    axum::serve(listener, app).await.unwrap();
}

async fn home() -> &'static str {
    "홈 페이지입니다"
}

async fn hello() -> &'static str {
    "안녕하세요!"
}

async fn about() -> &'static str {
    "Axum으로 만든 첫 번째 웹서버입니다"
}
```

이제 세 가지 URL에 접속할 수 있습니다:
- `http://localhost:3000/` -> "홈 페이지입니다"
- `http://localhost:3000/hello` -> "안녕하세요!"
- `http://localhost:3000/about` -> "Axum으로 만든 첫 번째 웹서버입니다"

### HTTP 메서드별 라우팅

웹에서는 **같은 URL이라도 HTTP 메서드에 따라 다른 동작**을 합니다:

| HTTP 메서드 | 용도 | 예시 |
|------------|------|------|
| `GET` | 데이터 조회 | 사용자 정보 가져오기 |
| `POST` | 데이터 생성 | 새 사용자 등록 |
| `PUT` | 데이터 수정 | 사용자 정보 업데이트 |
| `DELETE` | 데이터 삭제 | 사용자 삭제 |

```rust
use axum::routing::{get, post, put, delete};

let app = Router::new()
    .route("/users", get(list_users))      // GET /users -> 목록 조회
    .route("/users", post(create_user))    // POST /users -> 새로 생성
    .route("/users/:id", put(update_user))    // PUT /users/1 -> 수정
    .route("/users/:id", delete(delete_user)); // DELETE /users/1 -> 삭제
```

> 브라우저 주소창에서 접속하면 항상 **GET** 요청입니다. POST, PUT, DELETE는 나중에 API 클라이언트(curl, Postman 등)로 테스트합니다.

---

## 5. 핸들러 함수

핸들러 함수는 **요청을 받아서 응답을 반환하는 함수**입니다. 다양한 타입을 반환할 수 있습니다.

### 문자열 반환

```rust
// &'static str 반환
async fn hello() -> &'static str {
    "Hello!"
}

// String 반환
async fn greeting() -> String {
    let name = "Rust";
    format!("안녕하세요, {}!", name)
}
```

### HTML 반환

```rust
use axum::response::Html;

async fn home() -> Html<&'static str> {
    Html("<h1>환영합니다!</h1><p>Axum 웹서버입니다.</p>")
}

async fn dynamic_page() -> Html<String> {
    let title = "내 첫 웹서버";
    Html(format!(
        "<html><body><h1>{}</h1><p>Rust로 만들었습니다!</p></body></html>",
        title
    ))
}
```

브라우저에서 접속하면 HTML이 렌더링되어 보입니다.

### 상태 코드 반환

HTTP 응답에는 **상태 코드**가 포함됩니다. 성공은 200, 에러는 404 같은 숫자입니다.

```rust
use axum::http::StatusCode;

// 상태 코드만 반환
async fn health_check() -> StatusCode {
    StatusCode::OK  // 200
}

// 상태 코드 + 메시지
async fn not_found() -> (StatusCode, &'static str) {
    (StatusCode::NOT_FOUND, "페이지를 찾을 수 없습니다")
}

// 조건에 따라 다른 상태 코드
async fn check_status() -> (StatusCode, String) {
    let is_healthy = true;

    if !is_healthy {
        return (StatusCode::SERVICE_UNAVAILABLE, "서비스 점검 중".to_string());
    }

    (StatusCode::OK, "정상 운영 중".to_string())
}
```

자주 쓰는 상태 코드:

| 코드 | 상수 | 의미 |
|------|------|------|
| 200 | `StatusCode::OK` | 성공 |
| 201 | `StatusCode::CREATED` | 생성 성공 |
| 400 | `StatusCode::BAD_REQUEST` | 잘못된 요청 |
| 404 | `StatusCode::NOT_FOUND` | 찾을 수 없음 |
| 500 | `StatusCode::INTERNAL_SERVER_ERROR` | 서버 에러 |

---

## 6. 요청 파라미터 추출

실제 API에서는 URL에 담긴 정보를 읽어야 합니다. Axum에서는 **추출자(Extractor)**를 사용합니다.

### Path 파라미터

URL의 일부분을 변수로 받습니다. `/users/42`에서 `42`를 추출하는 식입니다.

```rust
use axum::extract::Path;

// /users/:id 경로에서 id를 추출
async fn get_user(Path(id): Path<u32>) -> String {
    format!("User ID: {}", id)
}

// /hello/:name 경로에서 name을 추출
async fn hello_name(Path(name): Path<String>) -> String {
    format!("안녕하세요, {}님!", name)
}
```

라우터에 등록할 때는 `:파라미터명`으로 표시합니다:

```rust
let app = Router::new()
    .route("/users/:id", get(get_user))
    .route("/hello/:name", get(hello_name));
```

- `http://localhost:3000/users/42` -> "User ID: 42"
- `http://localhost:3000/hello/홍길동` -> "안녕하세요, 홍길동님!"

### 여러 Path 파라미터

```rust
// /users/:user_id/posts/:post_id
async fn get_post(Path((user_id, post_id)): Path<(u32, u32)>) -> String {
    format!("사용자 {}의 게시물 {}", user_id, post_id)
}
```

### Query 파라미터

URL의 `?key=value` 부분을 읽습니다. `/search?q=rust`에서 `q`값을 추출합니다.

```rust
use axum::extract::Query;
use serde::Deserialize;

#[derive(Deserialize)]
struct SearchParams {
    q: String,
}

async fn search(Query(params): Query<SearchParams>) -> String {
    format!("검색어: {}", params.q)
}
```

라우터 등록:

```rust
let app = Router::new()
    .route("/search", get(search));
```

- `http://localhost:3000/search?q=rust` -> "검색어: rust"
- `http://localhost:3000/search?q=웹서버` -> "검색어: 웹서버"

### 선택적 Query 파라미터

파라미터가 없을 수도 있다면 `Option`을 사용합니다:

```rust
#[derive(Deserialize)]
struct PaginationParams {
    page: Option<u32>,
    size: Option<u32>,
}

async fn list_items(Query(params): Query<PaginationParams>) -> String {
    let page = params.page.unwrap_or(1);
    let size = params.size.unwrap_or(10);
    format!("페이지: {}, 크기: {}", page, size)
}
```

- `http://localhost:3000/items` -> "페이지: 1, 크기: 10" (기본값)
- `http://localhost:3000/items?page=3&size=20` -> "페이지: 3, 크기: 20"

<details>
<summary><b>🔍 추출자(Extractor)는 어떻게 동작하나? (원리)</b></summary>

### Axum의 Extractor 패턴

Axum의 핵심 아이디어는 **핸들러 함수의 매개변수 타입을 보고 자동으로 데이터를 추출하는 것**입니다.

```rust
// 매개변수가 Path<u32>이면 -> URL에서 숫자를 추출
async fn get_user(Path(id): Path<u32>) -> String { ... }

// 매개변수가 Query<T>이면 -> 쿼리 문자열에서 추출
async fn search(Query(params): Query<SearchParams>) -> String { ... }

// 매개변수가 Json<T>이면 -> 요청 바디의 JSON에서 추출
async fn create(Json(data): Json<CreateUser>) -> String { ... }
```

이것이 가능한 이유는 `FromRequestParts`와 `FromRequest`라는 **트레이트** 때문입니다. `Path`, `Query`, `Json` 등은 모두 이 트레이트를 구현하고 있어서, Axum이 자동으로 적절한 추출 로직을 실행합니다.

```
HTTP 요청 도착
    │
    ├── URL 경로에서 Path 파라미터 추출
    ├── URL 쿼리에서 Query 파라미터 추출
    ├── 요청 바디에서 Json 데이터 추출
    │
    └── 핸들러 함수 호출 (추출된 데이터를 매개변수로 전달)
```

**타입만 맞게 선언하면 Axum이 알아서 처리합니다.** 이것이 Axum의 가장 큰 장점입니다.

</details>

---

## 7. JSON 응답

실제 API 서버에서는 HTML이 아니라 **JSON**으로 데이터를 주고받습니다.

### JSON 응답 보내기

```rust
use axum::Json;
use serde::Serialize;

#[derive(Serialize)]
struct User {
    id: u32,
    name: String,
    email: String,
}

async fn get_user() -> Json<User> {
    let user = User {
        id: 1,
        name: "홍길동".to_string(),
        email: "hong@example.com".to_string(),
    };
    Json(user)
}
```

`http://localhost:3000/user`에 접속하면 이런 JSON 응답이 옵니다:

```json
{
  "id": 1,
  "name": "홍길동",
  "email": "hong@example.com"
}
```

`#[derive(Serialize)]`를 붙이면 구조체가 자동으로 JSON으로 변환됩니다.

### 목록 반환

```rust
async fn list_users() -> Json<Vec<User>> {
    let users = vec![
        User { id: 1, name: "홍길동".to_string(), email: "hong@example.com".to_string() },
        User { id: 2, name: "김철수".to_string(), email: "kim@example.com".to_string() },
    ];
    Json(users)
}
```

<details>
<summary><b>🔍 serde의 직렬화/역직렬화가 뭔가? (원리)</b></summary>

### 직렬화 (Serialization)

**직렬화**는 Rust의 구조체를 JSON 같은 텍스트 형식으로 변환하는 것입니다.

```
Rust 구조체                          JSON 문자열
User {                    ──────>    {"id":1,"name":"홍길동"}
    id: 1,                serialize
    name: "홍길동",
}
```

### 역직렬화 (Deserialization)

**역직렬화**는 반대로, JSON 텍스트를 Rust 구조체로 변환하는 것입니다.

```
JSON 문자열                          Rust 구조체
{"id":1,"name":"홍길동"}   ──────>   User {
                          deserialize    id: 1,
                                        name: "홍길동",
                                    }
```

### serde가 하는 일

`serde`는 이 변환을 **자동으로** 해주는 크레이트입니다.

```rust
#[derive(Serialize)]    // 구조체 -> JSON (직렬화)
#[derive(Deserialize)]  // JSON -> 구조체 (역직렬화)
struct User {
    id: u32,
    name: String,
}
```

`#[derive(Serialize)]`를 붙이면 serde가 컴파일 시점에 변환 코드를 자동 생성합니다. 런타임 비용이 거의 없고, 수동으로 JSON 파싱 코드를 작성할 필요가 없습니다.

### serde_json의 역할

`serde`는 변환 "프레임워크"이고, `serde_json`은 **JSON 형식 전용 구현**입니다.

```
serde (프레임워크)
├── serde_json  : JSON 형식
├── serde_yaml  : YAML 형식
├── serde_toml  : TOML 형식
└── ...
```

Axum의 `Json<T>`는 내부적으로 `serde_json`을 사용합니다.

</details>

---

## 8. JSON 요청 받기

클라이언트가 보내는 JSON 데이터를 받아서 처리해봅시다.

### Json<T>로 요청 바디 파싱

```rust
use axum::Json;
use serde::{Deserialize, Serialize};

#[derive(Deserialize)]
struct CreateUser {
    name: String,
    email: String,
}

#[derive(Serialize)]
struct CreateUserResponse {
    id: u32,
    name: String,
    message: String,
}

async fn create_user(Json(payload): Json<CreateUser>) -> Json<CreateUserResponse> {
    let response = CreateUserResponse {
        id: 42, // 실제로는 DB에서 생성된 ID
        name: payload.name.clone(),
        message: format!("{}님이 등록되었습니다!", payload.name),
    };
    Json(response)
}
```

라우터 등록:

```rust
use axum::routing::post;

let app = Router::new()
    .route("/users", post(create_user));
```

이 API를 테스트하려면 터미널에서 `curl`을 사용합니다:

```bash
curl -X POST http://localhost:3000/users \
  -H "Content-Type: application/json" \
  -d '{"name": "홍길동", "email": "hong@example.com"}'
```

응답:

```json
{
  "id": 42,
  "name": "홍길동",
  "message": "홍길동님이 등록되었습니다!"
}
```

### 입력 검증과 에러 처리

받은 데이터를 그대로 쓰면 안 됩니다. 검증은 필수입니다!

```rust
use axum::http::StatusCode;

async fn create_user(
    Json(payload): Json<CreateUser>,
) -> Result<Json<CreateUserResponse>, (StatusCode, String)> {
    if payload.name.is_empty() {
        return Err((StatusCode::BAD_REQUEST, "이름을 입력하세요".to_string()));
    }
    if !payload.email.contains('@') {
        return Err((StatusCode::BAD_REQUEST, "올바른 이메일 형식이 아닙니다".to_string()));
    }

    let response = CreateUserResponse {
        id: 42,
        name: payload.name.clone(),
        message: format!("{}님이 등록되었습니다!", payload.name),
    };
    Ok(Json(response))
}
```

7장에서 배운 **얼리 리턴 패턴**이 여기서도 쓰입니다! 검증 실패하면 즉시 에러를 반환하고, 성공 로직은 마지막에 둡니다.

---

## 9. 전체 예제: 미니 API 서버

지금까지 배운 것을 모두 합쳐서 하나의 완성된 서버를 만들어봅시다.

```rust
use axum::{
    extract::{Path, Query},
    http::StatusCode,
    routing::{get, post},
    Json, Router,
};
use serde::{Deserialize, Serialize};

// --- 데이터 구조체 ---

#[derive(Serialize, Clone)]
struct User {
    id: u32,
    name: String,
    email: String,
}

#[derive(Deserialize)]
struct CreateUser {
    name: String,
    email: String,
}

#[derive(Deserialize)]
struct SearchParams {
    q: Option<String>,
}

// --- 핸들러 함수 ---

async fn home() -> &'static str {
    "미니 API 서버에 오신 것을 환영합니다!"
}

async fn health() -> Json<serde_json::Value> {
    Json(serde_json::json!({
        "status": "ok",
        "version": "0.1.0"
    }))
}

async fn get_user(Path(id): Path<u32>) -> Result<Json<User>, (StatusCode, String)> {
    // 실제로는 DB에서 조회하지만, 여기서는 간단히 만듭니다
    if id == 0 {
        return Err((StatusCode::BAD_REQUEST, "ID는 0일 수 없습니다".to_string()));
    }
    if id > 100 {
        return Err((StatusCode::NOT_FOUND, format!("ID {}인 사용자를 찾을 수 없습니다", id)));
    }

    let user = User {
        id,
        name: format!("사용자{}", id),
        email: format!("user{}@example.com", id),
    };
    Ok(Json(user))
}

async fn list_users(Query(params): Query<SearchParams>) -> Json<Vec<User>> {
    let users = vec![
        User { id: 1, name: "홍길동".to_string(), email: "hong@example.com".to_string() },
        User { id: 2, name: "김철수".to_string(), email: "kim@example.com".to_string() },
        User { id: 3, name: "이영희".to_string(), email: "lee@example.com".to_string() },
    ];

    // 검색어가 있으면 필터링
    if let Some(query) = params.q {
        let filtered: Vec<User> = users
            .into_iter()
            .filter(|u| u.name.contains(&query))
            .collect();
        return Json(filtered);
    }

    Json(users)
}

async fn create_user(
    Json(payload): Json<CreateUser>,
) -> Result<(StatusCode, Json<User>), (StatusCode, String)> {
    if payload.name.is_empty() {
        return Err((StatusCode::BAD_REQUEST, "이름을 입력하세요".to_string()));
    }
    if !payload.email.contains('@') {
        return Err((StatusCode::BAD_REQUEST, "올바른 이메일 형식이 아닙니다".to_string()));
    }

    let user = User {
        id: 42, // 실제로는 DB에서 자동 생성
        name: payload.name,
        email: payload.email,
    };
    Ok((StatusCode::CREATED, Json(user)))
}

// --- 메인 함수 ---

#[tokio::main]
async fn main() {
    let app = Router::new()
        .route("/", get(home))
        .route("/health", get(health))
        .route("/users", get(list_users).post(create_user))
        .route("/users/:id", get(get_user));

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000")
        .await
        .unwrap();

    println!("서버가 http://localhost:3000 에서 실행 중입니다!");
    println!("사용 가능한 API:");
    println!("  GET  /          - 홈");
    println!("  GET  /health    - 상태 확인");
    println!("  GET  /users     - 사용자 목록 (검색: ?q=이름)");
    println!("  GET  /users/:id - 사용자 조회");
    println!("  POST /users     - 사용자 생성");

    axum::serve(listener, app).await.unwrap();
}
```

이 서버를 실행하고 다음 URL들을 테스트해보세요:

- `http://localhost:3000/` -> 환영 메시지
- `http://localhost:3000/health` -> 상태 JSON
- `http://localhost:3000/users` -> 전체 사용자 목록
- `http://localhost:3000/users?q=홍` -> "홍"이 포함된 사용자
- `http://localhost:3000/users/1` -> ID 1번 사용자
- `http://localhost:3000/users/999` -> 404 에러

---

## 10. 실습

### 실습 1: Hello World 서버 만들고 브라우저에서 확인

가장 기본적인 웹서버를 만드세요.

**요구사항:**
- `GET /` -> "Hello, World!" 반환
- `GET /ping` -> "pong" 반환
- 서버 시작 시 포트 번호 출력
- `cargo run`으로 실행 후 브라우저에서 두 경로 모두 확인

<details>
<summary>정답 보기</summary>

```rust
use axum::{Router, routing::get};

#[tokio::main]
async fn main() {
    let app = Router::new()
        .route("/", get(hello))
        .route("/ping", get(ping));

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000")
        .await
        .unwrap();

    println!("서버가 http://localhost:3000 에서 실행 중입니다!");

    axum::serve(listener, app).await.unwrap();
}

async fn hello() -> &'static str {
    "Hello, World!"
}

async fn ping() -> &'static str {
    "pong"
}
```

**확인 방법:**
1. `cargo run` 실행
2. 브라우저에서 `http://localhost:3000` 접속 -> "Hello, World!" 확인
3. 브라우저에서 `http://localhost:3000/ping` 접속 -> "pong" 확인

</details>

### 실습 2: 여러 라우트와 Path 파라미터

다양한 경로를 가진 서버를 만드세요.

**요구사항:**
- `GET /` -> HTML로 환영 메시지 (`<h1>환영합니다!</h1>`)
- `GET /hello/:name` -> "안녕하세요, {name}님!" 반환
- `GET /health` -> JSON으로 `{"status": "ok"}` 반환
- `GET /add/:a/:b` -> 두 숫자의 합을 반환 (예: `/add/3/5` -> "3 + 5 = 8")

<details>
<summary>정답 보기</summary>

```rust
use axum::{
    extract::Path,
    response::Html,
    routing::get,
    Json, Router,
};

#[tokio::main]
async fn main() {
    let app = Router::new()
        .route("/", get(home))
        .route("/hello/:name", get(hello))
        .route("/health", get(health))
        .route("/add/:a/:b", get(add));

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000")
        .await
        .unwrap();

    println!("서버가 http://localhost:3000 에서 실행 중입니다!");

    axum::serve(listener, app).await.unwrap();
}

async fn home() -> Html<&'static str> {
    Html("<h1>환영합니다!</h1>")
}

async fn hello(Path(name): Path<String>) -> String {
    format!("안녕하세요, {}님!", name)
}

async fn health() -> Json<serde_json::Value> {
    Json(serde_json::json!({ "status": "ok" }))
}

async fn add(Path((a, b)): Path<(i64, i64)>) -> String {
    format!("{} + {} = {}", a, b, a + b)
}
```

**확인 방법:**
1. `http://localhost:3000/` -> HTML 제목 확인
2. `http://localhost:3000/hello/홍길동` -> "안녕하세요, 홍길동님!"
3. `http://localhost:3000/health` -> JSON 응답
4. `http://localhost:3000/add/3/5` -> "3 + 5 = 8"

</details>

### 실습 3: JSON API 만들기 - 사용자 목록 조회와 생성

GET으로 사용자 목록을 조회하고, POST로 사용자를 추가하는 API를 만드세요.

**요구사항:**
- `GET /users` -> 하드코딩된 사용자 목록을 JSON으로 반환
- `POST /users` -> JSON 요청 바디를 받아 새 사용자 생성 응답 반환
  - 이름이 비어있으면 400 에러
  - 이메일에 `@`가 없으면 400 에러
  - 성공하면 201 상태 코드와 생성된 사용자 JSON 반환
- `GET /users/:id` -> 특정 사용자 조회 (ID가 1~3이면 성공, 아니면 404)

<details>
<summary>정답 보기</summary>

```rust
use axum::{
    extract::Path,
    http::StatusCode,
    routing::get,
    Json, Router,
};
use serde::{Deserialize, Serialize};

#[derive(Serialize, Clone)]
struct User {
    id: u32,
    name: String,
    email: String,
}

#[derive(Deserialize)]
struct CreateUser {
    name: String,
    email: String,
}

fn get_sample_users() -> Vec<User> {
    vec![
        User { id: 1, name: "홍길동".to_string(), email: "hong@example.com".to_string() },
        User { id: 2, name: "김철수".to_string(), email: "kim@example.com".to_string() },
        User { id: 3, name: "이영희".to_string(), email: "lee@example.com".to_string() },
    ]
}

async fn list_users() -> Json<Vec<User>> {
    Json(get_sample_users())
}

async fn get_user(
    Path(id): Path<u32>,
) -> Result<Json<User>, (StatusCode, String)> {
    let users = get_sample_users();

    let user = users.into_iter().find(|u| u.id == id);
    if user.is_none() {
        return Err((
            StatusCode::NOT_FOUND,
            format!("ID {}인 사용자를 찾을 수 없습니다", id),
        ));
    }

    Ok(Json(user.unwrap()))
}

async fn create_user(
    Json(payload): Json<CreateUser>,
) -> Result<(StatusCode, Json<User>), (StatusCode, String)> {
    if payload.name.is_empty() {
        return Err((StatusCode::BAD_REQUEST, "이름을 입력하세요".to_string()));
    }
    if !payload.email.contains('@') {
        return Err((
            StatusCode::BAD_REQUEST,
            "올바른 이메일 형식이 아닙니다".to_string(),
        ));
    }

    let new_user = User {
        id: 4, // 실제로는 DB에서 자동 생성
        name: payload.name,
        email: payload.email,
    };

    Ok((StatusCode::CREATED, Json(new_user)))
}

#[tokio::main]
async fn main() {
    let app = Router::new()
        .route("/users", get(list_users).post(create_user))
        .route("/users/:id", get(get_user));

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000")
        .await
        .unwrap();

    println!("서버가 http://localhost:3000 에서 실행 중입니다!");

    axum::serve(listener, app).await.unwrap();
}
```

**확인 방법:**

1. 사용자 목록 조회:
```bash
curl http://localhost:3000/users
```

2. 특정 사용자 조회:
```bash
curl http://localhost:3000/users/1
curl http://localhost:3000/users/99
```

3. 사용자 생성 (성공):
```bash
curl -X POST http://localhost:3000/users \
  -H "Content-Type: application/json" \
  -d '{"name":"박지민","email":"park@example.com"}'
```

4. 사용자 생성 (실패 - 빈 이름):
```bash
curl -X POST http://localhost:3000/users \
  -H "Content-Type: application/json" \
  -d '{"name":"","email":"test@example.com"}'
```

</details>

---

## 11. 확인 문제

### 문제 1

Axum에서 핸들러 함수가 되기 위한 두 가지 조건은?

### 문제 2

아래 코드의 빈칸을 채우세요:

```rust
use axum::{Router, routing::get};

let app = Router::new()
    .route("/hello", _____(say_hello));

async fn say_hello() -> ____________ {
    "안녕하세요!"
}
```

### 문제 3

`/users/42`에서 `42`를 추출하려면 핸들러 함수의 매개변수를 어떻게 작성해야 하나요?

- (a) `fn get_user(id: u32)`
- (b) `async fn get_user(Path(id): Path<u32>)`
- (c) `async fn get_user(Query(id): Query<u32>)`
- (d) `async fn get_user(Json(id): Json<u32>)`

### 문제 4

JSON 응답을 반환하기 위해 구조체에 붙여야 하는 derive 매크로는?

- (a) `#[derive(Debug)]`
- (b) `#[derive(Clone)]`
- (c) `#[derive(Serialize)]`
- (d) `#[derive(Deserialize)]`

### 문제 5

아래 핸들러 함수에서 에러 처리를 얼리 리턴 패턴으로 완성하세요:

```rust
async fn create_user(
    Json(payload): Json<CreateUser>,
) -> Result<Json<User>, (StatusCode, String)> {
    // payload.name이 비어있으면 400 에러 반환
    _______________________________________

    // payload.email에 '@'가 없으면 400 에러 반환
    _______________________________________

    // 성공
    Ok(Json(User {
        id: 1,
        name: payload.name,
        email: payload.email,
    }))
}
```

---

## 확인 문제 정답

<details>
<summary>정답 보기 (먼저 풀어본 후 클릭하세요!)</summary>

### 문제 1 정답

1. **`async fn`이어야 합니다** (비동기 함수)
2. **반환 타입이 `IntoResponse` 트레이트를 구현해야 합니다** (`&'static str`, `String`, `Json<T>`, `Html<T>`, `(StatusCode, String)` 등)

### 문제 2 정답

```rust
use axum::{Router, routing::get};

let app = Router::new()
    .route("/hello", get(say_hello));

async fn say_hello() -> &'static str {
    "안녕하세요!"
}
```

`get(say_hello)`로 GET 요청에 핸들러를 연결하고, 반환 타입은 `&'static str`입니다.

### 문제 3 정답: **(b) `async fn get_user(Path(id): Path<u32>)`**

URL 경로에서 값을 추출하려면 `Path` 추출자를 사용합니다. `Query`는 `?key=value` 형태의 쿼리 파라미터용이고, `Json`은 요청 바디용입니다.

### 문제 4 정답: **(c) `#[derive(Serialize)]`**

- `Serialize`: 구조체 -> JSON (응답 보낼 때)
- `Deserialize`: JSON -> 구조체 (요청 받을 때)

JSON **응답**을 반환하려면 `Serialize`가 필요합니다.

### 문제 5 정답

```rust
async fn create_user(
    Json(payload): Json<CreateUser>,
) -> Result<Json<User>, (StatusCode, String)> {
    if payload.name.is_empty() {
        return Err((StatusCode::BAD_REQUEST, "이름을 입력하세요".to_string()));
    }

    if !payload.email.contains('@') {
        return Err((StatusCode::BAD_REQUEST, "올바른 이메일 형식이 아닙니다".to_string()));
    }

    Ok(Json(User {
        id: 1,
        name: payload.name,
        email: payload.email,
    }))
}
```

검증 실패 시 `return Err(...)`로 즉시 반환하고, 모든 검증을 통과한 후에야 성공 응답을 반환합니다.

</details>

---

## 12. 17장 정리

| 배운 것 | 핵심 |
|---------|------|
| Axum | Tokio 기반 웹 프레임워크, 타입 안전 |
| 프로젝트 설정 | `Cargo.toml`에 axum, tokio, serde 추가 |
| Hello World 서버 | `Router::new().route("/", get(handler))` |
| 라우팅 | `.route()` 체이닝으로 여러 경로 등록 |
| HTTP 메서드 | `get()`, `post()`, `put()`, `delete()` |
| 핸들러 함수 | `async fn` + `IntoResponse` 반환 |
| 문자열/HTML 응답 | `&str`, `String`, `Html<T>` |
| 상태 코드 | `StatusCode::OK`, `(StatusCode, String)` |
| Path 파라미터 | `Path(id): Path<u32>` - URL에서 추출 |
| Query 파라미터 | `Query(params): Query<T>` - 쿼리에서 추출 |
| JSON 응답 | `Json<T>` + `#[derive(Serialize)]` |
| JSON 요청 | `Json<T>` + `#[derive(Deserialize)]` |
| 에러 처리 | `Result<성공, (StatusCode, String)>` + 얼리 리턴 |

---

## 다음 장 예고

> **18장. 라우팅과 미들웨어**에서는 더 복잡한 라우팅 구조를 배웁니다.
> 중첩 라우터로 API를 모듈별로 정리하고, 미들웨어로 요청 로깅, CORS 설정,
> 에러 처리를 일괄 적용하는 방법을 알아봅니다!

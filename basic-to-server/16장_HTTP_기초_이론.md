# 16장. HTTP 기초 이론

> **목표**: 웹의 핵심 프로토콜인 HTTP를 이해한다. 서버를 만들기 전에 알아야 할 기본 지식!

---

## 1. HTTP란?

**HTTP (HyperText Transfer Protocol)** 는 웹에서 데이터를 주고받는 **통신 규칙**입니다.

브라우저에서 주소를 입력하면 어떤 일이 벌어질까요?

```
┌──────────┐         요청 (Request)         ┌──────────┐
│          │  ──────────────────────────>   │          │
│ 클라이언트 │     "GET /index.html"         │   서버    │
│ (브라우저) │                               │ (백엔드)  │
│          │  <──────────────────────────   │          │
└──────────┘         응답 (Response)         └──────────┘
                  "200 OK + HTML 내용"
```

핵심 구조는 단순합니다: **요청(Request)** 을 보내면 **응답(Response)** 이 돌아옵니다.

### 식당 비유로 이해하기

HTTP 통신은 식당 주문 시스템과 똑같습니다:

```
┌─────────────────────────────────────────────────────────┐
│                    식당 = 웹 시스템                       │
│                                                         │
│  손님 (클라이언트)         점원           주방 (서버)       │
│  ┌─────┐               ┌─────┐         ┌─────┐        │
│  │     │ ── 주문서 ──> │     │ ──────> │     │        │
│  │     │   (요청)      │     │         │     │        │
│  │     │               │     │         │     │        │
│  │     │ <── 음식 ──── │     │ <────── │     │        │
│  │     │   (응답)      │     │         │     │        │
│  └─────┘               └─────┘         └─────┘        │
│                                                         │
│  메뉴판 = API 문서                                       │
│  주문서 = HTTP 요청                                      │
│  음식   = HTTP 응답                                      │
└─────────────────────────────────────────────────────────┘
```

| 식당 | 웹 | 설명 |
|------|-----|------|
| 손님 | 클라이언트 (브라우저, 앱) | 요청을 보내는 쪽 |
| 주방 | 서버 (백엔드) | 요청을 처리하고 응답하는 쪽 |
| 메뉴판 | API 문서 | 어떤 요청을 할 수 있는지 안내 |
| 주문서 | HTTP 요청 | 무엇을 원하는지 적어서 보냄 |
| 음식 | HTTP 응답 | 요청에 대한 결과물 |

<details>
<summary><b>HTTP의 특징 (원리)</b></summary>

### 비연결성 (Connectionless)

HTTP는 요청-응답이 끝나면 연결을 끊습니다. 식당에서 음식을 받으면 점원이 다른 손님에게 가는 것과 같습니다.

```
손님A: 주문 → 음식 받음 → 연결 끝
손님B: 주문 → 음식 받음 → 연결 끝
손님C: 주문 → 음식 받음 → 연결 끝
```

### 무상태성 (Stateless)

서버는 이전 요청을 기억하지 않습니다. 매번 새로운 손님처럼 대합니다.

```
1번째 요청: "저 아까 왔던 사람인데요"
서버: "누구세요? 주문서를 다시 보여주세요"

2번째 요청: "로그인한 사용자인데 내 정보 보여줘"
서버: "증명서(토큰)를 보여주세요"
```

그래서 로그인 상태를 유지하려면 **토큰(Token)** 이나 **쿠키(Cookie)** 같은 추가 장치가 필요합니다. 이 부분은 22장(인증과 인가)에서 자세히 다룹니다.

### HTTP 버전

| 버전 | 특징 |
|------|------|
| HTTP/1.0 | 매 요청마다 연결을 새로 맺음 |
| HTTP/1.1 | 연결을 재사용 (Keep-Alive). 현재 가장 많이 사용 |
| HTTP/2 | 하나의 연결로 여러 요청을 동시에 처리 |
| HTTP/3 | UDP 기반으로 더 빠른 연결 |

이 학습지에서는 **HTTP/1.1** 기준으로 설명합니다. 대부분의 웹 프레임워크가 이를 기본으로 지원합니다.

</details>

---

## 2. HTTP 요청 구조

클라이언트가 서버에 보내는 **요청(Request)** 은 크게 4가지 요소로 구성됩니다.

```
┌─────────────────────────────────────────────────┐
│              HTTP 요청 (Request)                  │
├─────────────────────────────────────────────────┤
│ 1. 요청 라인:  GET /users/1 HTTP/1.1             │  ← 메서드 + 경로 + 버전
├─────────────────────────────────────────────────┤
│ 2. 헤더:                                         │
│    Host: example.com                             │  ← 어디로 보내는지
│    Content-Type: application/json                │  ← 데이터 형식
│    Authorization: Bearer abc123                  │  ← 인증 정보
├─────────────────────────────────────────────────┤
│ 3. 빈 줄                                         │  ← 헤더와 바디 구분
├─────────────────────────────────────────────────┤
│ 4. 바디 (Body):                                  │
│    {"name": "홍길동", "age": 20}                  │  ← 전송할 데이터
└─────────────────────────────────────────────────┘
```

### HTTP 메서드

메서드는 **"무엇을 하고 싶은지"** 를 나타냅니다.

```
  GET       "정보 좀 보여주세요"      →  식당: 메뉴 조회
  POST      "새로운 걸 만들어주세요"   →  식당: 주문하기
  PUT       "기존 걸 바꿔주세요"      →  식당: 주문 변경
  DELETE    "지워주세요"             →  식당: 주문 취소
```

가장 많이 쓰는 4가지 메서드를 자세히 봅시다:

#### GET - 조회

데이터를 **가져올 때** 사용합니다. 바디가 없습니다.

```
GET /users/1 HTTP/1.1
Host: example.com
```

식당: "1번 테이블 주문 내역 보여주세요"

#### POST - 생성

새로운 데이터를 **만들 때** 사용합니다. 바디에 데이터를 담습니다.

```
POST /users HTTP/1.1
Host: example.com
Content-Type: application/json

{"name": "홍길동", "age": 20}
```

식당: "비빔밥 1개 주문할게요"

#### PUT - 수정

기존 데이터를 **바꿀 때** 사용합니다. 바디에 수정된 데이터를 담습니다.

```
PUT /users/1 HTTP/1.1
Host: example.com
Content-Type: application/json

{"name": "홍길동", "age": 21}
```

식당: "아까 주문한 비빔밥을 김치찌개로 변경할게요"

#### DELETE - 삭제

데이터를 **지울 때** 사용합니다. 보통 바디가 없습니다.

```
DELETE /users/1 HTTP/1.1
Host: example.com
```

식당: "아까 주문 취소해주세요"

### 메서드 비교표

| 메서드 | 용도 | 바디 유무 | 식당 비유 |
|--------|------|----------|----------|
| GET | 조회 | 없음 | 메뉴판 보기 |
| POST | 생성 | 있음 | 주문하기 |
| PUT | 수정 | 있음 | 주문 변경 |
| DELETE | 삭제 | 보통 없음 | 주문 취소 |

### URL 경로

URL은 **어떤 데이터에 대한 요청인지** 를 나타냅니다.

```
https://example.com/users/1?sort=name&page=2
└─┬──┘  └────┬─────┘└──┬──┘ └──────┬────────┘
 프로토콜    호스트     경로     쿼리 파라미터
```

| 부분 | 예시 | 설명 |
|------|------|------|
| 프로토콜 | `https://` | 통신 방식 (http 또는 https) |
| 호스트 | `example.com` | 서버 주소 |
| 경로 | `/users/1` | 원하는 리소스의 위치 |
| 쿼리 파라미터 | `?sort=name&page=2` | 추가 조건 (옵션) |

<details>
<summary><b>PATCH vs PUT, 그리고 다른 메서드들 (원리)</b></summary>

### PUT vs PATCH

PUT과 PATCH 모두 데이터를 수정하지만 차이가 있습니다:

```
기존 데이터: {"name": "홍길동", "age": 20, "email": "hong@mail.com"}

PUT /users/1   → 전체 교체
{"name": "홍길동", "age": 21, "email": "hong@mail.com"}
(모든 필드를 다 보내야 함)

PATCH /users/1 → 일부만 수정
{"age": 21}
(바꿀 필드만 보내면 됨)
```

### 그 외 메서드들

| 메서드 | 용도 | 자주 쓰나? |
|--------|------|-----------|
| HEAD | GET과 같지만 바디 없이 헤더만 반환 | 가끔 |
| OPTIONS | 서버가 지원하는 메서드 확인 (CORS에서 자동 사용) | 자동 |
| PATCH | 데이터 일부만 수정 | 자주 |

실무에서는 GET, POST, PUT, DELETE, PATCH 이 5개를 가장 많이 씁니다.

### 멱등성(Idempotency)

같은 요청을 여러 번 보내도 결과가 같으면 **멱등하다**고 합니다:

| 메서드 | 멱등? | 이유 |
|--------|------|------|
| GET | O | 몇 번 조회해도 데이터가 안 변함 |
| PUT | O | 같은 데이터로 수정하면 결과가 같음 |
| DELETE | O | 이미 삭제된 걸 다시 삭제해도 없는 건 마찬가지 |
| POST | X | 주문을 2번 보내면 주문이 2개 생김! |

이 개념은 API를 설계할 때 중요합니다. 예를 들어 결제 요청이 네트워크 오류로 중복 전송되면? POST라면 결제가 2번 될 수 있으므로, 멱등성 보장을 위한 별도 처리가 필요합니다.

</details>

---

## 3. HTTP 응답 구조

서버가 클라이언트에 보내는 **응답(Response)** 도 요청과 비슷한 구조입니다.

```
┌─────────────────────────────────────────────────┐
│              HTTP 응답 (Response)                 │
├─────────────────────────────────────────────────┤
│ 1. 상태 라인:  HTTP/1.1 200 OK                    │  ← 버전 + 상태코드 + 메시지
├─────────────────────────────────────────────────┤
│ 2. 헤더:                                         │
│    Content-Type: application/json                │  ← 응답 데이터 형식
│    Content-Length: 45                             │  ← 바디 크기
├─────────────────────────────────────────────────┤
│ 3. 빈 줄                                         │  ← 헤더와 바디 구분
├─────────────────────────────────────────────────┤
│ 4. 바디 (Body):                                  │
│    {"id": 1, "name": "홍길동", "age": 20}         │  ← 실제 데이터
└─────────────────────────────────────────────────┘
```

### 실제 통신 예시

```
[요청]                              [응답]

GET /users/1 HTTP/1.1               HTTP/1.1 200 OK
Host: example.com          →       Content-Type: application/json
Accept: application/json
                                    {"id": 1, "name": "홍길동"}
```

```
[요청]                              [응답]

POST /users HTTP/1.1                HTTP/1.1 201 Created
Host: example.com          →       Content-Type: application/json
Content-Type: application/json
                                    {"id": 2, "name": "김영희"}
{"name": "김영희", "age": 25}
```

```
[요청]                              [응답]

GET /users/999 HTTP/1.1             HTTP/1.1 404 Not Found
Host: example.com          →       Content-Type: application/json

                                    {"error": "사용자를 찾을 수 없습니다"}
```

---

## 4. 상태 코드

상태 코드는 **요청이 어떻게 처리되었는지** 를 숫자 3자리로 알려줍니다.

### 한 눈에 보기

```
1xx  정보         "잠깐만요, 처리 중입니다"       (거의 안 씀)
2xx  성공         "네, 잘 됐습니다!"             (제일 좋은 응답)
3xx  리다이렉트    "다른 곳으로 가세요"            (주소 이동)
4xx  클라이언트 에러 "요청이 잘못됐어요"            (당신 잘못)
5xx  서버 에러     "서버에 문제가 생겼어요"         (우리 잘못)
```

### 자주 쓰는 상태 코드

식당 비유와 함께 봅시다:

#### 2xx - 성공

```
200 OK          → "주문하신 음식 나왔습니다!"
                   가장 일반적인 성공 응답

201 Created     → "새 메뉴가 등록되었습니다!"
                   새로운 리소스가 생성됨 (POST 성공 시)

204 No Content  → "주문이 취소되었습니다" (영수증 없음)
                   성공했지만 돌려줄 데이터 없음 (DELETE 성공 시)
```

#### 4xx - 클라이언트 에러

```
400 Bad Request    → "주문서를 알아볼 수가 없어요"
                      요청 형식이 잘못됨 (필수 필드 누락 등)

401 Unauthorized   → "회원 카드를 보여주세요"
                      인증이 필요함 (로그인 안 됨)

403 Forbidden      → "VIP 전용 메뉴입니다"
                      권한이 없음 (로그인은 했지만 접근 불가)

404 Not Found      → "그런 메뉴는 없습니다"
                      요청한 리소스가 존재하지 않음

409 Conflict       → "이미 같은 주문이 있습니다"
                      리소스 충돌 (중복된 데이터)

422 Unprocessable  → "메뉴는 있는데 재료가 부족합니다"
                      요청 형식은 맞지만 내용이 처리 불가
```

#### 5xx - 서버 에러

```
500 Internal Server Error → "주방에 불이 났습니다!"
                             서버 내부 오류 (가장 일반적인 서버 에러)

502 Bad Gateway           → "식자재 업체에서 응답이 없습니다"
                             중간 서버(프록시)가 잘못된 응답을 받음

503 Service Unavailable   → "오늘 가게 쉽니다"
                             서버가 일시적으로 요청을 처리할 수 없음
```

### 상태 코드 정리표

| 코드 | 이름 | 의미 | 주로 쓰이는 상황 |
|------|------|------|-----------------|
| 200 | OK | 성공 | GET, PUT 성공 |
| 201 | Created | 생성됨 | POST 성공 |
| 204 | No Content | 내용 없음 | DELETE 성공 |
| 400 | Bad Request | 잘못된 요청 | 유효성 검사 실패 |
| 401 | Unauthorized | 인증 필요 | 로그인 안 됨 |
| 403 | Forbidden | 접근 금지 | 권한 없음 |
| 404 | Not Found | 없음 | 리소스 없음 |
| 500 | Internal Server Error | 서버 오류 | 서버 버그 |

<details>
<summary><b>401 vs 403, 그리고 상태 코드 선택 기준 (원리)</b></summary>

### 401 Unauthorized vs 403 Forbidden

이 두 코드는 자주 혼동됩니다:

```
401 Unauthorized (인증 실패)
├── "당신이 누구인지 모르겠어요"
├── 로그인을 안 했거나 토큰이 만료됨
└── 해결: 로그인하세요 (인증 정보를 보내세요)

403 Forbidden (인가 실패)
├── "당신이 누구인지는 알겠는데, 권한이 없어요"
├── 로그인은 했지만 관리자 전용 기능에 접근
└── 해결: 관리자에게 권한을 요청하세요
```

비유:
```
401 = 출입증 없이 건물에 들어가려는 것
403 = 출입증은 있지만 "임원 전용 층"에 가려는 것
```

### 상태 코드 선택 가이드

서버를 만들 때 어떤 상태 코드를 반환할지 고민되면:

```
요청 성공했나?
├── 예 → 데이터를 반환하나?
│        ├── 예 → 새로 만든 건가?
│        │        ├── 예 → 201 Created
│        │        └── 아니오 → 200 OK
│        └── 아니오 → 204 No Content
│
└── 아니오 → 누구 잘못인가?
             ├── 클라이언트 → 어떤 문제?
             │               ├── 형식 오류 → 400 Bad Request
             │               ├── 인증 필요 → 401 Unauthorized
             │               ├── 권한 부족 → 403 Forbidden
             │               └── 리소스 없음 → 404 Not Found
             │
             └── 서버 → 500 Internal Server Error
```

</details>

---

## 5. 헤더 (Header)

헤더는 요청이나 응답에 대한 **추가 정보**를 담습니다. "본문 말고도 알려줄 게 있어요"라는 뜻입니다.

```
┌─────────────────────────────────────────┐
│  헤더 = 편지 봉투의 겉면                   │
│                                         │
│  보내는 사람: 홍길동        (Host)         │
│  받는 형식: 한국어          (Accept)       │
│  내용 형식: 편지            (Content-Type) │
│  인증 도장: [직인]          (Authorization)│
│                                         │
│  ─────────────────────────              │
│  [편지 내용 = 바디(Body)]                 │
└─────────────────────────────────────────┘
```

### 자주 쓰는 헤더

#### Content-Type

**데이터의 형식**을 알려줍니다. "이 데이터가 뭔지"를 서버와 클라이언트가 알 수 있습니다.

```
Content-Type: application/json        ← JSON 형식 (API에서 가장 많이 씀)
Content-Type: text/html               ← HTML 웹페이지
Content-Type: text/plain              ← 일반 텍스트
Content-Type: multipart/form-data     ← 파일 업로드
```

#### Accept

클라이언트가 **원하는 응답 형식**을 서버에게 알려줍니다.

```
Accept: application/json    ← "JSON으로 응답해주세요"
Accept: text/html           ← "HTML로 응답해주세요"
Accept: */*                 ← "아무 형식이나 괜찮아요"
```

#### Authorization

**인증 정보**를 담습니다. 로그인 후 발급받은 토큰을 여기에 넣습니다.

```
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...    ← JWT 토큰
Authorization: Basic dXNlcjpwYXNz               ← ID/비밀번호 (Base64)
```

> 22장(인증과 인가)에서 JWT 토큰을 직접 만들고 검증하는 방법을 배웁니다.

### 요청 vs 응답 헤더

| 헤더 | 요청에서 | 응답에서 |
|------|---------|---------|
| Content-Type | 보내는 데이터 형식 | 응답 데이터 형식 |
| Content-Length | 보내는 데이터 크기 | 응답 데이터 크기 |
| Accept | 원하는 응답 형식 | (사용 안 함) |
| Authorization | 인증 토큰 | (사용 안 함) |
| Set-Cookie | (사용 안 함) | 쿠키 설정 |

---

## 6. JSON

### JSON이란?

**JSON (JavaScript Object Notation)** 은 웹에서 데이터를 주고받는 **표준 형식**입니다.

사람이 읽기 쉽고, 기계가 파싱하기도 쉬운 텍스트 형식입니다:

```json
{
  "name": "홍길동",
  "age": 20,
  "is_student": true,
  "hobbies": ["코딩", "독서", "게임"],
  "address": {
    "city": "서울",
    "zip": "12345"
  }
}
```

### JSON 기본 규칙

```
┌─────────────────────────────────────────┐
│            JSON 데이터 타입               │
├─────────────────────────────────────────┤
│  문자열    "hello"       (큰따옴표 필수!)  │
│  숫자      42, 3.14     (따옴표 없음)     │
│  불리언    true, false   (소문자!)        │
│  null     null          (값 없음)        │
│  배열     [1, 2, 3]     (대괄호)         │
│  객체     {"key": "val"} (중괄호)        │
└─────────────────────────────────────────┘
```

| JSON 타입 | 예시 | Rust 타입 |
|----------|------|----------|
| 문자열 | `"hello"` | `String` |
| 숫자 (정수) | `42` | `i32`, `i64` 등 |
| 숫자 (실수) | `3.14` | `f64` |
| 불리언 | `true` | `bool` |
| null | `null` | `Option<T>` (None) |
| 배열 | `[1, 2, 3]` | `Vec<T>` |
| 객체 | `{"key": "val"}` | `struct` 또는 `HashMap` |

### Rust에서 JSON 처리 맛보기

Rust에서 JSON을 다루려면 **serde**와 **serde_json** 크레이트를 사용합니다.

```toml
# Cargo.toml
[dependencies]
serde = { version = "1", features = ["derive"] }
serde_json = "1"
```

#### JSON 문자열 만들기

```rust
use serde::Serialize;

#[derive(Serialize)]
struct User {
    name: String,
    age: u32,
    hobbies: Vec<String>,
}

fn main() {
    let user = User {
        name: String::from("홍길동"),
        age: 20,
        hobbies: vec![
            String::from("코딩"),
            String::from("독서"),
        ],
    };

    // Rust 구조체 -> JSON 문자열
    let json_string = serde_json::to_string_pretty(&user).unwrap();
    println!("{}", json_string);
}
```

출력:
```json
{
  "name": "홍길동",
  "age": 20,
  "hobbies": [
    "코딩",
    "독서"
  ]
}
```

#### JSON 문자열 파싱하기

```rust
use serde::Deserialize;

#[derive(Deserialize, Debug)]
struct User {
    name: String,
    age: u32,
    hobbies: Vec<String>,
}

fn main() {
    let json_str = r#"
    {
        "name": "홍길동",
        "age": 20,
        "hobbies": ["코딩", "독서"]
    }
    "#;

    // JSON 문자열 -> Rust 구조체
    let user: User = serde_json::from_str(json_str).unwrap();
    println!("{:?}", user);
    println!("이름: {}, 나이: {}", user.name, user.age);
}
```

```
직렬화 (Serialize):   Rust 구조체  ──→  JSON 문자열
                      User { ... }      {"name": "홍길동", ...}

역직렬화 (Deserialize): JSON 문자열  ──→  Rust 구조체
                        {"name": ...}    User { ... }
```

> 지금은 serde가 이런 일을 한다는 것만 알면 됩니다. 17장에서 Axum과 함께 실전으로 사용합니다.

<details>
<summary><b>왜 JSON이 표준이 되었나? (원리)</b></summary>

### JSON 이전: XML의 시대

JSON이 등장하기 전에는 **XML**이 데이터 교환 표준이었습니다:

```xml
<!-- XML: 같은 데이터를 XML로 표현하면 -->
<user>
  <name>홍길동</name>
  <age>20</age>
  <hobbies>
    <hobby>코딩</hobby>
    <hobby>독서</hobby>
  </hobbies>
</user>
```

```json
// JSON: 훨씬 간결!
{
  "name": "홍길동",
  "age": 20,
  "hobbies": ["코딩", "독서"]
}
```

### JSON이 이긴 이유

| 비교 항목 | XML | JSON |
|----------|-----|------|
| 가독성 | 태그가 많아 복잡 | 간결하고 읽기 쉬움 |
| 데이터 크기 | 큼 (태그 반복) | 작음 |
| 파싱 속도 | 느림 | 빠름 |
| JavaScript 호환 | 별도 파서 필요 | 네이티브 지원 |

JSON은 이름에 "JavaScript"가 들어있지만, 사실상 **모든 프로그래밍 언어에서 지원**합니다. Rust, Python, Go, Java 등 어디서든 JSON을 쉽게 다룰 수 있습니다.

### 다른 형식들

JSON 외에도 데이터 교환 형식이 있습니다:

| 형식 | 특징 | 사용 사례 |
|------|------|----------|
| JSON | 가장 보편적, 사람이 읽기 쉬움 | REST API |
| XML | 오래된 표준, 복잡하지만 강력 | 레거시 시스템, SOAP |
| YAML | JSON보다 더 읽기 쉬움 | 설정 파일 (Docker, K8s) |
| Protocol Buffers | 바이너리 형식, 매우 빠름 | gRPC, 내부 서비스 통신 |
| MessagePack | 바이너리 JSON | 성능이 중요한 API |

웹 API에서는 **JSON이 사실상 표준**이므로, 이 학습지에서도 JSON을 중심으로 다룹니다.

</details>

---

## 7. REST API 개념

### REST란?

**REST (Representational State Transfer)** 는 API를 설계하는 **규칙**입니다. 핵심은 **리소스(자원) 중심으로 URL을 설계**하는 것입니다.

### 리소스 중심 URL 설계

```
좋은 URL (리소스 중심, 명사 사용):
  /users          ← 사용자 목록
  /users/1        ← 1번 사용자
  /users/1/posts  ← 1번 사용자의 게시글 목록

나쁜 URL (동사 사용):
  /getUsers       ← 동작을 URL에 넣지 마세요!
  /createUser     ← HTTP 메서드가 동작을 나타냅니다
  /deleteUser/1   ← DELETE /users/1 로 표현하세요
```

**동작은 HTTP 메서드로, 대상은 URL로** 표현합니다.

### CRUD와 HTTP 메서드 매핑

**CRUD**는 데이터의 기본 동작 4가지를 말합니다: Create(생성), Read(조회), Update(수정), Delete(삭제).

```
┌─────────────────────────────────────────────────────────────┐
│                  사용자(User) REST API                        │
├──────────┬───────────┬──────────────┬───────────────────────┤
│   동작    │ HTTP 메서드 │   URL 경로    │     설명              │
├──────────┼───────────┼──────────────┼───────────────────────┤
│ 목록 조회  │ GET       │ /users       │ 모든 사용자 조회        │
│ 단건 조회  │ GET       │ /users/1     │ 1번 사용자 조회         │
│ 생성      │ POST      │ /users       │ 새 사용자 생성          │
│ 수정      │ PUT       │ /users/1     │ 1번 사용자 정보 수정     │
│ 삭제      │ DELETE    │ /users/1     │ 1번 사용자 삭제         │
└──────────┴───────────┴──────────────┴───────────────────────┘
```

### 실제 요청/응답 예시

#### 목록 조회

```
요청: GET /users HTTP/1.1

응답: 200 OK
[
  {"id": 1, "name": "홍길동", "age": 20},
  {"id": 2, "name": "김영희", "age": 25}
]
```

#### 단건 조회

```
요청: GET /users/1 HTTP/1.1

응답: 200 OK
{"id": 1, "name": "홍길동", "age": 20}
```

#### 생성

```
요청: POST /users HTTP/1.1
Content-Type: application/json

{"name": "이철수", "age": 30}

응답: 201 Created
{"id": 3, "name": "이철수", "age": 30}
```

#### 수정

```
요청: PUT /users/1 HTTP/1.1
Content-Type: application/json

{"name": "홍길동", "age": 21}

응답: 200 OK
{"id": 1, "name": "홍길동", "age": 21}
```

#### 삭제

```
요청: DELETE /users/1 HTTP/1.1

응답: 204 No Content
(바디 없음)
```

### 전체 흐름 다이어그램

```
클라이언트                              서버
    │                                   │
    │  GET /users                       │
    │ ─────────────────────────────>    │
    │                                   │  DB에서 사용자 목록 조회
    │         200 OK + [사용자 목록]      │
    │ <─────────────────────────────    │
    │                                   │
    │  POST /users + {새 사용자 데이터}    │
    │ ─────────────────────────────>    │
    │                                   │  DB에 새 사용자 저장
    │         201 Created + {생성된 데이터} │
    │ <─────────────────────────────    │
    │                                   │
    │  GET /users/999                   │
    │ ─────────────────────────────>    │
    │                                   │  DB에서 찾음 -> 없음
    │         404 Not Found             │
    │ <─────────────────────────────    │
```

<details>
<summary><b>REST의 제약 조건과 RESTful이란 (원리)</b></summary>

### REST의 6가지 제약 조건

REST를 정의한 Roy Fielding의 논문에서는 6가지 제약 조건을 제시합니다:

1. **클라이언트-서버 분리**: 프론트엔드와 백엔드가 독립적으로 발전
2. **무상태성(Stateless)**: 각 요청은 독립적, 서버가 이전 요청을 기억하지 않음
3. **캐시 가능(Cacheable)**: 응답에 캐시 가능 여부를 명시
4. **계층화 시스템**: 중간에 프록시, 로드밸런서 등을 넣을 수 있음
5. **균일한 인터페이스**: URL과 HTTP 메서드로 일관된 API 제공
6. **Code-on-Demand** (선택): 서버가 클라이언트에 실행 코드를 전송 (거의 안 씀)

### "RESTful" API란?

REST 제약 조건을 잘 지키는 API를 **RESTful API**라고 부릅니다.

```
RESTful한 설계:
  GET    /articles         → 게시글 목록
  GET    /articles/42      → 42번 게시글
  POST   /articles         → 게시글 작성
  PUT    /articles/42      → 42번 게시글 수정
  DELETE /articles/42      → 42번 게시글 삭제

RESTful하지 않은 설계:
  GET    /getArticles      → 동사가 URL에 들어감
  POST   /articles/delete  → DELETE 메서드 대신 POST 사용
  GET    /articles?action=create → 액션을 쿼리로 전달
```

실무에서 완벽한 REST를 지키는 경우는 드뭅니다. 중요한 건 **팀 내에서 일관된 규칙**을 정하고 따르는 것입니다.

</details>

---

## 8. 실습

### 실습 1: curl로 공개 API 호출해보기

터미널(명령 프롬프트)에서 curl 명령어로 실제 API를 호출해봅시다.

> curl은 HTTP 요청을 보내는 명령줄 도구입니다. Windows에는 기본 설치되어 있습니다.

**Step 1: GET 요청 보내기**

아래 명령어를 터미널에 입력하세요:

```bash
curl https://jsonplaceholder.typicode.com/users/1
```

이 명령어는 "1번 사용자 정보를 조회해줘"라는 GET 요청을 보냅니다.

**예상 응답** (일부):
```json
{
  "id": 1,
  "name": "Leanne Graham",
  "username": "Bret",
  "email": "Sincere@april.biz"
}
```

**Step 2: 헤더 정보와 함께 보기**

```bash
curl -v https://jsonplaceholder.typicode.com/users/1
```

`-v` (verbose) 옵션을 붙이면 요청 헤더와 응답 헤더까지 모두 볼 수 있습니다.

출력에서 다음 항목들을 찾아보세요:
- `> GET /users/1 HTTP/1.1` (요청 라인)
- `< HTTP/1.1 200 OK` (상태 라인)
- `< content-type: application/json` (응답 헤더)

**Step 3: POST 요청 보내기**

```bash
curl -X POST https://jsonplaceholder.typicode.com/posts -H "Content-Type: application/json" -d "{\"title\": \"hello\", \"body\": \"world\", \"userId\": 1}"
```

| 옵션 | 의미 |
|------|------|
| `-X POST` | POST 메서드 사용 |
| `-H "..."` | 헤더 추가 |
| `-d "{...}"` | 바디 데이터 |

**확인할 것**: 응답의 상태 코드가 `201 Created`인지 확인하세요!

---

### 실습 2: HTTP 요청/응답 직접 적어보기

아래 상황에 맞는 HTTP 요청과 응답을 직접 적어보세요.

#### 문제 2-1: 게시글 목록 조회

상황: 게시판의 모든 게시글을 조회합니다.

요청을 적어보세요:
```
_______ /posts HTTP/1.1
Host: api.example.com
Accept: ___________________
```

<details>
<summary>정답 보기</summary>

```
GET /posts HTTP/1.1
Host: api.example.com
Accept: application/json
```

게시글 "목록 조회"이므로 **GET** 메서드를 사용합니다. JSON 응답을 원하므로 Accept에 `application/json`을 적습니다.

</details>

#### 문제 2-2: 새 게시글 작성

상황: 제목이 "안녕하세요"이고 내용이 "첫 게시글입니다"인 게시글을 작성합니다.

요청을 적어보세요:
```
_______ /posts HTTP/1.1
Host: api.example.com
Content-Type: ___________________

______________________________
```

<details>
<summary>정답 보기</summary>

```
POST /posts HTTP/1.1
Host: api.example.com
Content-Type: application/json

{"title": "안녕하세요", "content": "첫 게시글입니다"}
```

새 게시글 "작성"이므로 **POST** 메서드를 사용합니다. JSON 데이터를 보내므로 Content-Type은 `application/json`입니다.

</details>

#### 문제 2-3: 응답 상태 코드 고르기

다음 상황에서 서버가 반환해야 할 상태 코드를 적어보세요:

| 상황 | 상태 코드 |
|------|----------|
| 사용자 목록을 성공적으로 반환 | ______ |
| 새 게시글이 성공적으로 생성됨 | ______ |
| 요청한 게시글 번호가 존재하지 않음 | ______ |
| JSON 형식이 잘못되어 파싱 실패 | ______ |
| 서버 코드에 버그가 있어서 에러 발생 | ______ |

<details>
<summary>정답 보기</summary>

| 상황 | 상태 코드 |
|------|----------|
| 사용자 목록을 성공적으로 반환 | **200 OK** |
| 새 게시글이 성공적으로 생성됨 | **201 Created** |
| 요청한 게시글 번호가 존재하지 않음 | **404 Not Found** |
| JSON 형식이 잘못되어 파싱 실패 | **400 Bad Request** |
| 서버 코드에 버그가 있어서 에러 발생 | **500 Internal Server Error** |

</details>

---

### 실습 3: JSON 데이터 구조 설계해보기

간단한 게시판 API를 위한 JSON 데이터 구조를 설계해봅시다.

#### 요구사항

게시판 시스템에는 다음 기능이 필요합니다:
- 게시글: 제목, 내용, 작성자 이름, 작성 날짜, 좋아요 수
- 댓글: 내용, 작성자 이름, 작성 날짜

#### 문제 3-1: 게시글 1건의 JSON 구조를 설계하세요

```json
{
  // 여기에 설계하세요!
}
```

<details>
<summary>정답 보기</summary>

```json
{
  "id": 1,
  "title": "Rust 공부 시작!",
  "content": "오늘부터 Rust를 배워보려고 합니다.",
  "author": "홍길동",
  "created_at": "2026-02-16T10:30:00",
  "likes": 5,
  "comments": [
    {
      "id": 1,
      "content": "화이팅!",
      "author": "김영희",
      "created_at": "2026-02-16T11:00:00"
    },
    {
      "id": 2,
      "content": "저도 같이 공부해요!",
      "author": "이철수",
      "created_at": "2026-02-16T11:30:00"
    }
  ]
}
```

**설계 포인트**:
- `id`는 각 리소스를 구분하는 고유 번호입니다
- `created_at`은 ISO 8601 형식(`YYYY-MM-DDTHH:MM:SS`)을 사용합니다
- `comments`는 배열로, 게시글에 달린 댓글 목록을 포함합니다
- JSON 키는 `snake_case`를 사용합니다 (Rust 관례와 동일)

</details>

#### 문제 3-2: 이 게시글에 대응하는 Rust 구조체를 작성하세요

<details>
<summary>정답 보기</summary>

```rust
use serde::{Serialize, Deserialize};

#[derive(Debug, Serialize, Deserialize)]
struct Post {
    id: u64,
    title: String,
    content: String,
    author: String,
    created_at: String,
    likes: u32,
    comments: Vec<Comment>,
}

#[derive(Debug, Serialize, Deserialize)]
struct Comment {
    id: u64,
    content: String,
    author: String,
    created_at: String,
}
```

**포인트**: JSON의 구조가 곧 Rust 구조체의 구조가 됩니다. serde의 `Serialize`와 `Deserialize`를 derive하면 JSON과 자동으로 변환할 수 있습니다.

</details>

---

## 9. 확인 문제

### 문제 1

HTTP 요청의 4가지 구성 요소를 순서대로 나열하세요.
- (a) 바디 -> 헤더 -> 요청 라인 -> 빈 줄
- (b) 요청 라인 -> 헤더 -> 빈 줄 -> 바디
- (c) 헤더 -> 요청 라인 -> 바디 -> 빈 줄
- (d) 요청 라인 -> 바디 -> 헤더 -> 빈 줄

### 문제 2

아래 요청에서 HTTP 메서드, 경로, Content-Type을 찾으세요:
```
POST /api/products HTTP/1.1
Host: shop.example.com
Content-Type: application/json
Authorization: Bearer token123

{"name": "키보드", "price": 50000}
```

### 문제 3

다음 상황에 맞는 HTTP 메서드를 고르세요:

| 상황 | HTTP 메서드 |
|------|-----------|
| 상품 목록을 가져오기 | ______ |
| 장바구니에 상품 추가하기 | ______ |
| 배송지 주소 변경하기 | ______ |
| 주문 취소하기 | ______ |

### 문제 4

상태 코드 `401`과 `403`의 차이를 한 문장으로 설명하세요.

### 문제 5

다음 JSON에서 잘못된 부분을 모두 찾으세요:
```json
{
  name: "홍길동",
  "age": 20,
  "active": True,
  "skills": ["Rust", "Python",]
}
```

---

## 확인 문제 정답

<details>
<summary>정답 보기 (먼저 풀어본 후 클릭하세요!)</summary>

### 문제 1 정답: **(b) 요청 라인 -> 헤더 -> 빈 줄 -> 바디**

HTTP 요청은 항상 이 순서를 따릅니다:
```
요청 라인:  GET /users HTTP/1.1
헤더:       Host: example.com
            Content-Type: application/json
빈 줄:      (헤더와 바디를 구분)
바디:       {"name": "홍길동"}
```

### 문제 2 정답

- **HTTP 메서드**: `POST`
- **경로**: `/api/products`
- **Content-Type**: `application/json`

요청 라인 `POST /api/products HTTP/1.1`에서 메서드와 경로를 읽고, 헤더에서 `Content-Type`을 찾으면 됩니다.

### 문제 3 정답

| 상황 | HTTP 메서드 |
|------|-----------|
| 상품 목록을 가져오기 | **GET** |
| 장바구니에 상품 추가하기 | **POST** |
| 배송지 주소 변경하기 | **PUT** |
| 주문 취소하기 | **DELETE** |

조회 = GET, 생성 = POST, 수정 = PUT, 삭제 = DELETE

### 문제 4 정답

`401 Unauthorized`는 **인증(로그인)이 안 된 상태**이고, `403 Forbidden`은 **인증은 됐지만 해당 리소스에 대한 권한이 없는 상태**입니다.

### 문제 5 정답

3가지 오류가 있습니다:

```json
{
  name: "홍길동",          // 오류 1: 키에 큰따옴표 없음 → "name"
  "age": 20,
  "active": True,         // 오류 2: True → true (소문자여야 함)
  "skills": ["Rust", "Python",]  // 오류 3: 마지막 쉼표(trailing comma) 불가
}
```

올바른 JSON:
```json
{
  "name": "홍길동",
  "age": 20,
  "active": true,
  "skills": ["Rust", "Python"]
}
```

</details>

---

## 10. 16장 정리

| 배운 것 | 핵심 |
|---------|------|
| HTTP | 클라이언트-서버 간 통신 규칙. 요청 -> 응답 구조 |
| HTTP 메서드 | GET(조회), POST(생성), PUT(수정), DELETE(삭제) |
| URL 경로 | 리소스의 위치를 나타냄. `/users/1` |
| 상태 코드 | 2xx 성공, 4xx 클라이언트 에러, 5xx 서버 에러 |
| 헤더 | Content-Type(데이터 형식), Authorization(인증) 등 추가 정보 |
| JSON | 웹 표준 데이터 형식. `{"key": "value"}` |
| serde | Rust에서 JSON을 다루는 크레이트. Serialize/Deserialize |
| REST API | 리소스 중심 URL + HTTP 메서드로 CRUD 표현 |

### HTTP 요청/응답 전체 흐름 한눈에

```
클라이언트                                    서버
┌──────────┐                            ┌──────────┐
│          │   1. 요청 (Request)          │          │
│ 브라우저  │   ┌───────────────────┐     │ 백엔드    │
│ 또는     │──>│ POST /users       │────>│ (Axum)   │
│ 앱       │   │ Content-Type: JSON│     │          │
│          │   │ {"name":"홍길동"}  │     │          │
│          │   └───────────────────┘     │          │
│          │                             │  처리...  │
│          │   2. 응답 (Response)         │          │
│          │   ┌───────────────────┐     │          │
│          │<──│ 201 Created       │<────│          │
│          │   │ Content-Type: JSON│     │          │
│          │   │ {"id":1,"name":..}│     │          │
│          │   └───────────────────┘     │          │
└──────────┘                            └──────────┘
```

---

## 다음 장 예고

> **17장. Axum으로 첫 웹서버**에서는 이번 장에서 배운 HTTP 이론을 직접 코드로 구현합니다.
> Axum 웹 프레임워크로 진짜 동작하는 서버를 만들고, GET/POST 요청을 처리하고, JSON 응답을 반환하는 방법을 배웁니다!

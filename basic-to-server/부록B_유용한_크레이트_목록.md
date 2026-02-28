# 부록 B. 유용한 크레이트 목록

> **이 학습지에서 사용하는 크레이트와, 실무에서 자주 쓰이는 크레이트를 정리했습니다.**
>
> 크레이트(crate)는 Rust의 라이브러리 패키지입니다. [crates.io](https://crates.io)에서 검색할 수 있습니다.

---

## 1. 웹 개발

| 크레이트 | 설명 | Cargo.toml 추가 | 관련 장 |
|----------|------|-----------------|---------|
| `axum` | Rust 웹 프레임워크 (라우팅, 핸들러 등) | `cargo add axum` | 17~18장 |
| `tokio` | 비동기 런타임 (async/await 실행 엔진) | `cargo add tokio --features full` | 14~15장 |
| `tower` | 미들웨어 프레임워크 (axum의 기반) | `cargo add tower` | 18장 |
| `tower-http` | HTTP 관련 미들웨어 모음 (CORS, 로깅 등) | `cargo add tower-http --features cors,trace` | 18장 |
| `hyper` | 저수준 HTTP 라이브러리 (axum이 내부적으로 사용) | `cargo add hyper` | - |

### 사용 예시

```bash
# 웹서버 프로젝트 기본 의존성 추가
cargo add axum
cargo add tokio --features full
cargo add tower-http --features cors,trace
```

```toml
# Cargo.toml
[dependencies]
axum = "0.8"
tokio = { version = "1", features = ["full"] }
tower-http = { version = "0.6", features = ["cors", "trace"] }
```

---

## 2. 직렬화 / 역직렬화

| 크레이트 | 설명 | Cargo.toml 추가 | 관련 장 |
|----------|------|-----------------|---------|
| `serde` | 직렬화/역직렬화 프레임워크 (핵심) | `cargo add serde --features derive` | 17장 |
| `serde_json` | JSON 변환 (serde 기반) | `cargo add serde_json` | 17장 |
| `toml` | TOML 파일 파싱 | `cargo add toml` | 21장 |

### 사용 예시

```rust
use serde::{Deserialize, Serialize};

#[derive(Serialize, Deserialize)]
struct User {
    name: String,
    age: u32,
}

// 구조체 -> JSON 문자열
let user = User { name: "철수".to_string(), age: 25 };
let json = serde_json::to_string(&user).unwrap();
// {"name":"철수","age":25}

// JSON 문자열 -> 구조체
let user: User = serde_json::from_str(&json).unwrap();
```

---

## 3. 데이터베이스

| 크레이트 | 설명 | Cargo.toml 추가 | 관련 장 |
|----------|------|-----------------|---------|
| `sqlx` | 비동기 SQL 라이브러리 (컴파일 타임 쿼리 검증) | `cargo add sqlx --features runtime-tokio,postgres` | 19~20장 |
| `diesel` | ORM 라이브러리 (동기 방식) | `cargo add diesel --features postgres` | - |
| `sea-orm` | 비동기 ORM 라이브러리 | `cargo add sea-orm` | - |

### 이 학습지에서는 sqlx를 사용합니다

```bash
# SQLx + PostgreSQL 설정
cargo add sqlx --features runtime-tokio,postgres,macros,migrate
```

```rust
use sqlx::PgPool;

let pool = PgPool::connect("postgres://user:pass@localhost/mydb").await?;

let users = sqlx::query_as!(User, "SELECT id, name FROM users")
    .fetch_all(&pool)
    .await?;
```

---

## 4. 인증 / 보안

| 크레이트 | 설명 | Cargo.toml 추가 | 관련 장 |
|----------|------|-----------------|---------|
| `jsonwebtoken` | JWT 토큰 생성과 검증 | `cargo add jsonwebtoken` | 22장 |
| `argon2` | 비밀번호 해싱 (권장) | `cargo add argon2` | 22장 |
| `bcrypt` | 비밀번호 해싱 (간편) | `cargo add bcrypt` | - |

### 사용 예시

```rust
// 비밀번호 해싱 (argon2)
use argon2::{self, Argon2, PasswordHasher, PasswordVerifier};
use argon2::password_hash::{SaltString, rand_core::OsRng};

// 해싱
let salt = SaltString::generate(&mut OsRng);
let hash = Argon2::default()
    .hash_password(b"my_password", &salt)?
    .to_string();

// 검증
let parsed_hash = argon2::PasswordHash::new(&hash)?;
Argon2::default().verify_password(b"my_password", &parsed_hash)?;
```

---

## 5. 유효성 검사

| 크레이트 | 설명 | Cargo.toml 추가 | 관련 장 |
|----------|------|-----------------|---------|
| `validator` | 구조체 필드 유효성 검사 | `cargo add validator --features derive` | 23장 |

### 사용 예시

```rust
use validator::Validate;

#[derive(Validate)]
struct SignupRequest {
    #[validate(email)]
    email: String,

    #[validate(length(min = 8, max = 100))]
    password: String,

    #[validate(length(min = 2, max = 50))]
    username: String,
}

let req = SignupRequest { /* ... */ };
req.validate()?; // 유효하지 않으면 에러 반환
```

---

## 6. 설정 관리

| 크레이트 | 설명 | Cargo.toml 추가 | 관련 장 |
|----------|------|-----------------|---------|
| `dotenv` | .env 파일에서 환경변수 로드 | `cargo add dotenv` | 21장 |
| `config` | 다양한 형식의 설정 파일 관리 | `cargo add config` | 21장 |

### 사용 예시

```bash
# .env 파일
DATABASE_URL=postgres://user:pass@localhost/mydb
JWT_SECRET=my_secret_key
```

```rust
use dotenv::dotenv;
use std::env;

fn main() {
    dotenv().ok(); // .env 파일 로드
    let db_url = env::var("DATABASE_URL").expect("DATABASE_URL이 설정되지 않았습니다");
}
```

---

## 7. 로깅

| 크레이트 | 설명 | Cargo.toml 추가 | 관련 장 |
|----------|------|-----------------|---------|
| `tracing` | 구조화된 로깅 프레임워크 | `cargo add tracing` | 18장, 25장 |
| `tracing-subscriber` | tracing 출력 설정 | `cargo add tracing-subscriber` | 18장, 25장 |
| `env_logger` | 환경변수로 제어하는 간단한 로거 | `cargo add env_logger` | - |

### 사용 예시

```rust
use tracing::{info, warn, error};

fn main() {
    // 로깅 초기화
    tracing_subscriber::fmt::init();

    info!("서버가 시작되었습니다");
    warn!("설정 파일이 없어서 기본값을 사용합니다");
    error!("데이터베이스 연결에 실패했습니다");
}
```

실행 시 로그 레벨을 환경변수로 조절할 수 있습니다:

```bash
# 모든 로그 출력
RUST_LOG=trace cargo run

# info 이상만 출력
RUST_LOG=info cargo run
```

---

## 8. 에러 처리

| 크레이트 | 설명 | Cargo.toml 추가 | 관련 장 |
|----------|------|-----------------|---------|
| `thiserror` | 커스텀 에러 타입을 쉽게 만들기 | `cargo add thiserror` | 7장, 23장 |
| `anyhow` | 간편한 에러 처리 (프로토타이핑에 좋음) | `cargo add anyhow` | 7장 |

### thiserror vs anyhow

- **thiserror**: 라이브러리나 API 서버처럼 에러 타입을 명확히 정의해야 할 때
- **anyhow**: 스크립트나 빠른 프로토타이핑에서 간편하게 에러를 처리할 때

```rust
// thiserror - 커스텀 에러 타입 정의
use thiserror::Error;

#[derive(Error, Debug)]
enum AppError {
    #[error("사용자를 찾을 수 없습니다: {0}")]
    UserNotFound(String),

    #[error("데이터베이스 에러")]
    Database(#[from] sqlx::Error),
}
```

```rust
// anyhow - 간편한 에러 처리
use anyhow::{Context, Result};

fn read_config() -> Result<String> {
    let content = std::fs::read_to_string("config.toml")
        .context("설정 파일을 읽을 수 없습니다")?;
    Ok(content)
}
```

---

## 9. 유틸리티

| 크레이트 | 설명 | Cargo.toml 추가 | 관련 장 |
|----------|------|-----------------|---------|
| `chrono` | 날짜와 시간 처리 | `cargo add chrono` | 22~23장 |
| `uuid` | 고유 ID 생성 | `cargo add uuid --features v4` | 23장 |
| `rand` | 난수 생성 | `cargo add rand` | - |
| `regex` | 정규 표현식 | `cargo add regex` | - |

### 사용 예시

```rust
// UUID 생성
use uuid::Uuid;
let id = Uuid::new_v4();
println!("{}", id); // 예: "67e55044-10b1-426f-9247-bb680e5fe0c8"

// 현재 시간
use chrono::Utc;
let now = Utc::now();
println!("{}", now); // 예: "2026-02-16 12:00:00 UTC"

// 난수 생성
use rand::Rng;
let mut rng = rand::rng();
let n: u32 = rng.random_range(1..=100);
println!("랜덤 숫자: {}", n);
```

---

## 10. 테스트

| 크레이트 | 설명 | Cargo.toml 추가 | 관련 장 |
|----------|------|-----------------|---------|
| `mockall` | 목(mock) 객체 생성 | `cargo add mockall --dev` | 24장 |
| `wiremock` | HTTP 서버 모킹 (API 테스트용) | `cargo add wiremock --dev` | 24장 |
| `fake` | 가짜 테스트 데이터 생성 | `cargo add fake --dev --features derive` | 24장 |

`--dev` 옵션은 개발 의존성으로 추가합니다. 테스트할 때만 사용되고, 최종 빌드에는 포함되지 않습니다.

### 사용 예시

```rust
// fake - 테스트용 가짜 데이터 생성
use fake::{Fake, faker::internet::en::SafeEmail};

let email: String = SafeEmail().fake();
println!("{}", email); // 예: "test123@example.com"
```

---

## 한눈에 보기: 웹서버 프로젝트 기본 의존성

학습지의 최종 프로젝트(실전 API 서버)에서 사용하는 주요 크레이트입니다:

```toml
[dependencies]
# 웹 프레임워크
axum = "0.8"
tokio = { version = "1", features = ["full"] }
tower-http = { version = "0.6", features = ["cors", "trace"] }

# 직렬화
serde = { version = "1", features = ["derive"] }
serde_json = "1"

# 데이터베이스
sqlx = { version = "0.8", features = ["runtime-tokio", "postgres", "macros", "migrate"] }

# 인증
jsonwebtoken = "9"
argon2 = "0.5"

# 유효성 검사
validator = { version = "0.19", features = ["derive"] }

# 설정
dotenv = "0.15"

# 로깅
tracing = "0.1"
tracing-subscriber = "0.3"

# 에러 처리
thiserror = "2"

# 유틸리티
chrono = { version = "0.4", features = ["serde"] }
uuid = { version = "1", features = ["v4", "serde"] }

[dev-dependencies]
# 테스트
mockall = "0.13"
wiremock = "0.6"
fake = { version = "3", features = ["derive"] }
```

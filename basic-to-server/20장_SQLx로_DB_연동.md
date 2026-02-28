# 20장. SQLx로 DB 연동

> **목표**: SQLx를 사용하여 Rust 코드에서 PostgreSQL과 연동할 수 있다.
> 💡 19장에서 배운 SQL과 PostgreSQL 위에, Rust 코드로 DB를 다루는 방법을 배웁니다.

---

## 1. 프로젝트 설정

### Cargo.toml 의존성 추가

```toml
[dependencies]
sqlx = { version = "0.8", features = ["runtime-tokio-rustls", "postgres"] }
tokio = { version = "1", features = ["full"] }
dotenv = "0.15"
```

| 크레이트 | 역할 |
|---------|------|
| `sqlx` | Rust용 비동기 SQL 라이브러리 |
| `tokio` | 비동기 런타임 (14~15장에서 배움) |
| `dotenv` | `.env` 파일에서 환경 변수 읽기 |

### .env 파일 설정

프로젝트 루트에 `.env` 파일을 만들고 DB 접속 정보를 넣습니다:

```
DATABASE_URL=postgres://user:password@localhost:5432/mydb
```

```
형식: postgres://사용자이름:비밀번호@호스트:포트/데이터베이스이름
```

> `.env` 파일에는 비밀번호가 포함되므로, `.gitignore`에 반드시 추가하세요!

---

## 2. 데이터베이스 연결

### PgPool 생성

```rust
use sqlx::postgres::PgPoolOptions;
use std::env;

#[tokio::main]
async fn main() -> Result<(), sqlx::Error> {
    // .env 파일 로드
    dotenv::dotenv().ok();

    // 환경 변수에서 DB URL 읽기
    let database_url = env::var("DATABASE_URL")
        .expect("DATABASE_URL이 설정되지 않았습니다");

    // 커넥션 풀 생성
    let pool = PgPoolOptions::new()
        .max_connections(5)
        .connect(&database_url)
        .await?;

    println!("DB 연결 성공!");

    Ok(())
}
```

실행 결과:
```
DB 연결 성공!
```

### 코드 흐름 정리

```
1. dotenv::dotenv()     → .env 파일에서 환경 변수 로드
2. env::var()           → DATABASE_URL 값 읽기
3. PgPoolOptions::new() → 풀 설정 생성
4. .max_connections(5)  → 최대 동시 연결 5개
5. .connect()           → 실제 DB 연결 (비동기)
```

<details>
<summary><b>🔍 커넥션 풀이란? 왜 필요한가? (원리)</b></summary>

### 커넥션 풀 없이

```
요청 1 → DB 연결 생성 → 쿼리 → 연결 닫기
요청 2 → DB 연결 생성 → 쿼리 → 연결 닫기
요청 3 → DB 연결 생성 → 쿼리 → 연결 닫기
```

매번 연결을 만들고 닫으면 **시간이 오래** 걸립니다. DB 연결 하나를 만드는 데 수십 밀리초가 걸릴 수 있습니다.

### 커넥션 풀 사용 시

```
시작 시 → 연결 5개 미리 생성해서 풀에 보관

요청 1 → 풀에서 연결 빌림 → 쿼리 → 연결 반납
요청 2 → 풀에서 연결 빌림 → 쿼리 → 연결 반납
요청 3 → 풀에서 연결 빌림 → 쿼리 → 연결 반납
```

연결을 **미리 만들어두고 재사용**하므로 훨씬 빠릅니다.

### 비유

```
커넥션 풀 없이 = 매번 새 택시 호출 (배차 대기 시간 발생)
커넥션 풀 사용 = 대기 중인 택시 5대를 항상 배치 (바로 탑승)
```

### max_connections 설정 기준

- **너무 적으면**: 요청이 많을 때 연결을 기다려야 함
- **너무 많으면**: DB 서버에 부담
- **일반적으로**: CPU 코어 수 x 2 정도가 적당

</details>

---

## 3. 쿼리 작성과 실행

### 구조체에 매핑하기

DB에서 가져온 행(row)을 Rust 구조체로 변환하려면 `FromRow`를 derive합니다:

```rust
#[derive(Debug, sqlx::FromRow)]
struct User {
    id: i32,
    name: String,
    email: String,
}
```

### 전체 조회 (SELECT)

```rust
// 모든 사용자 조회
let users = sqlx::query_as::<_, User>(
    "SELECT id, name, email FROM users"
)
    .fetch_all(&pool)
    .await?;

for user in &users {
    println!("{}: {} ({})", user.id, user.name, user.email);
}
```

### 조건 조회 (바인드 파라미터)

```rust
// 특정 사용자 조회 ($1은 첫 번째 바인드 파라미터)
let user_id = 1;

let user = sqlx::query_as::<_, User>(
    "SELECT id, name, email FROM users WHERE id = $1"
)
    .bind(user_id)
    .fetch_one(&pool)
    .await?;

println!("찾은 사용자: {} ({})", user.name, user.email);
```

**바인드 파라미터**(`$1`, `$2`, ...)를 사용하면 SQL 인젝션 공격을 방지할 수 있습니다.

```rust
// 절대 이렇게 하지 마세요! (SQL 인젝션 위험)
let query = format!("SELECT * FROM users WHERE name = '{}'", user_input);

// 항상 바인드 파라미터를 사용하세요
let query = sqlx::query_as::<_, User>("SELECT * FROM users WHERE name = $1")
    .bind(user_input);
```

### fetch 메서드 비교

| 메서드 | 반환 타입 | 설명 |
|--------|----------|------|
| `fetch_one()` | `T` | 정확히 1행. 없으면 에러 |
| `fetch_optional()` | `Option<T>` | 0~1행. 없으면 `None` |
| `fetch_all()` | `Vec<T>` | 모든 행을 벡터로 |

```rust
// fetch_optional: 사용자가 있을 수도 없을 수도 있을 때
let maybe_user = sqlx::query_as::<_, User>(
    "SELECT id, name, email FROM users WHERE email = $1"
)
    .bind("test@example.com")
    .fetch_optional(&pool)
    .await?;

match maybe_user {
    Some(user) => println!("찾았습니다: {}", user.name),
    None => println!("해당 이메일의 사용자가 없습니다"),
}
```

> `fetch_one()`은 결과가 없으면 에러를 반환합니다. 결과가 없을 수 있는 상황에서는 `fetch_optional()`을 사용하세요.

<details>
<summary><b>🔍 sqlx::query vs sqlx::query_as 차이 (원리)</b></summary>

### sqlx::query()

행을 **구조체로 매핑하지 않고** 직접 컬럼에 접근합니다:

```rust
let row = sqlx::query("SELECT id, name FROM users WHERE id = $1")
    .bind(1)
    .fetch_one(&pool)
    .await?;

// 컬럼 이름으로 접근
let id: i32 = row.get("id");
let name: String = row.get("name");
```

### sqlx::query_as()

행을 **구조체에 자동 매핑**합니다:

```rust
let user = sqlx::query_as::<_, User>("SELECT id, name, email FROM users WHERE id = $1")
    .bind(1)
    .fetch_one(&pool)
    .await?;

// 바로 구조체 필드 사용
println!("{}", user.name);
```

### 어떤 걸 쓸까?

- **`query_as`**: 대부분의 경우 추천. 타입 안전하고 코드가 깔끔함
- **`query`**: 동적 쿼리나 구조체 매핑이 불필요한 단순 작업에 사용

### 타입 파라미터 `<_, User>`의 의미

```rust
sqlx::query_as::<_, User>(...)
//              ^   ^
//              |   └─ 매핑할 구조체 타입
//              └─ DB 백엔드 (자동 추론, _로 생략)
```

</details>

---

## 4. CRUD 구현

CRUD는 Create(생성), Read(조회), Update(수정), Delete(삭제)의 약자입니다.

### CREATE - 사용자 생성

```rust
async fn create_user(
    pool: &sqlx::PgPool,
    name: &str,
    email: &str,
) -> Result<User, sqlx::Error> {
    let user = sqlx::query_as::<_, User>(
        "INSERT INTO users (name, email) VALUES ($1, $2) RETURNING id, name, email"
    )
        .bind(name)
        .bind(email)
        .fetch_one(pool)
        .await?;

    Ok(user)
}
```

> `RETURNING` 절은 PostgreSQL 전용 문법으로, INSERT 후 생성된 행을 바로 반환합니다.

### READ - 사용자 조회

```rust
// 전체 조회
async fn get_all_users(pool: &sqlx::PgPool) -> Result<Vec<User>, sqlx::Error> {
    let users = sqlx::query_as::<_, User>(
        "SELECT id, name, email FROM users ORDER BY id"
    )
        .fetch_all(pool)
        .await?;

    Ok(users)
}

// 단건 조회
async fn get_user_by_id(
    pool: &sqlx::PgPool,
    id: i32,
) -> Result<Option<User>, sqlx::Error> {
    let user = sqlx::query_as::<_, User>(
        "SELECT id, name, email FROM users WHERE id = $1"
    )
        .bind(id)
        .fetch_optional(pool)
        .await?;

    Ok(user)
}
```

### UPDATE - 사용자 수정

```rust
async fn update_user_email(
    pool: &sqlx::PgPool,
    id: i32,
    new_email: &str,
) -> Result<Option<User>, sqlx::Error> {
    let user = sqlx::query_as::<_, User>(
        "UPDATE users SET email = $1 WHERE id = $2 RETURNING id, name, email"
    )
        .bind(new_email)
        .bind(id)
        .fetch_optional(pool)
        .await?;

    Ok(user)
}
```

### DELETE - 사용자 삭제

```rust
async fn delete_user(
    pool: &sqlx::PgPool,
    id: i32,
) -> Result<bool, sqlx::Error> {
    let result = sqlx::query("DELETE FROM users WHERE id = $1")
        .bind(id)
        .execute(pool)
        .await?;

    // rows_affected()로 실제 삭제된 행 수 확인
    Ok(result.rows_affected() > 0)
}
```

### 전체 사용 예시

```rust
use sqlx::postgres::PgPoolOptions;
use std::env;

#[derive(Debug, sqlx::FromRow)]
struct User {
    id: i32,
    name: String,
    email: String,
}

#[tokio::main]
async fn main() -> Result<(), sqlx::Error> {
    dotenv::dotenv().ok();
    let database_url = env::var("DATABASE_URL")
        .expect("DATABASE_URL이 설정되지 않았습니다");

    let pool = PgPoolOptions::new()
        .max_connections(5)
        .connect(&database_url)
        .await?;

    // 생성
    let user = create_user(&pool, "홍길동", "hong@example.com").await?;
    println!("생성됨: {:?}", user);

    // 조회
    let found = get_user_by_id(&pool, user.id).await?;
    println!("조회됨: {:?}", found);

    // 수정
    let updated = update_user_email(&pool, user.id, "new@example.com").await?;
    println!("수정됨: {:?}", updated);

    // 삭제
    let deleted = delete_user(&pool, user.id).await?;
    println!("삭제됨: {}", deleted);

    Ok(())
}
```

실행 결과:
```
생성됨: User { id: 1, name: "홍길동", email: "hong@example.com" }
조회됨: Some(User { id: 1, name: "홍길동", email: "hong@example.com" })
수정됨: Some(User { id: 1, name: "홍길동", email: "new@example.com" })
삭제됨: true
```

---

## 5. 마이그레이션

마이그레이션은 **코드로 DB 테이블 구조를 관리**하는 방법입니다.

### sqlx-cli 설치

```bash
cargo install sqlx-cli --no-default-features --features postgres
```

### 마이그레이션 생성

```bash
sqlx migrate add create_users
```

이 명령은 `migrations/` 폴더에 타임스탬프가 붙은 파일을 생성합니다:

```
migrations/
  └── 20240101000000_create_users.sql
```

### 마이그레이션 파일 작성

`migrations/20240101000000_create_users.sql`:

```sql
CREATE TABLE IF NOT EXISTS users (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(255) NOT NULL UNIQUE,
    created_at TIMESTAMP DEFAULT NOW()
);
```

| SQL 키워드 | 의미 |
|-----------|------|
| `SERIAL` | 자동 증가 정수 |
| `PRIMARY KEY` | 기본 키 (유일, NOT NULL) |
| `NOT NULL` | 빈 값 허용 안 함 |
| `UNIQUE` | 중복 값 허용 안 함 |
| `DEFAULT NOW()` | 기본값: 현재 시각 |

### 마이그레이션 실행

```bash
sqlx migrate run
```

실행 결과:
```
Applied 20240101000000/migrate create_users (32.521208ms)
```

### Rust 코드에서 마이그레이션 실행

코드에서 직접 실행할 수도 있습니다:

```rust
sqlx::migrate!("./migrations")
    .run(&pool)
    .await?;
```

<details>
<summary><b>🔍 왜 마이그레이션을 쓰나? DB 스키마 버전 관리 (원리)</b></summary>

### 마이그레이션 없이

```
개발자 A: "users 테이블에 phone 컬럼 추가했어"
개발자 B: "어? 내 DB에는 없는데?"
개발자 A: "직접 SQL 실행해줘: ALTER TABLE users ADD COLUMN phone VARCHAR(20)"
개발자 B: "서버에도 해야 하지 않아?"
→ 수동 관리 = 실수와 혼란
```

### 마이그레이션 사용 시

```
1. 개발자 A가 마이그레이션 파일 생성:
   migrations/20240102_add_phone.sql
   → ALTER TABLE users ADD COLUMN phone VARCHAR(20);

2. 코드를 git에 커밋

3. 개발자 B가 git pull 후 마이그레이션 실행:
   sqlx migrate run
   → 자동으로 phone 컬럼 추가됨

4. 서버 배포 시에도 동일하게 마이그레이션 실행
   → 모든 환경의 DB 구조가 동일
```

### 마이그레이션 = DB의 git

```
git이 코드의 변경 이력을 관리하듯,
마이그레이션은 DB 구조의 변경 이력을 관리합니다.

migrations/
  ├── 20240101_create_users.sql       ← 1차: 테이블 생성
  ├── 20240102_add_phone.sql          ← 2차: phone 컬럼 추가
  └── 20240103_add_avatar_url.sql     ← 3차: avatar_url 추가
```

SQLx는 `_sqlx_migrations` 테이블을 자동으로 만들어서, 어떤 마이그레이션이 이미 실행되었는지 추적합니다.

</details>

---

## 6. 트랜잭션

트랜잭션은 **여러 쿼리를 하나의 작업 단위**로 묶는 것입니다. 중간에 하나라도 실패하면 전부 취소(롤백)됩니다.

### 기본 사용법

```rust
async fn transfer_points(
    pool: &sqlx::PgPool,
    from_id: i32,
    to_id: i32,
    amount: i32,
) -> Result<(), sqlx::Error> {
    // 트랜잭션 시작
    let mut tx = pool.begin().await?;

    // 보내는 사람 포인트 차감
    sqlx::query("UPDATE users SET points = points - $1 WHERE id = $2")
        .bind(amount)
        .bind(from_id)
        .execute(&mut *tx)
        .await?;

    // 받는 사람 포인트 증가
    sqlx::query("UPDATE users SET points = points + $1 WHERE id = $2")
        .bind(amount)
        .bind(to_id)
        .execute(&mut *tx)
        .await?;

    // 모든 쿼리가 성공하면 커밋
    tx.commit().await?;

    Ok(())
}
```

### 왜 트랜잭션이 필요한가?

```
트랜잭션 없이 포인트 전송:
1. A에게서 100 포인트 차감 ✅
2. (서버 다운!) ❌
3. B에게 100 포인트 추가 (실행 안 됨)
→ A의 100 포인트가 증발!

트랜잭션 사용 시:
1. 트랜잭션 시작
2. A에게서 100 포인트 차감
3. (서버 다운!) ❌
4. 자동 롤백 → A의 포인트 원래대로 복구!
```

### 자동 롤백

`tx`가 `commit()` 없이 드롭되면 **자동으로 롤백**됩니다:

```rust
async fn example(pool: &sqlx::PgPool) -> Result<(), sqlx::Error> {
    let mut tx = pool.begin().await?;

    sqlx::query("INSERT INTO users (name, email) VALUES ($1, $2)")
        .bind("테스트")
        .bind("test@example.com")
        .execute(&mut *tx)
        .await?;

    // 여기서 에러가 발생하면?
    sqlx::query("이것은 잘못된 SQL입니다")
        .execute(&mut *tx)
        .await?;  // ← 에러 발생, ?로 함수 종료

    // commit에 도달하지 못함
    tx.commit().await?;

    Ok(())
    // tx가 드롭되면서 자동 롤백 → 첫 번째 INSERT도 취소됨
}
```

<details>
<summary><b>🔍 ACID 속성이란? (원리)</b></summary>

### ACID는 트랜잭션이 보장해야 할 4가지 속성입니다.

**A - Atomicity (원자성)**
```
트랜잭션의 모든 작업은 전부 성공하거나 전부 실패한다.
"반만 실행"은 없다.

예: 포인트 전송에서 차감만 되고 추가는 안 되는 상황 방지
```

**C - Consistency (일관성)**
```
트랜잭션 전후로 DB는 항상 유효한 상태를 유지한다.
제약 조건(UNIQUE, NOT NULL 등)이 항상 만족된다.

예: 이메일이 UNIQUE인데 중복 이메일 삽입 시 트랜잭션 실패
```

**I - Isolation (격리성)**
```
동시에 실행되는 트랜잭션들이 서로 영향을 주지 않는다.
각 트랜잭션은 마치 혼자 실행되는 것처럼 동작한다.

예: A가 잔액을 읽는 중에 B가 잔액을 변경해도,
    A는 변경 전 또는 변경 후 값만 봄 (중간 상태 안 봄)
```

**D - Durability (영속성)**
```
커밋된 트랜잭션의 결과는 영구적으로 저장된다.
서버가 꺼져도 데이터가 사라지지 않는다.

예: 결제 완료 후 서버 다운 → 재시작해도 결제 기록 유지
```

### 요약

| 속성 | 한줄 요약 |
|------|----------|
| Atomicity | 전부 아니면 전무 |
| Consistency | 규칙 항상 유지 |
| Isolation | 다른 트랜잭션과 독립 |
| Durability | 커밋하면 영구 저장 |

</details>

---

## 7. 실습

### 실습 1: DB 연결하고 테이블 생성 (마이그레이션)

sqlx-cli로 마이그레이션을 만들고 실행하여 `posts` 테이블을 생성하세요.

**요구사항:**
- `id`: 자동 증가 정수, 기본 키
- `title`: 최대 200자 문자열, NOT NULL
- `content`: 텍스트, NOT NULL
- `author_id`: 정수, NOT NULL (users 테이블 참조)
- `created_at`: 타임스탬프, 기본값 현재 시각

```bash
# 1. 마이그레이션 생성
sqlx migrate add create_posts

# 2. 생성된 파일 편집 (아래 SQL 작성)

# 3. 마이그레이션 실행
sqlx migrate run
```

<details>
<summary><b>정답 보기</b></summary>

`migrations/YYYYMMDD_create_posts.sql`:

```sql
CREATE TABLE IF NOT EXISTS posts (
    id SERIAL PRIMARY KEY,
    title VARCHAR(200) NOT NULL,
    content TEXT NOT NULL,
    author_id INTEGER NOT NULL REFERENCES users(id),
    created_at TIMESTAMP DEFAULT NOW()
);
```

**포인트**:
- `REFERENCES users(id)`는 외래 키(Foreign Key)로, `author_id`가 `users` 테이블의 `id`를 참조한다는 뜻입니다.
- 존재하지 않는 `author_id`를 넣으려 하면 DB가 거부합니다.

</details>

### 실습 2: 사용자 CRUD 함수 만들기

아래 구조를 참고하여 4개의 CRUD 함수를 완성하세요.

```rust
use sqlx::PgPool;

#[derive(Debug, sqlx::FromRow)]
struct User {
    id: i32,
    name: String,
    email: String,
}

// 1. 사용자 생성
async fn create_user(pool: &PgPool, name: &str, email: &str) -> Result<User, sqlx::Error> {
    // TODO: INSERT INTO users ... RETURNING ...
    todo!()
}

// 2. 이메일로 사용자 조회 (없을 수 있음)
async fn find_user_by_email(pool: &PgPool, email: &str) -> Result<Option<User>, sqlx::Error> {
    // TODO: SELECT ... WHERE email = $1
    todo!()
}

// 3. 사용자 이름 수정
async fn update_user_name(pool: &PgPool, id: i32, new_name: &str) -> Result<Option<User>, sqlx::Error> {
    // TODO: UPDATE ... SET name = $1 WHERE id = $2 RETURNING ...
    todo!()
}

// 4. 사용자 삭제
async fn delete_user(pool: &PgPool, id: i32) -> Result<bool, sqlx::Error> {
    // TODO: DELETE FROM users WHERE id = $1
    todo!()
}

#[tokio::main]
async fn main() -> Result<(), sqlx::Error> {
    dotenv::dotenv().ok();
    let database_url = std::env::var("DATABASE_URL")
        .expect("DATABASE_URL이 설정되지 않았습니다");

    let pool = sqlx::postgres::PgPoolOptions::new()
        .max_connections(5)
        .connect(&database_url)
        .await?;

    // 생성
    let user = create_user(&pool, "김철수", "kim@example.com").await?;
    println!("생성: {:?}", user);

    // 이메일로 조회
    let found = find_user_by_email(&pool, "kim@example.com").await?;
    println!("조회: {:?}", found);

    // 이름 수정
    let updated = update_user_name(&pool, user.id, "김영희").await?;
    println!("수정: {:?}", updated);

    // 삭제
    let deleted = delete_user(&pool, user.id).await?;
    println!("삭제: {}", deleted);

    Ok(())
}
```

<details>
<summary><b>정답 보기</b></summary>

```rust
async fn create_user(pool: &PgPool, name: &str, email: &str) -> Result<User, sqlx::Error> {
    let user = sqlx::query_as::<_, User>(
        "INSERT INTO users (name, email) VALUES ($1, $2) RETURNING id, name, email"
    )
        .bind(name)
        .bind(email)
        .fetch_one(pool)
        .await?;

    Ok(user)
}

async fn find_user_by_email(pool: &PgPool, email: &str) -> Result<Option<User>, sqlx::Error> {
    let user = sqlx::query_as::<_, User>(
        "SELECT id, name, email FROM users WHERE email = $1"
    )
        .bind(email)
        .fetch_optional(pool)
        .await?;

    Ok(user)
}

async fn update_user_name(pool: &PgPool, id: i32, new_name: &str) -> Result<Option<User>, sqlx::Error> {
    let user = sqlx::query_as::<_, User>(
        "UPDATE users SET name = $1 WHERE id = $2 RETURNING id, name, email"
    )
        .bind(new_name)
        .bind(id)
        .fetch_optional(pool)
        .await?;

    Ok(user)
}

async fn delete_user(pool: &PgPool, id: i32) -> Result<bool, sqlx::Error> {
    let result = sqlx::query("DELETE FROM users WHERE id = $1")
        .bind(id)
        .execute(pool)
        .await?;

    Ok(result.rows_affected() > 0)
}
```

실행 결과:
```
생성: User { id: 1, name: "김철수", email: "kim@example.com" }
조회: Some(User { id: 1, name: "김철수", email: "kim@example.com" })
수정: Some(User { id: 1, name: "김영희", email: "kim@example.com" })
삭제: true
```

**포인트**:
- `create_user`는 항상 결과가 있으므로 `fetch_one` 사용
- `find_user_by_email`과 `update_user_name`은 결과가 없을 수 있으므로 `fetch_optional` 사용
- `delete_user`는 삭제 성공 여부만 필요하므로 `execute` + `rows_affected()` 사용

</details>

### 실습 3: Axum 핸들러와 SQLx 연결하기

17장에서 배운 Axum과 SQLx를 결합하여 사용자 목록을 조회하는 API를 만드세요.

```toml
# Cargo.toml
[dependencies]
axum = "0.7"
sqlx = { version = "0.8", features = ["runtime-tokio-rustls", "postgres"] }
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
dotenv = "0.15"
```

```rust
use axum::{
    extract::State,
    routing::get,
    Json, Router,
};
use sqlx::PgPool;
use sqlx::postgres::PgPoolOptions;

#[derive(Debug, sqlx::FromRow, serde::Serialize)]
struct User {
    id: i32,
    name: String,
    email: String,
}

// TODO: 핸들러 함수 작성
// GET /users → 모든 사용자를 JSON으로 반환
async fn get_users(
    State(pool): State<PgPool>,
) -> Result<Json<Vec<User>>, String> {
    todo!()
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    dotenv::dotenv().ok();
    let database_url = std::env::var("DATABASE_URL")
        .expect("DATABASE_URL이 설정되지 않았습니다");

    let pool = PgPoolOptions::new()
        .max_connections(5)
        .connect(&database_url)
        .await?;

    let app = Router::new()
        .route("/users", get(get_users))
        .with_state(pool);

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await?;
    println!("서버 시작: http://localhost:3000");
    axum::serve(listener, app).await?;

    Ok(())
}
```

<details>
<summary><b>정답 보기</b></summary>

```rust
async fn get_users(
    State(pool): State<PgPool>,
) -> Result<Json<Vec<User>>, String> {
    let users = sqlx::query_as::<_, User>(
        "SELECT id, name, email FROM users ORDER BY id"
    )
        .fetch_all(&pool)
        .await
        .map_err(|e| format!("DB 에러: {}", e))?;

    Ok(Json(users))
}
```

**코드 흐름:**
```
1. Router에 .with_state(pool)로 커넥션 풀을 공유 상태로 등록
2. 핸들러에서 State(pool)로 풀을 꺼내서 사용
3. query_as로 DB 조회 → Vec<User> 반환
4. Json(users)로 JSON 응답 생성
```

**테스트:**
```bash
# 서버 실행 후
curl http://localhost:3000/users
```

응답 예시:
```json
[
  {"id": 1, "name": "홍길동", "email": "hong@example.com"},
  {"id": 2, "name": "김철수", "email": "kim@example.com"}
]
```

**포인트**:
- `with_state(pool)`: Axum의 상태 공유 기능으로 풀을 모든 핸들러에서 사용 가능
- `State(pool)`: 핸들러 파라미터에서 공유 상태 추출
- `PgPool`은 내부적으로 `Arc`를 사용하므로 clone 비용이 저렴합니다

</details>

---

## 8. 확인 문제

### 문제 1

SQLx에서 커넥션 풀을 생성하는 코드의 빈칸을 채우세요.

```rust
let pool = ______::new()
    .max_connections(5)
    .connect(&database_url)
    .await?;
```

<details>
<summary>정답 보기</summary>

**`PgPoolOptions`**

```rust
use sqlx::postgres::PgPoolOptions;

let pool = PgPoolOptions::new()
    .max_connections(5)
    .connect(&database_url)
    .await?;
```

`PgPoolOptions`는 PostgreSQL 커넥션 풀의 설정을 담당합니다. `max_connections`로 최대 동시 연결 수를 지정하고, `connect`로 실제 연결을 생성합니다.

</details>

### 문제 2

아래 코드에서 `$1`과 `$2`는 어떤 역할을 하나요?

```rust
sqlx::query("INSERT INTO users (name, email) VALUES ($1, $2)")
    .bind("홍길동")
    .bind("hong@example.com")
```

<details>
<summary>정답 보기</summary>

**바인드 파라미터(플레이스홀더)**입니다.

- `$1`에는 첫 번째 `.bind()` 값인 `"홍길동"`이 들어갑니다.
- `$2`에는 두 번째 `.bind()` 값인 `"hong@example.com"`이 들어갑니다.

바인드 파라미터를 사용하면 SQL 인젝션 공격을 방지할 수 있습니다. 사용자 입력을 직접 문자열에 넣지 않고, DB 드라이버가 안전하게 처리합니다.

</details>

### 문제 3

`fetch_one()`, `fetch_optional()`, `fetch_all()`의 차이점을 설명하세요.

<details>
<summary>정답 보기</summary>

| 메서드 | 반환 타입 | 결과 없을 때 |
|--------|----------|------------|
| `fetch_one()` | `T` | **에러 발생** |
| `fetch_optional()` | `Option<T>` | `None` 반환 |
| `fetch_all()` | `Vec<T>` | 빈 벡터 `[]` 반환 |

- `fetch_one`: 반드시 1행이 있어야 할 때 (예: ID로 조회하되 없으면 에러)
- `fetch_optional`: 있을 수도 없을 수도 있을 때 (예: 이메일로 검색)
- `fetch_all`: 여러 행을 모두 가져올 때 (예: 목록 조회)

</details>

### 문제 4

트랜잭션에서 `tx.commit()`을 호출하지 않고 함수가 종료되면 어떻게 되나요?

<details>
<summary>정답 보기</summary>

**자동으로 롤백(rollback)됩니다.**

SQLx의 트랜잭션 객체(`tx`)가 `commit()` 없이 드롭되면, `Drop` 트레이트 구현에 의해 자동으로 롤백이 실행됩니다. 트랜잭션 안에서 실행한 모든 쿼리가 취소되어, DB는 트랜잭션 시작 전 상태로 돌아갑니다.

```rust
let mut tx = pool.begin().await?;
sqlx::query("INSERT INTO ...").execute(&mut *tx).await?;
// commit 없이 함수 종료 → INSERT 취소됨
```

이 동작 덕분에 에러 발생 시 별도의 롤백 코드를 작성할 필요가 없습니다.

</details>

### 문제 5

아래 코드의 문제점은 무엇인가요?

```rust
let name = get_user_input();
let query = format!("SELECT * FROM users WHERE name = '{}'", name);
sqlx::query(&query).fetch_all(&pool).await?;
```

<details>
<summary>정답 보기</summary>

**SQL 인젝션 공격에 취약합니다.**

사용자가 `name`에 `'; DROP TABLE users; --` 같은 값을 입력하면, 실제 실행되는 SQL이 이렇게 됩니다:

```sql
SELECT * FROM users WHERE name = ''; DROP TABLE users; --'
```

`users` 테이블이 삭제될 수 있습니다.

**올바른 방법:**

```rust
let name = get_user_input();
sqlx::query_as::<_, User>("SELECT * FROM users WHERE name = $1")
    .bind(&name)
    .fetch_all(&pool)
    .await?;
```

바인드 파라미터(`$1`)를 사용하면 DB 드라이버가 입력값을 안전하게 이스케이프 처리합니다.

</details>

---

## 9. 20장 정리

| 배운 것 | 핵심 |
|---------|------|
| 프로젝트 설정 | `sqlx`, `tokio`, `dotenv` 의존성 추가, `.env`에 DATABASE_URL |
| DB 연결 | `PgPoolOptions::new().connect()` 로 커넥션 풀 생성 |
| 구조체 매핑 | `#[derive(sqlx::FromRow)]`로 쿼리 결과를 구조체에 매핑 |
| 쿼리 실행 | `query_as`로 타입 안전한 쿼리, `$1` 바인드로 인젝션 방지 |
| fetch 메서드 | `fetch_one`, `fetch_optional`, `fetch_all` 상황별 사용 |
| CRUD | INSERT / SELECT / UPDATE / DELETE 각각의 패턴 |
| 마이그레이션 | `sqlx-cli`로 DB 스키마를 코드로 관리 |
| 트랜잭션 | `pool.begin()` → 쿼리들 → `tx.commit()`, 실패 시 자동 롤백 |
| ACID | 원자성, 일관성, 격리성, 영속성 |
| Axum 연동 | `with_state(pool)`로 풀 공유, `State(pool)`로 핸들러에서 사용 |

---

## 다음 장 예고

> **21장. 프로젝트 구조 설계**에서는 실전 API 서버를 만들기 위한 아키텍처를 배웁니다.
> Handler(요청 처리) -> Service(비즈니스 로직) -> Repository(DB 접근)로 코드를 깔끔하게 분리하고,
> 설정 관리(dotenv, config)와 환경별 설정 분리까지 다룹니다.
> 이 장에서 배운 SQLx 코드가 Repository 레이어에 들어가게 됩니다!

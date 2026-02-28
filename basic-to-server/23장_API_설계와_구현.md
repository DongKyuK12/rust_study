# 23장. API 설계와 구현

> **목표**: RESTful API를 설계하고, 완전한 CRUD API를 구현한다.

> **드디어 실전!** 지금까지 배운 모든 것을 합쳐서 진짜 API 서버를 만듭니다.

---

## 1. RESTful API 설계 원칙

### REST란?

REST(Representational State Transfer)는 웹 API를 설계하는 **규칙**입니다. 전 세계 개발자들이 이 규칙을 따르기 때문에, REST를 따르면 **누구나 이해할 수 있는 API**가 됩니다.

### 핵심 원칙 1: 리소스 중심 URL

URL에는 **명사(리소스)**만 쓰고, **동사(행위)**는 HTTP 메서드로 표현합니다.

```
안 좋은 예시 (동사가 URL에 있음):
  GET    /getUsers
  POST   /createUser
  POST   /deleteUser/1
  GET    /fetchAllPosts

좋은 예시 (명사만 URL에 있음):
  GET    /users          -> 사용자 목록 조회
  POST   /users          -> 사용자 생성
  GET    /users/1        -> 사용자 1번 조회
  PUT    /users/1        -> 사용자 1번 수정
  DELETE /users/1        -> 사용자 1번 삭제
```

### 핵심 원칙 2: HTTP 메서드로 행위 표현

| HTTP 메서드 | 행위 | 예시 | 설명 |
|-------------|------|------|------|
| GET | 조회 | `GET /posts` | 데이터를 가져옴 (변경 없음) |
| POST | 생성 | `POST /posts` | 새 데이터를 만듦 |
| PUT | 전체 수정 | `PUT /posts/1` | 기존 데이터를 완전히 교체 |
| PATCH | 부분 수정 | `PATCH /posts/1` | 기존 데이터의 일부만 수정 |
| DELETE | 삭제 | `DELETE /posts/1` | 데이터를 삭제 |

### 핵심 원칙 3: 복수형 명사 사용

리소스 이름은 항상 **복수형**으로 씁니다. 하나를 조회할 때도 `/user/1`이 아니라 `/users/1`입니다.

```
안 좋은 예시:
  /user/1
  /post/5

좋은 예시:
  /users/1
  /posts/5
```

### 핵심 원칙 4: 중첩 리소스

리소스 사이에 **소속 관계**가 있으면 URL에 중첩으로 표현합니다.

```
사용자 1번의 게시글 목록:
  GET /users/1/posts

게시글 5번의 댓글 목록:
  GET /posts/5/comments

게시글 5번에 새 댓글 작성:
  POST /posts/5/comments
```

> 중첩은 **2단계까지만** 하는 것이 좋습니다. 3단계 이상은 URL이 너무 길어져서 읽기 어렵습니다. `/users/1/posts/5/comments/3/likes`처럼 길어지면 `/comments/3/likes`로 분리하세요.

### 핵심 원칙 5: 적절한 HTTP 상태 코드

| 상태 코드 | 의미 | 언제 쓰나 |
|-----------|------|-----------|
| 200 OK | 성공 | 조회, 수정 성공 |
| 201 Created | 생성됨 | POST로 새 리소스 생성 성공 |
| 204 No Content | 내용 없음 | 삭제 성공 (응답 본문 없음) |
| 400 Bad Request | 잘못된 요청 | 유효성 검사 실패 |
| 401 Unauthorized | 인증 필요 | 로그인 안 됨 |
| 403 Forbidden | 권한 없음 | 로그인은 했지만 권한 부족 |
| 404 Not Found | 찾을 수 없음 | 존재하지 않는 리소스 |
| 500 Internal Server Error | 서버 에러 | 서버 내부 오류 |

<details>
<summary><b>REST의 6가지 제약 조건 (원리)</b></summary>

### REST 아키텍처의 6가지 제약 조건

REST를 처음 정의한 Roy Fielding의 논문에서는 6가지 제약 조건을 제시합니다.

**1. Client-Server (클라이언트-서버 분리)**
클라이언트(프론트엔드)와 서버(백엔드)가 독립적으로 발전할 수 있어야 합니다.

**2. Stateless (무상태)**
서버가 클라이언트의 상태를 기억하지 않습니다. 모든 요청에 필요한 정보가 포함되어야 합니다. (그래서 JWT 토큰을 매 요청마다 보내는 것입니다.)

**3. Cacheable (캐시 가능)**
응답을 캐시할 수 있어야 합니다. `Cache-Control` 헤더 등을 사용합니다.

**4. Uniform Interface (통일된 인터페이스)**
URL, HTTP 메서드, 응답 형식이 일관되어야 합니다. 이것이 REST의 핵심입니다.

**5. Layered System (계층화)**
클라이언트는 서버와 직접 통신하는지, 중간에 로드밸런서나 프록시가 있는지 알 필요 없습니다.

**6. Code on Demand (선택적)**
서버가 클라이언트에 실행 코드를 보낼 수 있습니다. (JavaScript 같은 것) 유일한 선택 사항입니다.

실무에서 모든 제약 조건을 완벽하게 지키는 API는 드뭅니다. 핵심은 **통일된 인터페이스**와 **무상태**입니다. 이 두 가지만 잘 지켜도 좋은 API가 됩니다.

</details>

---

## 2. 완전한 CRUD API 구현

게시판(Post) API를 예시로, **레이어드 아키텍처**(Handler -> Service -> Repository)를 따라 전체 CRUD를 구현합니다.

### 프로젝트 의존성 (Cargo.toml)

```toml
[dependencies]
axum = "0.7"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
tokio = { version = "1", features = ["full"] }
sqlx = { version = "0.7", features = ["runtime-tokio-rustls", "postgres"] }
uuid = { version = "1", features = ["v4", "serde"] }
chrono = { version = "0.4", features = ["serde"] }
```

### 2-1. 모델 정의

데이터베이스 테이블과 대응하는 구조체를 정의합니다.

```rust
// src/models/post.rs

use chrono::NaiveDateTime;
use serde::{Deserialize, Serialize};
use sqlx::FromRow;
use uuid::Uuid;

/// 데이터베이스의 posts 테이블과 대응하는 구조체
#[derive(Debug, Clone, Serialize, Deserialize, FromRow)]
pub struct Post {
    pub id: Uuid,
    pub title: String,
    pub content: String,
    pub author_id: Uuid,
    pub created_at: NaiveDateTime,
    pub updated_at: NaiveDateTime,
}

/// 게시글 생성 요청
#[derive(Debug, Deserialize)]
pub struct CreatePostRequest {
    pub title: String,
    pub content: String,
}

/// 게시글 수정 요청
#[derive(Debug, Deserialize)]
pub struct UpdatePostRequest {
    pub title: Option<String>,
    pub content: Option<String>,
}

/// 게시글 응답 (클라이언트에게 보내는 형태)
#[derive(Debug, Serialize)]
pub struct PostResponse {
    pub id: Uuid,
    pub title: String,
    pub content: String,
    pub author_id: Uuid,
    pub created_at: String,
    pub updated_at: String,
}

impl From<Post> for PostResponse {
    fn from(post: Post) -> Self {
        Self {
            id: post.id,
            title: post.title,
            content: post.content,
            author_id: post.author_id,
            created_at: post.created_at.format("%Y-%m-%d %H:%M:%S").to_string(),
            updated_at: post.updated_at.format("%Y-%m-%d %H:%M:%S").to_string(),
        }
    }
}
```

### 2-2. Repository 레이어

데이터베이스와 직접 통신하는 계층입니다. **SQL 쿼리만 담당**합니다.

```rust
// src/repositories/post_repository.rs

use sqlx::PgPool;
use uuid::Uuid;
use crate::models::post::{Post, CreatePostRequest, UpdatePostRequest};

pub struct PostRepository {
    pool: PgPool,
}

impl PostRepository {
    pub fn new(pool: PgPool) -> Self {
        Self { pool }
    }

    /// 게시글 목록 조회
    pub async fn find_all(&self) -> Result<Vec<Post>, sqlx::Error> {
        let posts = sqlx::query_as::<_, Post>(
            "SELECT id, title, content, author_id, created_at, updated_at
             FROM posts
             ORDER BY created_at DESC"
        )
        .fetch_all(&self.pool)
        .await?;

        Ok(posts)
    }

    /// 게시글 단건 조회
    pub async fn find_by_id(&self, id: Uuid) -> Result<Option<Post>, sqlx::Error> {
        let post = sqlx::query_as::<_, Post>(
            "SELECT id, title, content, author_id, created_at, updated_at
             FROM posts
             WHERE id = $1"
        )
        .bind(id)
        .fetch_optional(&self.pool)
        .await?;

        Ok(post)
    }

    /// 게시글 생성
    pub async fn create(
        &self,
        req: &CreatePostRequest,
        author_id: Uuid,
    ) -> Result<Post, sqlx::Error> {
        let post = sqlx::query_as::<_, Post>(
            "INSERT INTO posts (id, title, content, author_id, created_at, updated_at)
             VALUES ($1, $2, $3, $4, NOW(), NOW())
             RETURNING id, title, content, author_id, created_at, updated_at"
        )
        .bind(Uuid::new_v4())
        .bind(&req.title)
        .bind(&req.content)
        .bind(author_id)
        .fetch_one(&self.pool)
        .await?;

        Ok(post)
    }

    /// 게시글 수정
    pub async fn update(
        &self,
        id: Uuid,
        req: &UpdatePostRequest,
    ) -> Result<Option<Post>, sqlx::Error> {
        let post = sqlx::query_as::<_, Post>(
            "UPDATE posts
             SET title = COALESCE($1, title),
                 content = COALESCE($2, content),
                 updated_at = NOW()
             WHERE id = $3
             RETURNING id, title, content, author_id, created_at, updated_at"
        )
        .bind(&req.title)
        .bind(&req.content)
        .bind(id)
        .fetch_optional(&self.pool)
        .await?;

        Ok(post)
    }

    /// 게시글 삭제
    pub async fn delete(&self, id: Uuid) -> Result<bool, sqlx::Error> {
        let result = sqlx::query("DELETE FROM posts WHERE id = $1")
            .bind(id)
            .execute(&self.pool)
            .await?;

        Ok(result.rows_affected() > 0)
    }
}
```

### 2-3. Service 레이어

**비즈니스 로직**을 담당합니다. Repository를 호출하고, 결과를 가공합니다.

```rust
// src/services/post_service.rs

use uuid::Uuid;
use crate::errors::AppError;
use crate::models::post::{CreatePostRequest, Post, UpdatePostRequest};
use crate::repositories::post_repository::PostRepository;

pub struct PostService {
    repository: PostRepository,
}

impl PostService {
    pub fn new(repository: PostRepository) -> Self {
        Self { repository }
    }

    pub async fn get_all_posts(&self) -> Result<Vec<Post>, AppError> {
        let posts = self.repository.find_all().await?;
        Ok(posts)
    }

    pub async fn get_post(&self, id: Uuid) -> Result<Post, AppError> {
        let post = self.repository.find_by_id(id).await?;

        // 얼리 리턴: 게시글이 없으면 바로 에러 반환
        let Some(post) = post else {
            return Err(AppError::NotFound(
                format!("게시글 {}을(를) 찾을 수 없습니다", id)
            ));
        };

        Ok(post)
    }

    pub async fn create_post(
        &self,
        req: CreatePostRequest,
        author_id: Uuid,
    ) -> Result<Post, AppError> {
        let post = self.repository.create(&req, author_id).await?;
        Ok(post)
    }

    pub async fn update_post(
        &self,
        id: Uuid,
        req: UpdatePostRequest,
    ) -> Result<Post, AppError> {
        let post = self.repository.update(id, &req).await?;

        let Some(post) = post else {
            return Err(AppError::NotFound(
                format!("게시글 {}을(를) 찾을 수 없습니다", id)
            ));
        };

        Ok(post)
    }

    pub async fn delete_post(&self, id: Uuid) -> Result<(), AppError> {
        let deleted = self.repository.delete(id).await?;

        if !deleted {
            return Err(AppError::NotFound(
                format!("게시글을 찾을 수 없습니다")
            ));
        }

        Ok(())
    }
}
```

### 2-4. Handler 레이어

HTTP 요청을 받아서 Service를 호출하고, HTTP 응답을 반환합니다.

```rust
// src/handlers/post_handler.rs

use axum::{
    extract::{Path, State},
    http::StatusCode,
    Json,
};
use uuid::Uuid;
use std::sync::Arc;
use crate::models::post::{CreatePostRequest, PostResponse, UpdatePostRequest};
use crate::services::post_service::PostService;

/// GET /posts - 게시글 목록 조회
pub async fn list_posts(
    State(service): State<Arc<PostService>>,
) -> Result<Json<Vec<PostResponse>>, StatusCode> {
    let posts = service.get_all_posts().await
        .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;

    let responses: Vec<PostResponse> = posts
        .into_iter()
        .map(PostResponse::from)
        .collect();

    Ok(Json(responses))
}

/// GET /posts/:id - 게시글 단건 조회
pub async fn get_post(
    State(service): State<Arc<PostService>>,
    Path(id): Path<Uuid>,
) -> Result<Json<PostResponse>, StatusCode> {
    let post = service.get_post(id).await
        .map_err(|_| StatusCode::NOT_FOUND)?;

    Ok(Json(PostResponse::from(post)))
}

/// POST /posts - 게시글 생성
pub async fn create_post(
    State(service): State<Arc<PostService>>,
    Json(req): Json<CreatePostRequest>,
) -> Result<(StatusCode, Json<PostResponse>), StatusCode> {
    // 실제로는 인증 미들웨어에서 author_id를 받아옴
    let author_id = Uuid::new_v4(); // 예시용 임시 값

    let post = service.create_post(req, author_id).await
        .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;

    Ok((StatusCode::CREATED, Json(PostResponse::from(post))))
}

/// PUT /posts/:id - 게시글 수정
pub async fn update_post(
    State(service): State<Arc<PostService>>,
    Path(id): Path<Uuid>,
    Json(req): Json<UpdatePostRequest>,
) -> Result<Json<PostResponse>, StatusCode> {
    let post = service.update_post(id, req).await
        .map_err(|_| StatusCode::NOT_FOUND)?;

    Ok(Json(PostResponse::from(post)))
}

/// DELETE /posts/:id - 게시글 삭제
pub async fn delete_post(
    State(service): State<Arc<PostService>>,
    Path(id): Path<Uuid>,
) -> Result<StatusCode, StatusCode> {
    service.delete_post(id).await
        .map_err(|_| StatusCode::NOT_FOUND)?;

    Ok(StatusCode::NO_CONTENT)
}
```

### 2-5. 라우터 연결

모든 핸들러를 라우터에 등록합니다.

```rust
// src/routes.rs

use axum::{
    routing::{get, post, put, delete},
    Router,
};
use std::sync::Arc;
use crate::handlers::post_handler;
use crate::services::post_service::PostService;

pub fn create_router(post_service: Arc<PostService>) -> Router {
    Router::new()
        .route("/posts", get(post_handler::list_posts))
        .route("/posts", post(post_handler::create_post))
        .route("/posts/:id", get(post_handler::get_post))
        .route("/posts/:id", put(post_handler::update_post))
        .route("/posts/:id", delete(post_handler::delete_post))
        .with_state(post_service)
}
```

### 2-6. main.rs

```rust
// src/main.rs

use sqlx::postgres::PgPoolOptions;
use std::sync::Arc;

mod errors;
mod handlers;
mod models;
mod repositories;
mod routes;
mod services;

use repositories::post_repository::PostRepository;
use services::post_service::PostService;

#[tokio::main]
async fn main() {
    // 데이터베이스 연결
    let database_url = std::env::var("DATABASE_URL")
        .expect("DATABASE_URL 환경 변수를 설정하세요");

    let pool = PgPoolOptions::new()
        .max_connections(5)
        .connect(&database_url)
        .await
        .expect("데이터베이스 연결 실패");

    // 레이어 조립
    let repository = PostRepository::new(pool);
    let service = Arc::new(PostService::new(repository));
    let app = routes::create_router(service);

    // 서버 시작
    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000")
        .await
        .expect("서버 시작 실패");

    println!("서버가 http://localhost:3000 에서 실행 중입니다");
    axum::serve(listener, app).await.expect("서버 실행 중 에러 발생");
}
```

### 전체 흐름 정리

```
클라이언트 요청
    ↓
[Router] POST /posts
    ↓
[Handler] create_post() - HTTP 요청 파싱, 응답 생성
    ↓
[Service] create_post() - 비즈니스 로직 (유효성 검사 등)
    ↓
[Repository] create()   - SQL 쿼리 실행
    ↓
[Database] INSERT INTO posts ...
    ↓
응답이 역순으로 올라와서 클라이언트에게 전달
```

> 각 레이어는 **자기 역할만** 합니다. Handler는 HTTP만 알고, Service는 비즈니스 규칙만 알고, Repository는 SQL만 압니다. 이렇게 분리하면 나중에 데이터베이스를 바꾸더라도 Repository만 수정하면 됩니다.

---

## 3. 요청 유효성 검사

사용자가 보낸 데이터를 **무조건 신뢰하면 안 됩니다**. 반드시 검증해야 합니다.

### validator 크레이트 설치

```toml
[dependencies]
validator = { version = "0.18", features = ["derive"] }
```

### 검증 규칙 추가

```rust
use serde::Deserialize;
use validator::Validate;

#[derive(Debug, Deserialize, Validate)]
pub struct CreatePostRequest {
    #[validate(length(min = 1, max = 100, message = "제목은 1~100자여야 합니다"))]
    pub title: String,

    #[validate(length(min = 1, max = 10000, message = "내용은 1~10000자여야 합니다"))]
    pub content: String,
}

#[derive(Debug, Deserialize, Validate)]
pub struct CreateUserRequest {
    #[validate(length(min = 2, max = 50, message = "이름은 2~50자여야 합니다"))]
    pub name: String,

    #[validate(email(message = "올바른 이메일 형식이 아닙니다"))]
    pub email: String,

    #[validate(length(min = 8, message = "비밀번호는 8자 이상이어야 합니다"))]
    pub password: String,
}
```

### 핸들러에서 검증 처리 (얼리 리턴)

```rust
use axum::{http::StatusCode, Json};
use serde_json::json;
use validator::Validate;

pub async fn create_post(
    State(service): State<Arc<PostService>>,
    Json(req): Json<CreatePostRequest>,
) -> Result<(StatusCode, Json<serde_json::Value>), (StatusCode, Json<serde_json::Value>)> {
    // 얼리 리턴: 유효성 검사 실패 시 즉시 400 에러 반환
    if let Err(errors) = req.validate() {
        return Err((
            StatusCode::BAD_REQUEST,
            Json(json!({
                "error": {
                    "code": "VALIDATION_ERROR",
                    "message": "입력값이 유효하지 않습니다",
                    "details": errors.to_string()
                }
            })),
        ));
    }

    // 검증 통과 -> 정상 처리
    let author_id = Uuid::new_v4();
    let post = service.create_post(req, author_id).await
        .map_err(|e| (
            StatusCode::INTERNAL_SERVER_ERROR,
            Json(json!({"error": {"code": "INTERNAL_ERROR", "message": e.to_string()}})),
        ))?;

    Ok((StatusCode::CREATED, Json(json!(PostResponse::from(post)))))
}
```

### 검증 에러 응답 예시

```json
{
    "error": {
        "code": "VALIDATION_ERROR",
        "message": "입력값이 유효하지 않습니다",
        "details": "title: 제목은 1~100자여야 합니다"
    }
}
```

> **원칙**: 검증은 **가능한 한 빨리** 실패시키세요. 데이터베이스까지 가기 전에 잘못된 요청을 걸러내야 합니다. 이것이 얼리 리턴의 핵심입니다.

---

## 4. 페이지네이션

게시글이 10만 개인데 한 번에 다 보내면? 서버도 클라이언트도 힘들어집니다. **페이지네이션**으로 나눠서 보내야 합니다.

### 쿼리 파라미터로 페이지 정보 받기

```rust
use serde::Deserialize;

#[derive(Debug, Deserialize)]
pub struct PaginationParams {
    pub page: Option<i64>,      // 기본값: 1
    pub per_page: Option<i64>,  // 기본값: 20
}

impl PaginationParams {
    pub fn page(&self) -> i64 {
        self.page.unwrap_or(1).max(1) // 최소 1페이지
    }

    pub fn per_page(&self) -> i64 {
        self.per_page.unwrap_or(20).clamp(1, 100) // 1~100 사이
    }

    pub fn offset(&self) -> i64 {
        (self.page() - 1) * self.per_page()
    }
}
```

### 응답에 메타데이터 포함

```rust
use serde::Serialize;

#[derive(Debug, Serialize)]
pub struct PaginatedResponse<T: Serialize> {
    pub data: Vec<T>,
    pub page: i64,
    pub per_page: i64,
    pub total: i64,
    pub total_pages: i64,
}

impl<T: Serialize> PaginatedResponse<T> {
    pub fn new(data: Vec<T>, page: i64, per_page: i64, total: i64) -> Self {
        let total_pages = (total as f64 / per_page as f64).ceil() as i64;
        Self {
            data,
            page,
            per_page,
            total,
            total_pages,
        }
    }
}
```

### Repository에 페이지네이션 적용

```rust
// src/repositories/post_repository.rs

impl PostRepository {
    /// 게시글 목록 조회 (페이지네이션)
    pub async fn find_all_paginated(
        &self,
        limit: i64,
        offset: i64,
    ) -> Result<Vec<Post>, sqlx::Error> {
        let posts = sqlx::query_as::<_, Post>(
            "SELECT id, title, content, author_id, created_at, updated_at
             FROM posts
             ORDER BY created_at DESC
             LIMIT $1 OFFSET $2"
        )
        .bind(limit)
        .bind(offset)
        .fetch_all(&self.pool)
        .await?;

        Ok(posts)
    }

    /// 전체 게시글 수 조회
    pub async fn count_all(&self) -> Result<i64, sqlx::Error> {
        let record = sqlx::query_scalar::<_, i64>(
            "SELECT COUNT(*) FROM posts"
        )
        .fetch_one(&self.pool)
        .await?;

        Ok(record)
    }
}
```

### Handler에서 사용

```rust
use axum::extract::Query;

/// GET /posts?page=1&per_page=20
pub async fn list_posts(
    State(service): State<Arc<PostService>>,
    Query(params): Query<PaginationParams>,
) -> Result<Json<PaginatedResponse<PostResponse>>, StatusCode> {
    let page = params.page();
    let per_page = params.per_page();

    let posts = service
        .get_posts_paginated(per_page, params.offset())
        .await
        .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;

    let total = service
        .count_posts()
        .await
        .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;

    let responses: Vec<PostResponse> = posts
        .into_iter()
        .map(PostResponse::from)
        .collect();

    Ok(Json(PaginatedResponse::new(responses, page, per_page, total)))
}
```

### 실제 응답 예시

```
GET /posts?page=2&per_page=3
```

```json
{
    "data": [
        {
            "id": "550e8400-e29b-41d4-a716-446655440004",
            "title": "네 번째 게시글",
            "content": "내용입니다",
            "author_id": "...",
            "created_at": "2026-02-14 10:00:00",
            "updated_at": "2026-02-14 10:00:00"
        },
        {
            "id": "550e8400-e29b-41d4-a716-446655440005",
            "title": "다섯 번째 게시글",
            "content": "내용입니다",
            "author_id": "...",
            "created_at": "2026-02-13 09:00:00",
            "updated_at": "2026-02-13 09:00:00"
        },
        {
            "id": "550e8400-e29b-41d4-a716-446655440006",
            "title": "여섯 번째 게시글",
            "content": "내용입니다",
            "author_id": "...",
            "created_at": "2026-02-12 08:00:00",
            "updated_at": "2026-02-12 08:00:00"
        }
    ],
    "page": 2,
    "per_page": 3,
    "total": 15,
    "total_pages": 5
}
```

<details>
<summary><b>커서 기반 페이지네이션 vs 오프셋 기반 (원리)</b></summary>

### 오프셋 기반 (OFFSET/LIMIT)

위에서 구현한 방식입니다. 간단하지만 단점이 있습니다.

```sql
-- 100만 번째부터 20개 가져오기
SELECT * FROM posts ORDER BY created_at DESC LIMIT 20 OFFSET 1000000;
-- 데이터베이스가 100만 개를 건너뛰어야 해서 느림!
```

**장점**: 구현이 간단함, "3페이지로 이동"이 가능
**단점**: 데이터가 많을수록 느림, 새 데이터 삽입 시 중복/누락 가능

### 커서 기반 (Cursor-based)

마지막으로 본 항목의 ID를 기준으로 다음 데이터를 가져옵니다.

```sql
-- 마지막으로 본 게시글의 created_at 이후 20개
SELECT * FROM posts
WHERE created_at < '2026-02-10 12:00:00'
ORDER BY created_at DESC
LIMIT 20;
```

**장점**: 데이터가 아무리 많아도 빠름, 중복/누락 없음
**단점**: "N페이지로 이동"이 불가능, 구현이 약간 복잡

**결론**: 데이터가 적으면 오프셋, 많으면 커서 기반을 쓰세요. 이 장에서는 학습 목적으로 오프셋 기반을 사용합니다.

</details>

---

## 5. 에러 응답 표준화

API에서 에러가 발생하면, **항상 같은 형식**으로 응답해야 합니다. 어떤 API는 문자열을 보내고, 어떤 API는 JSON을 보내면 클라이언트가 혼란스럽습니다.

### 통일된 에러 응답 형식

```json
{
    "error": {
        "code": "NOT_FOUND",
        "message": "게시글을 찾을 수 없습니다"
    }
}
```

### AppError enum 정의

```rust
// src/errors.rs

use axum::{
    http::StatusCode,
    response::{IntoResponse, Response},
    Json,
};
use serde_json::json;

#[derive(Debug)]
pub enum AppError {
    NotFound(String),
    BadRequest(String),
    Unauthorized(String),
    Forbidden(String),
    InternalError(String),
}

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        let (status, code, message) = match self {
            AppError::NotFound(msg) => (
                StatusCode::NOT_FOUND,
                "NOT_FOUND",
                msg,
            ),
            AppError::BadRequest(msg) => (
                StatusCode::BAD_REQUEST,
                "BAD_REQUEST",
                msg,
            ),
            AppError::Unauthorized(msg) => (
                StatusCode::UNAUTHORIZED,
                "UNAUTHORIZED",
                msg,
            ),
            AppError::Forbidden(msg) => (
                StatusCode::FORBIDDEN,
                "FORBIDDEN",
                msg,
            ),
            AppError::InternalError(msg) => (
                StatusCode::INTERNAL_SERVER_ERROR,
                "INTERNAL_ERROR",
                msg,
            ),
        };

        let body = json!({
            "error": {
                "code": code,
                "message": message
            }
        });

        (status, Json(body)).into_response()
    }
}

// sqlx 에러를 AppError로 자동 변환
impl From<sqlx::Error> for AppError {
    fn from(err: sqlx::Error) -> Self {
        AppError::InternalError(format!("데이터베이스 에러: {}", err))
    }
}
```

### Handler에서 AppError 활용

AppError가 `IntoResponse`를 구현했으므로, Handler에서 깔끔하게 사용할 수 있습니다.

```rust
use crate::errors::AppError;

/// GET /posts/:id
pub async fn get_post(
    State(service): State<Arc<PostService>>,
    Path(id): Path<Uuid>,
) -> Result<Json<PostResponse>, AppError> {
    // Service에서 AppError를 반환하므로 ?로 바로 전파 가능
    let post = service.get_post(id).await?;
    Ok(Json(PostResponse::from(post)))
}

/// POST /posts
pub async fn create_post(
    State(service): State<Arc<PostService>>,
    Json(req): Json<CreatePostRequest>,
) -> Result<(StatusCode, Json<PostResponse>), AppError> {
    // 얼리 리턴: 유효성 검사
    if let Err(errors) = req.validate() {
        return Err(AppError::BadRequest(
            format!("유효성 검사 실패: {}", errors)
        ));
    }

    let author_id = Uuid::new_v4();
    let post = service.create_post(req, author_id).await?;
    Ok((StatusCode::CREATED, Json(PostResponse::from(post))))
}
```

이전 코드와 비교하면 **훨씬 깔끔**합니다. `map_err`로 매번 에러를 변환할 필요가 없어졌습니다.

### 에러 코드와 상태 코드 대응표

| AppError 종류 | HTTP 상태 코드 | 에러 코드 | 예시 상황 |
|---------------|---------------|-----------|-----------|
| NotFound | 404 | NOT_FOUND | 존재하지 않는 게시글 조회 |
| BadRequest | 400 | BAD_REQUEST | 유효성 검사 실패, 잘못된 JSON |
| Unauthorized | 401 | UNAUTHORIZED | 로그인 필요, 토큰 만료 |
| Forbidden | 403 | FORBIDDEN | 다른 사용자의 게시글 삭제 시도 |
| InternalError | 500 | INTERNAL_ERROR | DB 연결 실패, 예상치 못한 에러 |

<details>
<summary><b>에러 처리 패턴: thiserror 크레이트 (원리)</b></summary>

### thiserror로 더 깔끔하게

`thiserror` 크레이트를 사용하면 에러 타입을 더 선언적으로 정의할 수 있습니다.

```toml
[dependencies]
thiserror = "1"
```

```rust
use thiserror::Error;

#[derive(Debug, Error)]
pub enum AppError {
    #[error("찾을 수 없습니다: {0}")]
    NotFound(String),

    #[error("잘못된 요청: {0}")]
    BadRequest(String),

    #[error("인증이 필요합니다: {0}")]
    Unauthorized(String),

    #[error("권한이 없습니다: {0}")]
    Forbidden(String),

    #[error("서버 내부 에러: {0}")]
    InternalError(String),

    #[error("데이터베이스 에러: {0}")]
    Database(#[from] sqlx::Error),
}
```

`#[from]` 속성을 쓰면 `From` 트레이트 구현을 자동 생성합니다. `impl From<sqlx::Error> for AppError`를 직접 쓸 필요가 없어집니다.

`#[error("...")]`는 `Display` 트레이트를 자동 구현합니다. `err.to_string()`을 호출하면 지정한 메시지가 나옵니다.

</details>

---

## 6. API 문서화

API를 만들었으면, 다른 개발자(또는 프론트엔드 팀)가 **어떻게 사용하는지** 알려줘야 합니다.

### utoipa로 OpenAPI 문서 자동 생성

`utoipa` 크레이트를 사용하면 코드에서 **자동으로 API 문서**를 생성할 수 있습니다.

```toml
[dependencies]
utoipa = { version = "4", features = ["axum_extras"] }
utoipa-swagger-ui = { version = "7", features = ["axum"] }
```

```rust
use utoipa::ToSchema;
use utoipa::OpenApi;

/// 게시글 생성 요청
#[derive(Debug, Deserialize, Validate, ToSchema)]
pub struct CreatePostRequest {
    /// 게시글 제목 (1~100자)
    #[validate(length(min = 1, max = 100))]
    pub title: String,
    /// 게시글 내용 (1~10000자)
    #[validate(length(min = 1, max = 10000))]
    pub content: String,
}

#[derive(OpenApi)]
#[openapi(
    paths(
        post_handler::list_posts,
        post_handler::get_post,
        post_handler::create_post,
    ),
    components(schemas(CreatePostRequest, PostResponse))
)]
struct ApiDoc;
```

이렇게 하면 `/swagger-ui` 경로에서 API 문서를 브라우저로 확인할 수 있습니다.

### 수동 문서 작성 팁

자동 생성이 어렵다면, 최소한 다음 정보는 문서로 남기세요.

```markdown
## POST /posts

게시글을 생성합니다.

### 요청 헤더
| 헤더 | 값 | 필수 |
|------|---|------|
| Authorization | Bearer {token} | O |
| Content-Type | application/json | O |

### 요청 본문
| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| title | string | O | 1~100자 |
| content | string | O | 1~10000자 |

### 응답 (201 Created)
{
    "id": "uuid",
    "title": "제목",
    "content": "내용",
    "author_id": "uuid",
    "created_at": "2026-02-16 12:00:00",
    "updated_at": "2026-02-16 12:00:00"
}

### 에러 응답
- 400: 유효성 검사 실패
- 401: 인증 필요
```

> **팁**: API 문서는 코드와 함께 관리하세요. 코드는 바뀌었는데 문서가 안 바뀌면 더 혼란스럽습니다. utoipa 같은 도구를 쓰면 코드에서 문서가 자동으로 나오기 때문에 이 문제를 방지할 수 있습니다.

---

## 7. 실습

### 실습 1: 게시판 CRUD API 전체 구현

이 장에서 배운 모든 레이어를 조합하여, 게시판 CRUD API를 직접 구현하세요.

```
요구사항:
1. 모델: Post (id, title, content, author_id, created_at, updated_at)
2. 다음 5개의 API를 구현하세요:
   - GET    /posts       -> 게시글 목록 조회
   - POST   /posts       -> 게시글 생성
   - GET    /posts/:id   -> 게시글 단건 조회
   - PUT    /posts/:id   -> 게시글 수정
   - DELETE /posts/:id   -> 게시글 삭제
3. 레이어: Handler -> Service -> Repository
4. 응답은 JSON 형식

테스트 방법 (curl):
  curl http://localhost:3000/posts
  curl -X POST http://localhost:3000/posts \
    -H "Content-Type: application/json" \
    -d '{"title": "첫 게시글", "content": "안녕하세요!"}'
```

<details>
<summary><b>정답 보기</b></summary>

이 장의 섹션 2에서 제시한 코드 전체가 정답입니다. 파일 구조는 다음과 같습니다.

```
src/
├── main.rs
├── routes.rs
├── errors.rs
├── models/
│   ├── mod.rs
│   └── post.rs
├── handlers/
│   ├── mod.rs
│   └── post_handler.rs
├── services/
│   ├── mod.rs
│   └── post_service.rs
└── repositories/
    ├── mod.rs
    └── post_repository.rs
```

각 `mod.rs` 파일에는 모듈을 공개합니다.

```rust
// src/models/mod.rs
pub mod post;

// src/handlers/mod.rs
pub mod post_handler;

// src/services/mod.rs
pub mod post_service;

// src/repositories/mod.rs
pub mod post_repository;
```

핵심은 각 레이어가 **자기 역할에만 집중**하는 것입니다.
- Handler: HTTP 요청/응답만 처리
- Service: 비즈니스 로직만 처리
- Repository: 데이터베이스 쿼리만 처리

</details>

### 실습 2: 유효성 검사 추가

실습 1의 코드에 `validator` 크레이트를 사용한 유효성 검사를 추가하세요.

```
요구사항:
1. CreatePostRequest에 검증 규칙 추가:
   - title: 1~100자 필수
   - content: 1~10000자 필수
2. 검증 실패 시 400 에러 응답 (JSON 형식)
3. 얼리 리턴 패턴 사용

테스트 방법 (빈 제목으로 요청):
  curl -X POST http://localhost:3000/posts \
    -H "Content-Type: application/json" \
    -d '{"title": "", "content": "내용"}'

예상 응답 (400 Bad Request):
  {
      "error": {
          "code": "VALIDATION_ERROR",
          "message": "입력값이 유효하지 않습니다",
          "details": "title: 제목은 1~100자여야 합니다"
      }
  }
```

<details>
<summary><b>정답 보기</b></summary>

```rust
// src/models/post.rs - CreatePostRequest에 검증 추가
use validator::Validate;

#[derive(Debug, Deserialize, Validate)]
pub struct CreatePostRequest {
    #[validate(length(min = 1, max = 100, message = "제목은 1~100자여야 합니다"))]
    pub title: String,

    #[validate(length(min = 1, max = 10000, message = "내용은 1~10000자여야 합니다"))]
    pub content: String,
}
```

```rust
// src/handlers/post_handler.rs - 핸들러에 검증 로직 추가
use validator::Validate;
use crate::errors::AppError;

pub async fn create_post(
    State(service): State<Arc<PostService>>,
    Json(req): Json<CreatePostRequest>,
) -> Result<(StatusCode, Json<PostResponse>), AppError> {
    // 얼리 리턴: 유효성 검사 실패 시 즉시 에러 반환
    if let Err(errors) = req.validate() {
        return Err(AppError::BadRequest(
            format!("유효성 검사 실패: {}", errors)
        ));
    }

    let author_id = Uuid::new_v4();
    let post = service.create_post(req, author_id).await?;
    Ok((StatusCode::CREATED, Json(PostResponse::from(post))))
}
```

검증 로직이 핸들러 **맨 처음**에 위치하는 것이 포인트입니다. 실패하면 Service까지 가지도 않고 바로 에러를 돌려보냅니다.

</details>

### 실습 3: 페이지네이션 적용

게시글 목록 API에 페이지네이션을 적용하세요.

```
요구사항:
1. GET /posts?page=1&per_page=20 형태로 쿼리 파라미터 지원
2. 기본값: page=1, per_page=20
3. per_page 최대값: 100
4. 응답에 메타데이터 포함 (page, per_page, total, total_pages)

테스트 방법:
  curl "http://localhost:3000/posts?page=2&per_page=5"

예상 응답:
  {
      "data": [...],
      "page": 2,
      "per_page": 5,
      "total": 23,
      "total_pages": 5
  }
```

<details>
<summary><b>정답 보기</b></summary>

```rust
// src/models/pagination.rs
use serde::{Deserialize, Serialize};

#[derive(Debug, Deserialize)]
pub struct PaginationParams {
    pub page: Option<i64>,
    pub per_page: Option<i64>,
}

impl PaginationParams {
    pub fn page(&self) -> i64 {
        self.page.unwrap_or(1).max(1)
    }

    pub fn per_page(&self) -> i64 {
        self.per_page.unwrap_or(20).clamp(1, 100)
    }

    pub fn offset(&self) -> i64 {
        (self.page() - 1) * self.per_page()
    }
}

#[derive(Debug, Serialize)]
pub struct PaginatedResponse<T: Serialize> {
    pub data: Vec<T>,
    pub page: i64,
    pub per_page: i64,
    pub total: i64,
    pub total_pages: i64,
}

impl<T: Serialize> PaginatedResponse<T> {
    pub fn new(data: Vec<T>, page: i64, per_page: i64, total: i64) -> Self {
        let total_pages = if per_page > 0 {
            (total as f64 / per_page as f64).ceil() as i64
        } else {
            0
        };
        Self { data, page, per_page, total, total_pages }
    }
}
```

```rust
// src/repositories/post_repository.rs - 페이지네이션 쿼리 추가
impl PostRepository {
    pub async fn find_all_paginated(
        &self,
        limit: i64,
        offset: i64,
    ) -> Result<Vec<Post>, sqlx::Error> {
        let posts = sqlx::query_as::<_, Post>(
            "SELECT id, title, content, author_id, created_at, updated_at
             FROM posts ORDER BY created_at DESC LIMIT $1 OFFSET $2"
        )
        .bind(limit)
        .bind(offset)
        .fetch_all(&self.pool)
        .await?;

        Ok(posts)
    }

    pub async fn count_all(&self) -> Result<i64, sqlx::Error> {
        let count = sqlx::query_scalar::<_, i64>("SELECT COUNT(*) FROM posts")
            .fetch_one(&self.pool)
            .await?;

        Ok(count)
    }
}
```

```rust
// src/handlers/post_handler.rs - 핸들러 수정
use axum::extract::Query;
use crate::models::pagination::{PaginatedResponse, PaginationParams};

pub async fn list_posts(
    State(service): State<Arc<PostService>>,
    Query(params): Query<PaginationParams>,
) -> Result<Json<PaginatedResponse<PostResponse>>, AppError> {
    let page = params.page();
    let per_page = params.per_page();

    let posts = service
        .get_posts_paginated(per_page, params.offset())
        .await?;

    let total = service.count_posts().await?;

    let responses: Vec<PostResponse> = posts
        .into_iter()
        .map(PostResponse::from)
        .collect();

    Ok(Json(PaginatedResponse::new(responses, page, per_page, total)))
}
```

`PaginationParams`가 기본값 처리와 범위 제한을 캡슐화하고 있어서, 핸들러가 깔끔합니다. 다른 리소스(사용자, 댓글 등)에도 같은 `PaginationParams`와 `PaginatedResponse`를 재사용할 수 있습니다.

</details>

---

## 8. 확인 문제

### 문제 1
다음 URL 설계 중 RESTful하지 않은 것을 모두 고르세요.
- (a) `GET /users`
- (b) `POST /createUser`
- (c) `DELETE /users/1`
- (d) `GET /getAllPosts`
- (e) `PUT /users/1`

### 문제 2
레이어드 아키텍처에서 Handler, Service, Repository 각각의 역할은 무엇인가요?

### 문제 3
다음 코드의 문제점은 무엇인가요?

```rust
pub async fn create_post(
    State(service): State<Arc<PostService>>,
    Json(req): Json<CreatePostRequest>,
) -> Result<Json<PostResponse>, AppError> {
    let author_id = Uuid::new_v4();
    let post = service.create_post(req, author_id).await?;
    if let Err(errors) = req.validate() {
        return Err(AppError::BadRequest(errors.to_string()));
    }
    Ok(Json(PostResponse::from(post)))
}
```

### 문제 4
`PaginationParams`에서 `per_page`의 기본값이 20이고, 최대값이 100입니다. 사용자가 `?per_page=500`을 보내면 어떻게 되어야 하나요?

### 문제 5
다음 중 HTTP 상태 코드와 상황의 연결이 **잘못된 것**은?
- (a) 200: 게시글 조회 성공
- (b) 201: 게시글 생성 성공
- (c) 204: 게시글 삭제 성공
- (d) 401: 존재하지 않는 게시글 조회
- (e) 400: 유효성 검사 실패

---

## 확인 문제 정답

<details>
<summary>정답 보기 (먼저 풀어본 후 클릭하세요!)</summary>

### 문제 1 정답
**(b) `POST /createUser`** 와 **(d) `GET /getAllPosts`**

URL에 동사(`create`, `getAll`)가 포함되어 있습니다. RESTful API에서는 행위를 HTTP 메서드로 표현하고, URL에는 리소스 이름(명사)만 써야 합니다.
- `POST /createUser` -> `POST /users`
- `GET /getAllPosts` -> `GET /posts`

### 문제 2 정답

| 레이어 | 역할 | 담당하는 것 |
|--------|------|------------|
| Handler | HTTP 처리 | 요청 파싱, 응답 생성, 상태 코드 |
| Service | 비즈니스 로직 | 유효성 검사, 데이터 가공, 규칙 적용 |
| Repository | 데이터 접근 | SQL 쿼리, 데이터베이스 통신 |

각 레이어는 자기 역할만 하고, 아래 레이어만 호출합니다. Handler -> Service -> Repository 순서입니다.

### 문제 3 정답
**유효성 검사가 비즈니스 로직 실행 이후에 있습니다.** `service.create_post()`를 먼저 호출한 뒤에 `req.validate()`를 하고 있어서, 이미 잘못된 데이터가 데이터베이스에 저장된 후에야 검증이 실행됩니다. 또한 `req`는 `service.create_post(req, ...)`에서 이미 소유권이 이동했으므로 컴파일 에러가 발생합니다.

올바른 코드는 유효성 검사를 **맨 처음**에 얼리 리턴으로 처리해야 합니다:

```rust
pub async fn create_post(
    State(service): State<Arc<PostService>>,
    Json(req): Json<CreatePostRequest>,
) -> Result<Json<PostResponse>, AppError> {
    // 먼저 검증!
    if let Err(errors) = req.validate() {
        return Err(AppError::BadRequest(errors.to_string()));
    }

    let author_id = Uuid::new_v4();
    let post = service.create_post(req, author_id).await?;
    Ok(Json(PostResponse::from(post)))
}
```

### 문제 4 정답
`per_page`가 **100으로 제한**됩니다. `clamp(1, 100)` 메서드가 값을 1~100 범위로 자르기 때문입니다. 500을 보내도 100으로 처리됩니다. 이렇게 서버가 최대값을 제한해야 악의적인 요청(`?per_page=999999`)으로 서버에 부담을 주는 것을 방지할 수 있습니다.

### 문제 5 정답
**(d) 401: 존재하지 않는 게시글 조회**

존재하지 않는 리소스를 조회할 때는 **404 Not Found**가 맞습니다. 401 Unauthorized는 **인증이 필요한데 인증하지 않았을 때** 사용합니다. (로그인 안 한 상태에서 접근 등)

</details>

---

## 9. 23장 정리

| 배운 것 | 핵심 |
|---------|------|
| RESTful 설계 | URL에는 명사, 행위는 HTTP 메서드, 복수형 사용 |
| CRUD API | GET(조회), POST(생성), PUT(수정), DELETE(삭제) |
| 레이어드 아키텍처 | Handler -> Service -> Repository, 각자 역할만 |
| 유효성 검사 | validator 크레이트, 얼리 리턴으로 빠르게 실패 |
| 페이지네이션 | page/per_page 쿼리, LIMIT/OFFSET, 메타데이터 포함 |
| 에러 응답 표준화 | AppError enum, 통일된 JSON 형식, 적절한 상태 코드 |
| API 문서화 | utoipa로 OpenAPI 자동 생성, 코드와 문서 함께 관리 |

---

## 다음 장 예고

> **24장. 테스트**에서는 우리가 만든 API가 **정말로 올바르게 동작하는지** 검증하는 법을 배웁니다.
> 단위 테스트로 각 함수를 개별 검증하고, 통합 테스트로 레이어 간 연동을 확인하고, API 테스트로 실제 HTTP 요청/응답을 테스트합니다. 테스트 데이터베이스를 따로 설정해서 실제 데이터에 영향 없이 안전하게 테스트하는 법도 알아봅니다!

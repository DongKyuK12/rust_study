# 부록 A. 자주 쓰는 Cargo 명령어 모음

> **이 부록은 빠르게 찾아보는 치트시트입니다. 필요할 때마다 돌아와서 확인하세요!**

---

## 1. 프로젝트 관리

| 명령어 | 설명 | 예시 |
|--------|------|------|
| `cargo new 이름` | 새 프로젝트 생성 (폴더 + 기본 파일) | `cargo new my_server` |
| `cargo new 이름 --lib` | 라이브러리 프로젝트 생성 | `cargo new my_lib --lib` |
| `cargo init` | 현재 폴더를 Cargo 프로젝트로 초기화 | `cargo init` |
| `cargo clean` | 빌드 결과물(target 폴더) 삭제 | `cargo clean` |

```bash
# 새 웹서버 프로젝트 만들기
cargo new my_web_server
cd my_web_server
```

`cargo new`를 실행하면 다음 구조가 만들어집니다:

```
my_web_server/
├── Cargo.toml    # 프로젝트 설정 파일
└── src/
    └── main.rs   # 코드 시작점
```

---

## 2. 빌드와 실행

| 명령어 | 설명 | 예시 |
|--------|------|------|
| `cargo build` | 프로젝트 빌드 (디버그 모드) | `cargo build` |
| `cargo build --release` | 최적화된 릴리즈 빌드 | `cargo build --release` |
| `cargo run` | 빌드 + 실행을 한 번에 | `cargo run` |
| `cargo run --release` | 릴리즈 모드로 빌드 + 실행 | `cargo run --release` |
| `cargo check` | 컴파일 가능한지만 확인 (빠름) | `cargo check` |

### 디버그 빌드 vs 릴리즈 빌드

```bash
# 개발 중에는 이렇게 (빌드 빠름, 실행 느림)
cargo build
cargo run

# 배포할 때는 이렇게 (빌드 느림, 실행 빠름)
cargo build --release
cargo run --release
```

- **디버그 빌드**: `target/debug/` 폴더에 생성됩니다. 빌드가 빠르지만 실행 속도가 느립니다.
- **릴리즈 빌드**: `target/release/` 폴더에 생성됩니다. 빌드가 느리지만 실행 속도가 빠릅니다.

### cargo check를 적극 활용하세요

`cargo check`는 실제 실행 파일을 만들지 않고 코드가 올바른지만 확인합니다. `cargo build`보다 훨씬 빠르기 때문에, 코드를 수정하면서 자주 확인할 때 유용합니다.

```bash
# 코드 수정 후 빠르게 에러 확인
cargo check

# 에러가 없으면 그때 실행
cargo run
```

---

## 3. 의존성 관리

| 명령어 | 설명 | 예시 |
|--------|------|------|
| `cargo add 크레이트` | 의존성 추가 | `cargo add serde` |
| `cargo add 크레이트 --features 기능` | 특정 기능 포함하여 추가 | `cargo add serde --features derive` |
| `cargo remove 크레이트` | 의존성 제거 | `cargo remove serde` |
| `cargo update` | 모든 의존성을 최신 버전으로 갱신 | `cargo update` |
| `cargo tree` | 의존성 트리 확인 | `cargo tree` |

### 웹서버 프로젝트 의존성 추가 예시

```bash
# Axum 웹 프레임워크 추가
cargo add axum

# Tokio 비동기 런타임 추가 (전체 기능 포함)
cargo add tokio --features full

# Serde JSON 직렬화 추가
cargo add serde --features derive
cargo add serde_json

# 어떤 크레이트가 설치되어 있는지 트리로 확인
cargo tree
```

`cargo add`를 실행하면 `Cargo.toml` 파일의 `[dependencies]` 섹션에 자동으로 추가됩니다:

```toml
[dependencies]
axum = "0.8"
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
```

---

## 4. 테스트와 문서

| 명령어 | 설명 | 예시 |
|--------|------|------|
| `cargo test` | 모든 테스트 실행 | `cargo test` |
| `cargo test 이름` | 이름이 포함된 테스트만 실행 | `cargo test user` |
| `cargo test -- --nocapture` | 테스트 중 println! 출력 보기 | `cargo test -- --nocapture` |
| `cargo doc` | 문서 생성 | `cargo doc` |
| `cargo doc --open` | 문서 생성 후 브라우저에서 열기 | `cargo doc --open` |
| `cargo bench` | 벤치마크 테스트 실행 | `cargo bench` |

```bash
# 모든 테스트 실행
cargo test

# "login" 이 이름에 포함된 테스트만 실행
cargo test login

# 테스트에서 println! 출력이 보고 싶을 때
cargo test -- --nocapture

# 내 코드와 사용 중인 크레이트의 문서를 브라우저에서 확인
cargo doc --open
```

---

## 5. 배포

| 명령어 | 설명 | 예시 |
|--------|------|------|
| `cargo publish` | crates.io에 크레이트 배포 | `cargo publish` |
| `cargo install 크레이트` | 크레이트를 실행 파일로 설치 | `cargo install cargo-watch` |
| `cargo uninstall 크레이트` | 설치한 크레이트 제거 | `cargo uninstall cargo-watch` |

```bash
# 유용한 도구 설치 예시
cargo install cargo-watch    # 파일 변경 시 자동 재빌드
cargo install cargo-expand   # 매크로 전개 결과 확인
```

---

## 6. 유용한 옵션들

| 옵션 | 설명 | 사용 예시 |
|------|------|-----------|
| `--release` | 최적화된 릴리즈 모드로 빌드 | `cargo build --release` |
| `--verbose` 또는 `-v` | 상세한 출력 보기 | `cargo build --verbose` |
| `--quiet` 또는 `-q` | 출력 최소화 | `cargo build --quiet` |
| `--features 기능` | 특정 기능(feature) 활성화 | `cargo build --features "full"` |
| `--no-default-features` | 기본 기능 비활성화 | `cargo add tokio --no-default-features` |
| `--all-targets` | 모든 타겟 빌드 | `cargo check --all-targets` |

---

## 7. 개발 중 자주 쓰는 조합

### 기본 개발 흐름

```bash
# 1. 프로젝트 생성
cargo new my_project
cd my_project

# 2. 의존성 추가
cargo add serde --features derive
cargo add serde_json

# 3. 코드 작성 후 확인
cargo check

# 4. 실행
cargo run

# 5. 테스트
cargo test
```

### cargo-watch로 자동 재실행

`cargo-watch`를 설치하면 파일을 저장할 때마다 자동으로 다시 빌드하고 실행합니다.

```bash
# 설치 (한 번만)
cargo install cargo-watch

# 파일 변경 시 자동으로 cargo run 실행
cargo watch -x run

# 파일 변경 시 자동으로 cargo check 실행
cargo watch -x check

# 파일 변경 시 자동으로 cargo test 실행
cargo watch -x test
```

---

## 한눈에 보기

```
프로젝트 시작     cargo new / cargo init
코드 확인        cargo check          (빠른 문법 확인)
코드 실행        cargo run            (빌드 + 실행)
테스트 실행      cargo test           (테스트 코드 실행)
의존성 추가      cargo add 크레이트     (Cargo.toml에 자동 추가)
의존성 확인      cargo tree           (설치된 크레이트 트리 확인)
문서 확인        cargo doc --open     (브라우저에서 문서 열기)
릴리즈 빌드      cargo build --release (최적화된 빌드)
정리             cargo clean          (빌드 결과물 삭제)
```

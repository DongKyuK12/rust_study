# 부록 D. 참고 자료 및 커뮤니티

> **Rust를 더 깊이 배우고 싶을 때 참고할 수 있는 자료와 커뮤니티를 정리했습니다.**

---

## 1. 공식 자료

Rust 공식 팀에서 제공하는 무료 자료입니다. 가장 정확하고 최신 정보를 담고 있습니다.

| 자료 | 설명 | 링크 |
|------|------|------|
| The Rust Programming Language | Rust 공식 입문서 ("The Book"이라고 불림) | https://doc.rust-lang.org/book/ |
| Rust by Example | 예제 중심의 Rust 학습 자료 | https://doc.rust-lang.org/rust-by-example/ |
| Rust Reference | Rust 언어 사양 (문법 세부사항) | https://doc.rust-lang.org/reference/ |
| Rust Standard Library | 표준 라이브러리 API 문서 | https://doc.rust-lang.org/std/ |
| The Cargo Book | Cargo 사용법 공식 문서 | https://doc.rust-lang.org/cargo/ |
| Rustonomicon | 안전하지 않은(unsafe) Rust 가이드 (고급) | https://doc.rust-lang.org/nomicon/ |

### 추천 순서

1. 이 학습지를 먼저 완료하세요.
2. The Rust Programming Language (The Book)으로 빠진 부분을 보충하세요.
3. Rust by Example로 다양한 예제를 연습하세요.
4. 표준 라이브러리 문서를 참고하며 실전 프로젝트를 진행하세요.

---

## 2. 온라인 학습

### 실습 중심 학습

| 자료 | 설명 | 링크 |
|------|------|------|
| Rustlings | 작은 연습 문제를 풀면서 Rust 배우기 (강력 추천) | https://github.com/rust-lang/rustlings |
| Exercism Rust Track | 멘토링이 가능한 Rust 연습 문제 | https://exercism.org/tracks/rust |
| Rust Playground | 설치 없이 브라우저에서 Rust 코드 실행 | https://play.rust-lang.org/ |
| Advent of Code | 매년 12월 프로그래밍 퍼즐 (Rust로 풀어보기) | https://adventofcode.com/ |
| LeetCode | 알고리즘 문제 풀기 (Rust 지원) | https://leetcode.com/ |

### Rustlings 시작 방법

```bash
# Rustlings 설치
cargo install rustlings

# 연습 시작
rustlings init
cd rustlings
rustlings
```

Rustlings는 소유권, 트레이트 등 Rust 핵심 개념을 작은 문제로 연습할 수 있어서, 이 학습지와 병행하면 효과가 좋습니다.

---

## 3. 한국어 자료

| 자료 | 설명 | 링크 |
|------|------|------|
| Rust 한국어 번역 (The Book) | 공식 입문서의 한국어 번역 | https://doc.rust-kr.org/ |
| 한국 Rust 사용자 그룹 | 한국어로 Rust를 논의하는 커뮤니티 | https://rust-kr.org/ |
| Rust Korea Discord | 한국어 Rust 디스코드 채널 | https://discord.gg/uqXGjEz |

한국어 자료는 영어 자료에 비해 양이 적지만, 한국 Rust 사용자 그룹에서 질문하면 친절하게 답변을 받을 수 있습니다.

---

## 4. 영어 자료

### 뉴스레터와 블로그

| 자료 | 설명 | 링크 |
|------|------|------|
| This Week in Rust | 매주 발행되는 Rust 뉴스레터 | https://this-week-in-rust.org/ |
| Rust Blog | Rust 공식 블로그 (업데이트, 릴리즈 소식) | https://blog.rust-lang.org/ |
| Read Rust | Rust 관련 글 모음 | https://readrust.net/ |

### 유튜브 채널

| 채널 | 설명 | 링크 |
|------|------|------|
| Jon Gjengset | 고급 Rust 주제를 깊이 있게 설명 (중급 이상) | https://www.youtube.com/@jonhoo |
| Let's Get Rusty | Rust 입문자 친화적인 튜토리얼 | https://www.youtube.com/@letsgetrusty |
| No Boilerplate | Rust의 장점을 재미있게 소개 | https://www.youtube.com/@NoBoilerplate |
| Logan Smith | Rust 개념을 명확하게 설명 | https://www.youtube.com/@_noisecode |

### 추천 도서

| 도서 | 설명 | 난이도 |
|------|------|--------|
| Programming Rust (O'Reilly) | 포괄적인 Rust 학습서 | 중급 |
| Rust in Action | 시스템 프로그래밍 관점의 Rust | 중급 |
| Zero To Production In Rust | Rust로 웹 백엔드 만들기 (이 학습지 이후 추천) | 중~고급 |
| Rust for Rustaceans | 고급 Rust 패턴과 관용구 | 고급 |

---

## 5. 커뮤니티

### 질문하고 토론하기

| 커뮤니티 | 설명 | 링크 |
|----------|------|------|
| Rust 공식 Discord | 가장 활발한 Rust 커뮤니티 (영어) | https://discord.gg/rust-lang |
| Rust Users Forum | 공식 사용자 포럼 (영어) | https://users.rust-lang.org/ |
| Reddit r/rust | Rust 서브레딧 (뉴스, 토론) | https://www.reddit.com/r/rust/ |
| Reddit r/learnrust | Rust 학습 전용 서브레딧 (초보 질문 환영) | https://www.reddit.com/r/learnrust/ |
| Stack Overflow [rust] | 프로그래밍 Q&A (rust 태그) | https://stackoverflow.com/questions/tagged/rust |

### 질문할 때 팁

1. **에러 메시지 전체**를 포함하세요.
2. **재현 가능한 최소 코드**를 첨부하세요 (Rust Playground 링크가 좋습니다).
3. **이미 시도한 것**을 적으세요.
4. r/learnrust나 Discord의 `#beginners` 채널은 초보 질문에 매우 친절합니다.

---

## 6. 유용한 도구

| 도구 | 설명 | 링크 |
|------|------|------|
| Rust Playground | 브라우저에서 Rust 코드 실행 | https://play.rust-lang.org/ |
| crates.io | Rust 크레이트(라이브러리) 저장소 | https://crates.io/ |
| docs.rs | 모든 크레이트의 자동 생성 문서 | https://docs.rs/ |
| lib.rs | crates.io를 더 보기 좋게 정리한 사이트 | https://lib.rs/ |
| Rust Analyzer | VS Code용 Rust 확장 (필수) | https://rust-analyzer.github.io/ |
| Clippy | Rust 코드 린터 (더 나은 코드 작성 도움) | `cargo clippy` |

---

## 7. 다음 단계 추천

이 학습지를 완료한 후 더 배울 수 있는 주제들입니다.

### 더 배울 것들

| 주제 | 설명 | 추천 자료 |
|------|------|-----------|
| 매크로 | `macro_rules!`, 절차적 매크로 | The Rust Programming Language 19장 |
| unsafe Rust | 안전하지 않은 코드 작성 | Rustonomicon |
| WebAssembly | Rust를 웹 브라우저에서 실행 | Rust and WebAssembly Book |
| 임베디드 Rust | 마이크로컨트롤러 프로그래밍 | The Embedded Rust Book |
| 고급 비동기 | 커스텀 Future, 런타임 이해 | Tokio 공식 튜토리얼 |

### 추천 프로젝트 아이디어 5개

학습지를 마친 후 실력을 키울 수 있는 프로젝트입니다. 난이도 순으로 정리했습니다.

#### 1. CLI 할 일 관리 앱

- **난이도**: 초급
- **배우는 것**: 파일 I/O, 구조체, 에러 처리, CLI 인자 파싱
- **설명**: 터미널에서 할 일을 추가, 삭제, 완료 표시할 수 있는 프로그램
- **추천 크레이트**: `clap` (CLI 인자 파싱), `serde_json` (데이터 저장)

#### 2. URL 단축 서비스

- **난이도**: 중급
- **배우는 것**: 웹 서버, 데이터베이스, 해시 생성
- **설명**: 긴 URL을 짧은 URL로 변환하고 리다이렉트하는 웹 서비스
- **추천 크레이트**: `axum`, `sqlx`, `nanoid` (짧은 ID 생성)

#### 3. 채팅 서버

- **난이도**: 중급
- **배우는 것**: WebSocket, 동시성, 메시지 브로드캐스트
- **설명**: 여러 사용자가 실시간으로 대화할 수 있는 채팅 서버
- **추천 크레이트**: `axum`, `tokio-tungstenite` (WebSocket)

#### 4. REST API + 프론트엔드 풀스택 프로젝트

- **난이도**: 중~고급
- **배우는 것**: 풀스택 개발, CORS, 인증, 배포
- **설명**: Rust 백엔드 + React/Vue 프론트엔드로 완전한 웹 애플리케이션 구축
- **추천 크레이트**: 학습지에서 사용한 전체 스택

#### 5. 간단한 Redis 클론

- **난이도**: 고급
- **배우는 것**: 네트워크 프로그래밍, 프로토콜 파싱, 동시성, 메모리 관리
- **설명**: TCP 소켓으로 기본적인 GET/SET 명령을 처리하는 인메모리 키-값 저장소
- **추천 크레이트**: `tokio` (네트워크 I/O), `bytes` (바이트 처리)
- **참고 자료**: Tokio 공식 미니 Redis 튜토리얼 (https://tokio.rs/tokio/tutorial)

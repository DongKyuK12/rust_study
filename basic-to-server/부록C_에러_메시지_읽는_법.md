# 부록 C. Rust 에러 메시지 읽는 법

> **Rust 컴파일러는 세상에서 가장 친절한 컴파일러 중 하나입니다.**
> 에러 메시지를 잘 읽으면 대부분의 문제를 스스로 해결할 수 있습니다.

---

## 1. 에러 메시지 구조 분석

Rust 컴파일러 에러 메시지는 일정한 구조를 가지고 있습니다. 이 구조를 이해하면 에러를 빠르게 파악할 수 있습니다.

### 에러 메시지 예시

```
error[E0382]: borrow of moved value: `s`
 --> src/main.rs:4:20
  |
2 |     let s = String::from("hello");
  |         - move occurs because `s` has type `String`, which does not implement the `Copy` trait
3 |     let s2 = s;
  |              - value moved here
4 |     println!("{}", s);
  |                    ^ value borrowed here after move
  |
help: consider cloning the value if the performance cost is acceptable
  |
3 |     let s2 = s.clone();
  |               ++++++++
```

### 각 부분의 의미

```
[1] error[E0382]: borrow of moved value: `s`
     ^^^^^ ^^^^^  ^^^^^^^^^^^^^^^^^^^^^^^^^^^
     종류  에러코드       에러 내용

[2]  --> src/main.rs:4:20
         ^^^^^^^^^^^ ^ ^^
         파일명      줄 열(칸)

[3]  |
  2  |     let s = String::from("hello");
     |         - move occurs because ...
  3  |     let s2 = s;
     |              - value moved here
  4  |     println!("{}", s);
     |                    ^ value borrowed here after move
     코드와 함께 문제 위치를 표시

[4] help: consider cloning the value ...
     ^^^^
     해결 방법 제안
```

| 부분 | 의미 | 확인 포인트 |
|------|------|-------------|
| `error` / `warning` | 에러의 종류 | `error`는 반드시 고쳐야 하고, `warning`은 참고 사항 |
| `[E0382]` | 에러 코드 | `rustc --explain E0382`로 자세한 설명 확인 가능 |
| 에러 내용 | 무엇이 잘못되었는지 | 핵심 문제를 한 줄로 설명 |
| `-->` 위치 | 에러가 발생한 파일, 줄, 칸 | 해당 위치의 코드를 확인 |
| `help:` | 해결 방법 제안 | **이 부분을 가장 먼저 읽으세요!** |

---

## 2. 자주 만나는 에러 Top 10

### E0382: borrow of moved value (소유권 이동 후 사용)

**의미**: 이미 다른 변수로 소유권이 넘어간 값을 다시 사용하려고 했습니다.

**관련 장**: 04장 (소유권)

**원인**: `String` 같은 힙 데이터는 대입(=)하면 소유권이 이동합니다. 이동 후 원래 변수는 사용할 수 없습니다.

```rust
// 에러 코드
fn main() {
    let s = String::from("hello");
    let s2 = s;        // s의 소유권이 s2로 이동
    println!("{}", s);  // 에러! s는 더 이상 유효하지 않음
}
```

```rust
// 해결 방법 1: clone() 사용
fn main() {
    let s = String::from("hello");
    let s2 = s.clone();   // 값을 복사
    println!("{}", s);     // s도 여전히 사용 가능
}
```

```rust
// 해결 방법 2: 참조 사용
fn main() {
    let s = String::from("hello");
    let s2 = &s;           // 빌리기만 함
    println!("{}", s);     // s도 여전히 사용 가능
}
```

---

### E0502: cannot borrow as mutable (가변/불변 빌림 충돌)

**의미**: 불변 참조가 존재하는 동안 가변 참조를 만들려고 했습니다.

**관련 장**: 04장 (소유권)

**원인**: Rust는 데이터 경합을 막기 위해, 불변 참조가 있는 동안에는 가변 참조를 허용하지 않습니다.

```rust
// 에러 코드
fn main() {
    let mut v = vec![1, 2, 3];
    let first = &v[0];      // 불변 빌림
    v.push(4);               // 에러! 가변 빌림 시도
    println!("{}", first);
}
```

```rust
// 해결 방법: 불변 참조 사용이 끝난 후에 가변 참조 사용
fn main() {
    let mut v = vec![1, 2, 3];
    let first = &v[0];
    println!("{}", first);   // 불변 참조 사용 완료
    v.push(4);               // 이제 가변 빌림 가능
}
```

---

### E0308: mismatched types (타입 불일치)

**의미**: 예상된 타입과 실제 타입이 다릅니다.

**관련 장**: 02장 (변수와 데이터 타입)

**원인**: 함수 반환값, 변수 대입, 함수 인자 등에서 타입이 맞지 않습니다.

```rust
// 에러 코드
fn add(a: i32, b: i32) -> i32 {
    a + b;  // 에러! 세미콜론 때문에 ()를 반환
}
```

```rust
// 해결 방법: 세미콜론 제거 (마지막 표현식이 반환값)
fn add(a: i32, b: i32) -> i32 {
    a + b   // 세미콜론 없으면 이 값이 반환됨
}
```

```rust
// 또 다른 흔한 경우
fn get_name() -> String {
    "hello"  // 에러! &str인데 String을 기대함
}

// 해결 방법
fn get_name() -> String {
    "hello".to_string()  // &str을 String으로 변환
}
```

---

### E0599: no method named ... found (메서드 없음)

**의미**: 해당 타입에 호출한 메서드가 존재하지 않습니다.

**관련 장**: 05장 (구조체와 열거형), 09장 (트레이트)

**원인**: 오타, 트레이트를 `use`하지 않음, 또는 해당 메서드가 실제로 없는 경우입니다.

```rust
// 에러 코드
let v = vec![1, 2, 3];
let s = v.join(",");  // 에러! Vec<i32>에는 join 메서드가 없음
```

```rust
// 해결 방법: 올바른 메서드 사용
let v = vec!["a", "b", "c"];
let s = v.join(",");  // Vec<&str>에는 join이 있음 -> "a,b,c"

// 숫자 벡터를 문자열로 합치려면
let v = vec![1, 2, 3];
let s: String = v.iter().map(|n| n.to_string()).collect::<Vec<_>>().join(",");
```

---

### E0425: cannot find value (변수나 함수를 찾을 수 없음)

**의미**: 사용하려는 이름(변수, 함수, 모듈 등)을 찾을 수 없습니다.

**관련 장**: 03장 (함수), 08장 (모듈)

**원인**: 오타, `use` 누락, 스코프 밖에서 접근 등입니다.

```rust
// 에러 코드
fn main() {
    let result = add(1, 2);  // 에러! add 함수를 찾을 수 없음
}
```

```rust
// 해결 방법: 함수 정의 추가 또는 use로 가져오기
fn add(a: i32, b: i32) -> i32 {
    a + b
}

fn main() {
    let result = add(1, 2);  // 이제 동작함
}
```

---

### E0106: missing lifetime specifier (라이프타임 누락)

**의미**: 참조를 반환하는데 라이프타임을 명시하지 않았습니다.

**관련 장**: 10장 (라이프타임 심화)

**원인**: 함수가 참조(&)를 반환할 때, 컴파일러가 그 참조가 얼마나 유효한지 알 수 없는 경우입니다.

```rust
// 에러 코드
fn longest(a: &str, b: &str) -> &str {  // 에러! 라이프타임 누락
    if a.len() > b.len() { a } else { b }
}
```

```rust
// 해결 방법: 라이프타임 어노테이션 추가
fn longest<'a>(a: &'a str, b: &'a str) -> &'a str {
    if a.len() > b.len() { a } else { b }
}
```

---

### E0277: the trait bound ... is not satisfied (트레이트 미구현)

**의미**: 필요한 트레이트가 해당 타입에 구현되어 있지 않습니다.

**관련 장**: 09장 (제네릭과 트레이트)

**원인**: 제네릭 함수에서 요구하는 트레이트를 타입이 구현하지 않은 경우입니다.

```rust
// 에러 코드
#[derive(Debug)]
struct User {
    name: String,
}

let user = User { name: "철수".to_string() };
println!("{}", user);  // 에러! Display 트레이트가 구현되지 않음
```

```rust
// 해결 방법 1: Display 트레이트 직접 구현
use std::fmt;

impl fmt::Display for User {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "{}", self.name)
    }
}
```

```rust
// 해결 방법 2: Debug 출력 사용 (간단한 경우)
println!("{:?}", user);  // Debug 출력 사용
```

---

### E0433: failed to resolve (모듈이나 크레이트를 찾을 수 없음)

**의미**: `use`로 가져오려는 모듈이나 크레이트를 찾을 수 없습니다.

**관련 장**: 08장 (모듈과 패키지)

**원인**: 크레이트가 `Cargo.toml`에 추가되지 않았거나, 모듈 경로가 잘못된 경우입니다.

```rust
// 에러 코드
use serde::Serialize;  // 에러! serde가 Cargo.toml에 없음
```

```bash
# 해결 방법: 크레이트 추가
cargo add serde --features derive
```

```rust
// 이제 사용 가능
use serde::Serialize;

#[derive(Serialize)]
struct User {
    name: String,
}
```

---

### E0507: cannot move out of borrowed content (빌린 값에서 소유권 이동 불가)

**의미**: 빌린 참조(&)에서 값의 소유권을 가져오려고 했습니다.

**관련 장**: 04장 (소유권)

**원인**: `&self`나 `&T`에서 내부 값을 이동(move)하려고 시도한 경우입니다.

```rust
// 에러 코드
struct User {
    name: String,
}

fn get_name(user: &User) -> String {
    user.name  // 에러! 빌린 값에서 소유권을 가져올 수 없음
}
```

```rust
// 해결 방법 1: clone() 사용
fn get_name(user: &User) -> String {
    user.name.clone()  // 값을 복사해서 반환
}
```

```rust
// 해결 방법 2: 참조를 반환
fn get_name(user: &User) -> &str {
    &user.name  // 참조를 반환
}
```

---

### E0515: cannot return reference to temporary value (임시 값의 참조 반환 불가)

**의미**: 함수 안에서 만든 임시 값의 참조를 반환하려고 했습니다.

**관련 장**: 04장 (소유권), 10장 (라이프타임)

**원인**: 함수가 끝나면 지역 변수가 사라지는데, 사라질 값의 참조를 반환하면 댕글링 참조가 됩니다.

```rust
// 에러 코드
fn create_greeting() -> &str {
    let s = String::from("안녕하세요");
    &s  // 에러! s는 함수가 끝나면 사라짐
}
```

```rust
// 해결 방법: 소유권을 가진 값을 반환
fn create_greeting() -> String {
    let s = String::from("안녕하세요");
    s  // String을 통째로 반환 (소유권 이동)
}
```

---

## 3. 에러 해결 팁

### 팁 1: help 메시지를 가장 먼저 읽으세요

Rust 컴파일러의 `help:` 부분은 대부분 정확한 해결책을 알려줍니다. 에러 내용보다 `help:`를 먼저 보는 습관을 들이세요.

```
help: consider cloning the value if the performance cost is acceptable
  |
3 |     let s2 = s.clone();
  |               ++++++++
```

위 메시지는 "`.clone()`을 추가하세요"라고 구체적으로 알려주고 있습니다.

### 팁 2: rustc --explain으로 자세한 설명 보기

에러 코드(E0XXX)에 대한 자세한 설명을 볼 수 있습니다.

```bash
rustc --explain E0382
```

실행하면 해당 에러의 원인, 예시 코드, 해결 방법이 상세하게 출력됩니다.

### 팁 3: 첫 번째 에러부터 해결하세요

에러가 여러 개 나올 때, 하나의 에러가 다른 에러를 연쇄적으로 발생시키는 경우가 많습니다. **맨 위의 첫 번째 에러**부터 해결하면 나머지 에러도 자동으로 사라지는 경우가 많습니다.

```bash
# 에러가 많을 때, 처음 몇 개만 보기
cargo check 2>&1 | head -30
```

### 팁 4: cargo check를 자주 실행하세요

코드를 조금 수정할 때마다 `cargo check`를 실행하세요. 에러가 쌓이기 전에 하나씩 잡는 것이 훨씬 쉽습니다.

### 팁 5: 에러 메시지를 검색하세요

에러 메시지를 그대로 복사해서 검색하면 같은 문제를 겪은 다른 개발자의 해결 방법을 찾을 수 있습니다.

- 검색 예시: `rust E0382 borrow of moved value`
- Stack Overflow의 `[rust]` 태그에 많은 해결 사례가 있습니다

---

## 에러 종류 한눈에 보기

| 에러 코드 | 한줄 요약 | 관련 개념 | 관련 장 |
|-----------|-----------|-----------|---------|
| E0382 | 이동된 값을 다시 사용 | 소유권 | 04장 |
| E0502 | 가변/불변 빌림 충돌 | 빌림 규칙 | 04장 |
| E0308 | 타입이 맞지 않음 | 타입 시스템 | 02장 |
| E0599 | 메서드가 없음 | 구조체, 트레이트 | 05장, 09장 |
| E0425 | 이름을 찾을 수 없음 | 스코프, 모듈 | 03장, 08장 |
| E0106 | 라이프타임 누락 | 라이프타임 | 10장 |
| E0277 | 트레이트 미구현 | 트레이트 | 09장 |
| E0433 | 모듈/크레이트 못 찾음 | 모듈, 의존성 | 08장 |
| E0507 | 빌린 값에서 소유권 이동 | 소유권, 빌림 | 04장 |
| E0515 | 임시 값의 참조 반환 | 라이프타임 | 04장, 10장 |

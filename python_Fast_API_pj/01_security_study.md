# Security

비밀번호 암호화와 JWT 토큰 생성/검증을 담당하는 모듈

---

## 역할

- 비밀번호 해싱(Hashing)
- 비밀번호 검증
- Access Token 생성
- Refresh Token 생성
- JWT 토큰 검증
- 토큰 만료 시간 관리

---

## 1. 비밀번호 해싱

비밀번호를 DB에 평문으로 저장하면 해킹 시 그대로 노출된다.

따라서 bcrypt 알고리즘으로 암호화하여 저장하고, 로그인 시에는 암호화된 값과 비교만 수행한다.

### bcrypt 설정

```python
pwd_context = CryptContext(
    schemes=["bcrypt"],
    deprecated="auto"
)
```

- `passlib`의 `CryptContext` 사용
- bcrypt 알고리즘 사용
- 비밀번호 암호화 및 검증 기능 제공

### 비밀번호 암호화

```python
def hash_password(password: str) -> str:
    return pwd_context.hash(password)
```

회원가입 시 사용한다.

#### 동작 과정

1. 사용자가 비밀번호 입력
2. bcrypt로 해싱
3. 해싱된 문자열을 DB에 저장

예시

```text
입력 비밀번호
↓
1234

DB 저장 값
↓
$2b$12$...
```

### 비밀번호 검증

```python
def verify_password(
    plain: str,
    hashed: str
) -> bool:
    return pwd_context.verify(plain, hashed)
```

로그인 시 사용한다.

#### 동작 과정

1. 사용자가 비밀번호 입력
2. DB에서 암호화된 비밀번호 조회
3. bcrypt가 내부적으로 비교
4. 일치 여부 반환

반환값

```python
True
False
```

---

## 2. JWT(JSON Web Token)

JWT는 로그인 상태를 유지하기 위한 토큰 방식이다.

### JWT 구조

```text
Header.Payload.Signature
```

예시

```text
eyJhbGciOi...
.
eyJ1c2VyX2lkIjox...
.
abcedfg...
```

### Header

토큰 생성에 사용된 알고리즘 정보

예시

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

### Payload

실제 저장할 데이터

예시

```json
{
  "user_id": 1,
  "type": "access",
  "exp": 1719999999
}
```

### Signature

토큰 위조 방지용 서명

생성 시 사용

```python
SECRET_KEY
```

---

## 3. Access Token

```python
def create_access_token(
    user_id: int
) -> str:
```

### 역할

API 요청 시 사용자 인증

### 특징

- 만료 시간: 15분
- 사용자 정보 포함
- 요청마다 전송

### Payload 예시

```json
{
  "user_id": 1,
  "type": "access",
  "exp": "만료시간"
}
```

### 사용 흐름

```text
로그인 성공
↓
Access Token 발급
↓
클라이언트 저장
↓
API 요청 시 Authorization 헤더에 포함
```

---

## 4. Refresh Token

```python
def create_refresh_token(
    user_id: int
) -> tuple[str, str]:
```

### 역할

Access Token 재발급

### 특징

- 만료 시간: 7일
- 고유 ID(jti) 포함
- Redis와 함께 사용

### Payload 예시

```json
{
  "user_id": 1,
  "type": "refresh",
  "jti": "uuid값",
  "exp": "만료시간"
}
```

### 반환값

```python
(token, jti)
```

예시

```python
(
    "jwt.token.value",
    "c4afc9c2-..."
)
```

---

## 5. jti가 필요한 이유

JWT는 Stateless 방식이다.

즉, 서버가 토큰을 저장하지 않으므로 이미 발급된 토큰을 강제로 취소하기 어렵다.

### 문제 상황

```text
로그인
↓
JWT 발급
↓
토큰 탈취
↓
만료 전까지 사용 가능
```

### 해결 방법

Refresh Token의 `jti`를 Redis에 저장한다.

```text
Redis
↓
jti 저장
```

로그아웃 시

```text
Redis에서 삭제
↓
토큰 무효화
```

즉,

- 로그인 → Redis 저장
- 로그아웃 → Redis 삭제
- 재발급 → Redis 조회

방식으로 관리한다.

---

## 6. 토큰 검증

```python
def decode_token(
    token: str
) -> Optional[dict]:
```

### 역할

JWT를 검증하고 Payload를 반환한다.

### 동작 과정

```text
JWT 수신
↓
서명 검증
↓
만료 시간 검증
↓
Payload 반환
```

### 성공

```python
{
    "user_id": 1,
    "type": "access"
}
```

### 실패

```python
None
```

발생 가능한 경우

- 만료된 토큰
- 변조된 토큰
- 잘못된 서명

---

## 7. UTC를 사용하는 이유

서버는 서로 다른 시간대를 사용할 수 있다.

예를 들어

```text
한국 서버 : UTC+9
미국 서버 : UTC-5
```

각 서버의 로컬 시간을 사용하면 만료 시간 계산이 달라질 수 있다.

### 해결 방법

모든 시간을 UTC 기준으로 저장

```text
UTC
↓
세계 공통 표준 시간
```

### 장점

- 서버 위치와 무관
- 만료 시간 계산 일관성 유지
- 시간 비교 오류 감소

---

## 전체 흐름

```text
회원가입
↓
비밀번호 bcrypt 해싱
↓
DB 저장

로그인
↓
비밀번호 검증
↓
Access Token 발급
↓
Refresh Token 발급
↓
Redis에 jti 저장

API 요청
↓
Access Token 검증
↓
사용자 인증

Access Token 만료
↓
Refresh Token 사용
↓
새 Access Token 발급

로그아웃
↓
Redis에서 jti 삭제
↓
Refresh Token 무효화
```
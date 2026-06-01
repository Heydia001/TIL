# Redis & 프로젝트 구조 노트

## 지금까지 만든 파일들

`config.py`
- .env에서 환경변수를 읽어서 전역 상수로 관리
- SECRET_KEY, ALGORITHM, DB/Redis URL, 토큰 만료시간 등

`security.py`
- 비밀번호 bcrypt 암호화/검증
- JWT Access Token, Refresh Token 생성
- JWT 토큰 decode (위조/만료 시 None 반환)

`database.py`
- PostgreSQL 커넥션 풀 관리 (최소 1개, 최대 10개 연결 유지)
- get_db() — 요청마다 연결 꺼내고, 끝나면 반납

`redis_client.py`
- Refresh Token을 Redis에 저장/조회/삭제

---

## Redis란?

메모리 기반의 Key-Value 저장소
DB보다 훨씬 빠르고, TTL(만료시간) 설정이 가능해서 토큰 관리에 적합함

---

## Refresh Token을 Redis에 저장하는 이유

JWT는 한번 발급하면 서버에서 강제로 무효화할 수 없음
그래서 jti(고유 ID)를 Redis에 저장해두고
로그아웃하거나 탈취가 의심되면 Redis에서 삭제해서 무효화함

---

## Redis 저장 구조

```
key   : refresh:{jti}
value : user_id
TTL   : 7일 (자동 만료)
```

예시
```
key   : refresh:550e8400-e29b-41d4-a716-446655440000
value : 42
TTL   : 604800초 (7 * 24 * 60 * 60)
```

---

## redis_client.py 함수 정리

`save_refresh_token(jti, user_id)`
- 로그인 성공 시 호출
- Redis에 jti를 키로, user_id를 값으로 저장
- TTL 7일 설정

`get_refresh_token_owner(jti)`
- 토큰 재발급 시 호출
- jti로 Redis 조회해서 user_id 반환
- 없으면 None (만료되거나 삭제된 토큰)

`delete_refresh_token(jti)`
- 로그아웃 시 호출
- Redis에서 jti 삭제 → 토큰 무효화

---

## TTL 계산

```python
ttl = REFRESH_TOKEN_EXPIRE_DAYS * 24 * 60 * 60
#     7일                        시  분  초
#     = 604800초
```
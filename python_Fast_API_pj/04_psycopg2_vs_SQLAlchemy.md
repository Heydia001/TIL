# SQLAlchemy 전환 이유 노트

## 핵심 차이

psycopg2는 DB 드라이버, SQLAlchemy는 ORM이에요.
SQLAlchemy가 내부적으로 psycopg2를 드라이버로 사용해요.

```
SQLAlchemy (ORM) → psycopg2 (드라이버) → PostgreSQL
```

---

## database.py

**변경 전**
```python
from psycopg2.pool import SimpleConnectionPool
pool = SimpleConnectionPool(1, 10, DATABASE_URL)
```

**변경 후**
```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
engine = create_engine(DATABASE_URL)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
```

**이유**
- psycopg2는 연결 풀을 직접 관리해야 했음
- SQLAlchemy는 `create_engine` 이 내부적으로 연결 풀 자동 관리
- `Session` 으로 DB 작업을 트랜잭션 단위로 관리 가능

---

## routers/auth.py

**변경 전**
```python
with conn.cursor(cursor_factory=psycopg2.extras.RealDictCursor) as cur:
    cur.execute("SELECT user_id FROM users WHERE email = %s", (body.email,))
    existing = cur.fetchone()
```

**변경 후**
```python
existing = db.query(User).filter(User.email == body.email).first()
```

**이유**
- raw SQL 문자열 대신 Python 코드로 쿼리 작성
- 오타가 줄고 IDE 자동완성 지원
- `conn` 대신 `db` (Session) 으로 트랜잭션 자동 관리

---

## dependencies.py

**변경 전**
```python
from psycopg2.extensions import connection

with conn.cursor(cursor_factory=psycopg2.extras.RealDictCursor) as cur:
    cur.execute("SELECT user_id, email, nickname FROM users WHERE user_id = %s", ...)
    user = cur.fetchone()
```

**변경 후**
```python
from sqlalchemy.orm import Session

user = db.query(User).filter(User.user_id == int(payload["sub"])).first()
```

**이유**
- raw SQL 제거
- 이전에는 딕셔너리로 반환됐는데, 이제 `User` 객체로 반환되어 타입이 명확해짐

---

## 정리

| | psycopg2 | SQLAlchemy |
|---|---|---|
| 쿼리 방식 | raw SQL 문자열 | Python 코드 |
| 연결 풀 | 직접 관리 | 자동 관리 |
| 반환값 | 딕셔너리 | 객체 |
| 오타 감지 | 런타임에서 발견 | IDE에서 미리 감지 |
| DB 교체 | 코드 수정 필요 | URL만 변경 |
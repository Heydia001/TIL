# SQLAlchemy 전환 노트

## 핵심 개념

```
SQLAlchemy (ORM) → psycopg2 (드라이버) → PostgreSQL
```

psycopg2 = DB 드라이버
SQLAlchemy = 그 위에서 동작하는 ORM
SQLAlchemy가 내부적으로 psycopg2를 사용함

---

## 1. DB 연결 방식

```python
# psycopg2 — 연결 풀 직접 관리
from psycopg2.pool import SimpleConnectionPool
pool = SimpleConnectionPool(1, 10, DATABASE_URL)

def get_db():
    conn = pool.getconn()
    try:
        yield conn
        conn.commit()
    except Exception:
        conn.rollback()
        raise
    finally:
        pool.putconn(conn)  # 직접 반납해야 함


# SQLAlchemy — 연결 풀 자동 관리
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
engine = create_engine(DATABASE_URL)  # 내부적으로 풀 자동 관리
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

def get_db():
    db = SessionLocal()
    try:
        yield db
    except Exception:
        db.rollback()
        raise
    finally:
        db.close()
```

psycopg2는 getconn/putconn으로 직접 꺼내고 반납해야 함
SQLAlchemy는 engine이 알아서 풀 관리해줌

---

## 2. 쿼리 방식

```python
# psycopg2 — raw SQL 문자열
with conn.cursor(cursor_factory=psycopg2.extras.RealDictCursor) as cur:
    cur.execute(
        "SELECT user_id FROM users WHERE email = %s",
        (body.email,)
    )
    existing = cur.fetchone()


# SQLAlchemy — Python 코드로 쿼리
existing = db.query(User).filter(User.email == body.email).first()
```

raw SQL은 오타나도 런타임에서야 발견됨
SQLAlchemy는 Python 코드라 IDE가 미리 잡아줌

---

## 3. 데이터 저장

```python
# psycopg2 — INSERT SQL 직접 작성
with conn.cursor(cursor_factory=psycopg2.extras.RealDictCursor) as cur:
    cur.execute(
        "INSERT INTO users(email, password, nickname, created_at) VALUES (%s, %s, %s, %s) RETURNING user_id, email, nickname, created_at",
        (body.email, hash_password(body.password), body.nickname, datetime.now())
    )
    user = cur.fetchone()


# SQLAlchemy — 객체로 저장
user = User(
    email=body.email,
    password=hash_password(body.password),
    nickname=body.nickname,
    created_at=datetime.now()
)
db.add(user)
db.commit()
db.refresh(user)  # DB에서 최신 상태로 갱신
```

psycopg2는 SQL 직접 써야 함
SQLAlchemy는 객체 만들고 add/commit만 하면 됨
db.refresh(user) — commit 후 DB에서 자동 생성된 값(user_id 등) 다시 불러옴

---

## 4. 반환값

```python
# psycopg2 — 딕셔너리
user["user_id"]
user["email"]

# SQLAlchemy — 객체
user.user_id
user.email
```

딕셔너리는 오타나도 모름
객체는 IDE 자동완성 + 타입 체크 됨

---

## 비교 요약

| | psycopg2 | SQLAlchemy |
|---|---|---|
| 쿼리 방식 | raw SQL 문자열 | Python 코드 |
| 연결 풀 | 직접 관리 | 자동 관리 |
| 반환값 | 딕셔너리 | 객체 |
| 오타 감지 | 런타임 | IDE |
| DB 교체 | 코드 수정 필요 | URL만 변경 |

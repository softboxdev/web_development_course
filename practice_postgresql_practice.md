# Практическая работа № 5 
## Разработка веб-приложения с интеграцией PostgreSQL

**Дисциплина:** Информационные системы и программирование  
**Специальность:** 09.02.07  
**Тема:** Разработка полного веб-приложения ИС «Конференции.РФ» на PostgreSQL  
**Время выполнения:** 4 академических часа  
**ОС:** РОСА Линукс

---

## Цель работы

Переписать проект «Конференции.РФ» с SQLite на **PostgreSQL** и реализовать полный функционал:

1. Интегрировать фронтенд с бекендом.
2. Реализовать CRUD через PostgreSQL (`psycopg2`).
3. Реализовать смену статусов: **Новая → Мероприятие назначено → Завершено**.
4. Реализовать отзывы после завершения мероприятия.


## Оснащение рабочего места

| Компонент | Версия |
|-----------|--------|
| ОС | РОСА Линукс |
| Python | 3.8+ |
| PostgreSQL | 12+ |
| Flask | 2.3.3 |
| Werkzeug | 2.3.7 |
| psycopg2-binary | 2.9.9 |
| python-dotenv | 1.0.0 |

---

## Шаг 1. Проверка PostgreSQL (10 мин)

### 1.1. Проверка статуса службы

```bash
sudo systemctl status postgresql12.service
```

**Ожидаемый результат:**
```
● postgresql12.service - PostgreSQL 12 database server
   Active: active (running)
```

**Если не запущена:**

```bash
sudo systemctl start postgresql12.service
sudo systemctl enable postgresql12.service
```

### 1.2. Проверка подключения

```bash
sudo -u postgres psql -c "SELECT version();"
```

**Ожидаемый результат:**
```
PostgreSQL 12.22 on x86_64-rosa-linux-gnu
```

### 1.3. Проверка базы данных и пользователя

```bash
sudo -u postgres psql -c "\l" | grep conference
```

**Ожидаемый результат:**
```
conference_db | conference_user | UTF8 | ...
```

**Если базы нет** — создайте:

```bash
sudo -u postgres psql <<EOF
CREATE USER conference_user WITH PASSWORD 'Conf2027!';
CREATE DATABASE conference_db OWNER conference_user;
GRANT ALL PRIVILEGES ON DATABASE conference_db TO conference_user;
EOF
```

---

## Шаг 2. Адаптация `schema.sql` под PostgreSQL (15 мин)

### 2.1. Ключевые различия SQLite и PostgreSQL

| SQLite | PostgreSQL |
|--------|------------|
| `INTEGER PRIMARY KEY AUTOINCREMENT` | `SERIAL PRIMARY KEY` |
| `?` (плейсхолдер) | `%s` |
| `INSERT OR IGNORE` | `INSERT ... ON CONFLICT DO NOTHING` |
| `DATETIME` | `TIMESTAMP` |
| `PRAGMA foreign_keys = ON` | Не требуется (FK работают по умолчанию) |
| `user` — не зарезервировано | `"user"` — **зарезервировано**, нужны кавычки |

### 2.2. Создайте `schema_postgres.sql`

```sql
-- Таблица «Пользователь»
CREATE TABLE IF NOT EXISTS "user" (
    id SERIAL PRIMARY KEY,
    login VARCHAR(50) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    phone VARCHAR(18) NOT NULL,
    role VARCHAR(20) NOT NULL DEFAULT 'user'
);

-- Таблица «Мероприятие»
CREATE TABLE IF NOT EXISTS event (
    id SERIAL PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    date DATE NOT NULL,
    place VARCHAR(200) NOT NULL,
    description TEXT
);

-- Таблица «Статус»
CREATE TABLE IF NOT EXISTS status (
    id SERIAL PRIMARY KEY,
    name VARCHAR(50) UNIQUE NOT NULL
);

-- Наполнение справочника статусов
INSERT INTO status (name) VALUES
    ('Новая'),
    ('Мероприятие назначено'),
    ('Завершено')
ON CONFLICT (name) DO NOTHING;

-- Таблица «Заявка»
CREATE TABLE IF NOT EXISTS request (
    id SERIAL PRIMARY KEY,
    user_id INTEGER NOT NULL REFERENCES "user"(id) ON DELETE CASCADE,
    event_id INTEGER NOT NULL REFERENCES event(id) ON DELETE CASCADE,
    status_id INTEGER NOT NULL DEFAULT 1 REFERENCES status(id),
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- Индексы для ускорения запросов
CREATE INDEX IF NOT EXISTS idx_request_user ON request(user_id);
CREATE INDEX IF NOT EXISTS idx_request_status ON request(status_id);

-- Таблица «Отзыв»
CREATE TABLE IF NOT EXISTS review (
    id SERIAL PRIMARY KEY,
    request_id INTEGER UNIQUE NOT NULL REFERENCES request(id) ON DELETE CASCADE,
    text TEXT NOT NULL,
    rating INTEGER CHECK (rating BETWEEN 1 AND 5),
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

### 2.3. Применение схемы

```bash
PGPASSWORD='Conf2027!' psql -U conference_user -d conference_db -h 127.0.0.1 -f schema_postgres.sql
```

### 2.4. Проверка таблиц

```bash
PGPASSWORD='Conf2027!' psql -U conference_user -d conference_db -h 127.0.0.1 -c "\dt"
```

**Ожидаемый результат:**
```
 Schema |  Name   | Type  |     Owner
--------+---------+-------+------------------
 public | event   | table | conference_user
 public | request | table | conference_user
 public | review  | table | conference_user
 public | status  | table | conference_user
 public | user    | table | conference_user
```

---

## Шаг 3. Установка зависимостей (10 мин)

### 3.1. Активация виртуального окружения

```bash
cd ~/conference_portal
source venv/bin/activate
```

### 3.2. Установка psycopg2 и python-dotenv

```bash
pip install psycopg2-binary==2.9.9 python-dotenv==1.0.0
```

### 3.3. Обновление `requirements.txt`

```bash
pip freeze > requirements.txt
```

**Содержимое:**
```
Flask==2.3.3
Werkzeug==2.3.7
psycopg2-binary==2.9.9
python-dotenv==1.0.0
```

### 3.4. Создание `.env`

```bash
nano .env
```

**Содержимое:**

```env
DATABASE_URL=postgresql://conference_user:Conf2027!@127.0.0.1:5432/conference_db
FLASK_SECRET_KEY=change-me-to-random-32-bytes-hex-string
FLASK_DEBUG=False
```

> **Важно:** добавьте `.env` в `.gitignore`, если там есть реальные пароли.

---

## Шаг 4. Переписывание `database.py` (20 мин)

Откройте `backend/database.py` и **полностью замените** содержимое:

```python
import psycopg2
import psycopg2.extras
import os
from dotenv import load_dotenv

load_dotenv()


class Database:
    """Singleton-класс для работы с PostgreSQL."""
    _instance = None

    def __new__(cls, db_url=None):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
            cls._instance._initialized = False
        return cls._instance

    def __init__(self, db_url=None):
        if self._initialized:
            return
        if db_url is None:
            db_url = os.environ.get(
                'DATABASE_URL',
                'postgresql://conference_user:Conf2027!@127.0.0.1:5432/conference_db'
            )
        self.db_url = db_url
        self._initialized = True

    def get_connection(self):
        """Создать новое подключение к PostgreSQL."""
        conn = psycopg2.connect(self.db_url)
        return conn

    def execute(self, query, params=()):
        """
        Выполнить INSERT / UPDATE / DELETE.
        Для INSERT с RETURNING id возвращает id.
        """
        with self.get_connection() as conn:
            with conn.cursor() as cursor:
                cursor.execute(query, params)
                conn.commit()
                try:
                    result = cursor.fetchone()
                    return result[0] if result else None
                except psycopg2.ProgrammingError:
                    return None

    def fetch_one(self, query, params=()):
        """Получить одну запись как словарь."""
        with self.get_connection() as conn:
            with conn.cursor(
                cursor_factory=psycopg2.extras.RealDictCursor
            ) as cursor:
                cursor.execute(query, params)
                row = cursor.fetchone()
                return dict(row) if row else None

    def fetch_all(self, query, params=()):
        """Получить все записи как список словарей."""
        with self.get_connection() as conn:
            with conn.cursor(
                cursor_factory=psycopg2.extras.RealDictCursor
            ) as cursor:
                cursor.execute(query, params)
                return [dict(row) for row in cursor.fetchall()]

    def execute_script(self, script_path: str):
        """Выполнить SQL-скрипт из файла (например, schema_postgres.sql)."""
        with open(script_path, 'r', encoding='utf-8') as f:
            sql = f.read()
        with self.get_connection() as conn:
            with conn.cursor() as cursor:
                cursor.execute(sql)
                conn.commit()
```

**Ключевые изменения:**

| Было (SQLite) | Стало (PostgreSQL) |
|---------------|---------------------|
| `import sqlite3` | `import psycopg2` |
| `sqlite3.connect(self.db_path)` | `psycopg2.connect(self.db_url)` |
| `sqlite3.Row` | `psycopg2.extras.RealDictCursor` |
| `PRAGMA foreign_keys = ON` | Не нужно |
| `cursor.lastrowid` | `RETURNING id` в запросе |
| `database/conference.db` | `postgresql://user:pass@host:port/db` |

---

## Шаг 5. Адаптация всех моделей (30 мин)

### 5.1. Базовый класс `base_model.py`

```python
from backend.database import Database


class BaseModel:
    """Базовый класс для всех моделей."""
    table_name = None

    def __init__(self, **kwargs):
        self.db = Database()
        for key, value in kwargs.items():
            setattr(self, key, value)

    def save(self):
        raise NotImplementedError("Метод save() должен быть реализован")

    @classmethod
    def get_all(cls):
        db = Database()
        return db.fetch_all(f'SELECT * FROM {cls.table_name}')

    @classmethod
    def get_by_id(cls, record_id: int):
        db = Database()
        return db.fetch_one(
            f'SELECT * FROM {cls.table_name} WHERE id = %s',
            (record_id,)
        )
```

> **Обратите внимание:** `?` → `%s`.

### 5.2. Модель `User` — `backend/models/user.py`

```python
from backend.models.base_model import BaseModel
from backend.validators.validators import Validator
from werkzeug.security import generate_password_hash, check_password_hash


class User(BaseModel):
    table_name = '"user"'   # ← кавычки обязательны!

    def __init__(self, login=None, password=None, email=None,
                 phone=None, role='user', **kwargs):
        super().__init__(**kwargs)
        self.login = login
        self.password = password
        self.email = email
        self.phone = phone
        self.role = role

    def validate(self) -> tuple:
        checks = [
            Validator.validate_login(self.login),
            Validator.validate_password(self.password),
            Validator.validate_email(self.email),
            Validator.validate_phone(self.phone),
        ]
        for ok, msg in checks:
            if not ok:
                return False, msg
        return True, ""

    def login_exists(self) -> bool:
        row = self.db.fetch_one(
            'SELECT id FROM "user" WHERE login = %s', (self.login,)
        )
        return row is not None

    def email_exists(self) -> bool:
        row = self.db.fetch_one(
            'SELECT id FROM "user" WHERE email = %s', (self.email,)
        )
        return row is not None

    def save(self) -> tuple:
        ok, msg = self.validate()
        if not ok:
            return False, msg
        if self.login_exists():
            return False, "Пользователь с таким логином уже существует"
        if self.email_exists():
            return False, "Пользователь с таким email уже существует"

        password_hash = generate_password_hash(
            self.password,
            method='pbkdf2:sha256:600000',
            salt_length=16
        )
        user_id = self.db.execute(
            '''INSERT INTO "user" (login, password_hash, email, phone, role)
               VALUES (%s, %s, %s, %s, %s)
               RETURNING id''',
            (self.login, password_hash, self.email, self.phone, self.role)
        )
        self.id = user_id
        return True, user_id

    @staticmethod
    def authenticate(login: str, password: str) -> dict:
        db = Database()
        user = db.fetch_one(
            'SELECT * FROM "user" WHERE login = %s', (login,)
        )
        if user and check_password_hash(user['password_hash'], password):
            return user
        return None
```

### 5.3. Модель `Event` — `backend/models/event.py`

```python
from backend.models.base_model import BaseModel


class Event(BaseModel):
    table_name = "event"

    def __init__(self, name=None, date=None, place=None,
                 description=None, **kwargs):
        super().__init__(**kwargs)
        self.name = name
        self.date = date
        self.place = place
        self.description = description

    def save(self) -> int:
        event_id = self.db.execute(
            """INSERT INTO event (name, date, place, description)
               VALUES (%s, %s, %s, %s)
               RETURNING id""",
            (self.name, self.date, self.place, self.description)
        )
        self.id = event_id
        return event_id
```

### 5.4. Модель `Status` — `backend/models/status.py`

```python
from backend.models.base_model import BaseModel


class Status(BaseModel):
    table_name = "status"

    def __init__(self, name=None, **kwargs):
        super().__init__(**kwargs)
        self.name = name
```

### 5.5. Модель `Request` — `backend/models/request.py`

```python
from backend.database import Database
from backend.models.base_model import BaseModel


class Request(BaseModel):
    table_name = "request"

    def __init__(self, user_id=None, event_id=None,
                 status_id=1, **kwargs):
        super().__init__(**kwargs)
        self.user_id = user_id
        self.event_id = event_id
        self.status_id = status_id

    def save(self) -> int:
        request_id = self.db.execute(
            """INSERT INTO request (user_id, event_id, status_id)
               VALUES (%s, %s, %s)
               RETURNING id""",
            (self.user_id, self.event_id, self.status_id)
        )
        self.id = request_id
        return request_id

    @classmethod
    def get_by_user(cls, user_id: int):
        db = Database()
        return db.fetch_all(
            """SELECT r.id, e.name AS event_name, e.date,
                      e.place, s.name AS status_name,
                      r.status_id, r.created_at
               FROM request r
               JOIN event e ON r.event_id = e.id
               JOIN status s ON r.status_id = s.id
               WHERE r.user_id = %s
               ORDER BY r.created_at DESC""",
            (user_id,)
        )

    @classmethod
    def get_all_with_details(cls):
        db = Database()
        return db.fetch_all(
            """SELECT r.id, u.login, u.email, e.name AS event_name,
                      e.date, e.place, s.name AS status_name,
                      r.status_id, r.created_at
               FROM request r
               JOIN "user" u ON r.user_id = u.id
               JOIN event e ON r.event_id = e.id
               JOIN status s ON r.status_id = s.id
               ORDER BY r.created_at DESC"""
        )

    @classmethod
    def get_by_id(cls, request_id: int):
        db = Database()
        return db.fetch_one(
            """SELECT r.id, r.user_id, r.event_id, r.status_id,
                      u.login, e.name AS event_name, e.date, e.place,
                      s.name AS status_name, r.created_at
               FROM request r
               JOIN "user" u ON r.user_id = u.id
               JOIN event e ON r.event_id = e.id
               JOIN status s ON r.status_id = s.id
               WHERE r.id = %s""",
            (request_id,)
        )

    @classmethod
    def update_status(cls, request_id: int, status_id: int) -> bool:
        db = Database()
        db.execute(
            "UPDATE request SET status_id = %s WHERE id = %s",
            (status_id, request_id)
        )
        return True

    @classmethod
    def delete(cls, request_id: int) -> bool:
        db = Database()
        db.execute("DELETE FROM request WHERE id = %s", (request_id,))
        return True

    @classmethod
    def can_be_reviewed(cls, request_id: int) -> bool:
        db = Database()
        row = db.fetch_one(
            "SELECT status_id FROM request WHERE id = %s",
            (request_id,)
        )
        return row and row['status_id'] == 3
```

### 5.6. Модель `Review` — `backend/models/review.py`

```python
from backend.database import Database
from backend.models.base_model import BaseModel


class Review(BaseModel):
    table_name = "review"

    def __init__(self, request_id=None, text=None,
                 rating=None, **kwargs):
        super().__init__(**kwargs)
        self.request_id = request_id
        self.text = text
        self.rating = rating

    def validate(self) -> tuple:
        if not self.text or len(self.text.strip()) < 5:
            return False, "Текст отзыва минимум 5 символов"
        if len(self.text) > 2000:
            return False, "Текст отзыва слишком длинный"
        if not self.rating or not (1 <= int(self.rating) <= 5):
            return False, "Оценка должна быть от 1 до 5"
        return True, ""

    def save(self) -> tuple:
        ok, msg = self.validate()
        if not ok:
            return False, msg

        from backend.models.request import Request
        req = Request.get_by_id(self.request_id)
        if not req:
            return False, "Заявка не найдена"
        if req['status_id'] != 3:
            return False, "Отзыв — только после завершения мероприятия"
        if self.exists_for_request(self.request_id):
            return False, "Отзыв уже оставлен"

        review_id = self.db.execute(
            """INSERT INTO review (request_id, text, rating)
               VALUES (%s, %s, %s)
               RETURNING id""",
            (self.request_id, self.text.strip(), int(self.rating))
        )
        self.id = review_id
        return True, review_id

    @staticmethod
    def exists_for_request(request_id: int) -> bool:
        db = Database()
        row = db.fetch_one(
            "SELECT id FROM review WHERE request_id = %s",
            (request_id,)
        )
        return row is not None

    @classmethod
    def get_by_request(cls, request_id: int):
        db = Database()
        return db.fetch_one(
            """SELECT id, text, rating, created_at
               FROM review WHERE request_id = %s""",
            (request_id,)
        )

    @classmethod
    def get_all_with_details(cls):
        db = Database()
        return db.fetch_all(
            """SELECT rv.id, rv.text, rv.rating, rv.created_at,
                      u.login, e.name AS event_name
               FROM review rv
               JOIN request r ON rv.request_id = r.id
               JOIN "user" u ON r.user_id = u.id
               JOIN event e ON r.event_id = e.id
               ORDER BY rv.created_at DESC"""
        )

    @classmethod
    def delete(cls, review_id: int) -> bool:
        db = Database()
        db.execute("DELETE FROM review WHERE id = %s", (review_id,))
        return True
```

---

## Шаг 6. Обновление `app.py` (10 мин)

Откройте `backend/app.py` и **замените** содержимое:

```python
import os
from flask import Flask, render_template
from dotenv import load_dotenv

load_dotenv()

app = Flask(
    __name__,
    template_folder='../frontend/templates',
    static_folder='../frontend/static'
)

# Безопасность сессий
app.secret_key = os.environ.get('FLASK_SECRET_KEY', 'dev-fallback-key')
app.config.update(
    SESSION_COOKIE_SECURE=False,
    SESSION_COOKIE_HTTPONLY=True,
    SESSION_COOKIE_SAMESITE='Lax',
    PERMANENT_SESSION_LIFETIME=1800,
    SESSION_COOKIE_NAME='conf_session',
)

# Blueprints
from backend.routes.auth_routes import auth_bp
from backend.routes.request_routes import request_bp
from backend.routes.admin_routes import admin_bp
from backend.routes.review_routes import review_bp
from backend.models.user import User

app.register_blueprint(auth_bp, url_prefix='/api/auth')
app.register_blueprint(request_bp)
app.register_blueprint(admin_bp)
app.register_blueprint(review_bp)


@app.route('/')
def index():
    return render_template('login.html')


def init_admin():
    """Создание администратора по умолчанию."""
    admin = User(
        login='Conf2027',
        password='Demo77',
        email='admin@conf2027.ru',
        phone='8(999)000-00-00',
        role='admin'
    )
    if not admin.login_exists():
        ok, result = admin.save()
        if ok:
            print(f"✓ Администратор Conf2027 создан (id={result})")
        else:
            print(f"✗ Ошибка создания админа: {result}")
    else:
        print("✓ Администратор уже существует")


if __name__ == '__main__':
    init_admin()
    app.run(
        debug=os.environ.get('FLASK_DEBUG', 'False') == 'True',
        host='127.0.0.1',
        port=5000
    )
```

---

## Шаг 7. Тест на SQL-инъекцию (PostgreSQL) (10 мин)

Создайте `test_pg_injection.py`:

```python
from backend.database import Database

db = Database()

attacks = [
    "' OR '1'='1",
    "'; DROP TABLE \"user\"; --",
    "admin'--",
    "' UNION SELECT * FROM \"user\" --",
]

print("=" * 60)
print("ТЕСТ SQL-ИНЪЕКЦИЙ (PostgreSQL)")
print("=" * 60)

for attack in attacks:
    result = db.fetch_all(
        'SELECT * FROM "user" WHERE login = %s',
        (attack,)
    )
    status = "✅ ЗАЩИЩЕНО" if len(result) == 0 else "❌ УЯЗВИМО"
    print(f"{status}: '{attack}' → {len(result)} записей")

# Проверка, что таблица user не удалена
check = db.fetch_all(
    """SELECT table_name FROM information_schema.tables
       WHERE table_schema='public' AND table_name='user'"""
)
print("=" * 60)
if check:
    print("✅ Таблица user существует — инъекции не сработали")
else:
    print("❌ Таблица user удалена!")
print("=" * 60)
```

**Запустите:**

```bash
python test_pg_injection.py
```

**Ожидаемый результат:**

```
============================================================
ТЕСТ SQL-ИНЪЕКЦИЙ (PostgreSQL)
============================================================
✅ ЗАЩИЩЕНО: '' OR '1'='1' → 0 записей
✅ ЗАЩИЩЕНО: ''; DROP TABLE "user"; --' → 0 записей
✅ ЗАЩИЩЕНО: 'admin'--' → 0 записей
✅ ЗАЩИЩЕНО: '' UNION SELECT * FROM "user" --' → 0 записей
============================================================
✅ Таблица user существует — инъекции не сработали
============================================================
```

---

## Шаг 8. Наполнение БД мероприятиями (15 мин)

Обновите `seed_events.py` (теперь работает через PostgreSQL):

```python
from backend.models.event import Event
from backend.database import Database


def seed_events():
    db = Database()
    existing = db.fetch_one("SELECT COUNT(*) AS cnt FROM event")
    if existing['cnt'] > 0:
        print(f"✓ Мероприятия уже есть: {existing['cnt']}")
        return

    events = [
        {'name': 'Аудитория №101', 'date': '2027-06-15',
         'place': 'Аудитория', 'description': 'Большая аудитория на 100 человек'},
        {'name': 'Коворкинг «Прогресс»', 'date': '2027-06-20',
         'place': 'Коворкинг', 'description': 'Пространство для командной работы'},
        {'name': 'Кинозал «Октябрь»', 'date': '2027-07-01',
         'place': 'Кинозал', 'description': 'Dolby Atmos'},
        {'name': 'Конференц-зал «Восток»', 'date': '2027-07-10',
         'place': 'Аудитория', 'description': 'Видеоконференцсвязь'},
        {'name': 'Коворкинг «Цифра»', 'date': '2027-08-05',
         'place': 'Коворкинг', 'description': 'Для IT-специалистов'},
    ]

    for data in events:
        Event(**data).save()
        print(f"  + {data['name']}")
    print(f"✓ Загружено {len(events)} мероприятий")


if __name__ == '__main__':
    seed_events()
```

**Запустите:**

```bash
python seed_events.py
```

**Проверка через psql:**

```bash
PGPASSWORD='Conf2027!' psql -U conference_user -d conference_db -h 127.0.0.1 \
  -c "SELECT id, name, place FROM event;"
```

---

## Шаг 9. Запуск и тестирование (20 мин)

### 9.1. Запуск приложения

```bash
python -m backend.app
```

**Ожидаемый вывод:**
```
✓ Администратор Conf2027 создан (id=1)
 * Running on http://127.0.0.1:5000
```

### 9.2. Полный сценарий тестирования

| № | Действие | Ожидаемый результат |
|---|----------|---------------------|
| 1 | Регистрация `testuser2027 / password123` | ✅ Пользователь создан |
| 2 | Вход под `testuser2027` | ✅ Dashboard |
| 3 | Создание заявки (Аудитория №101) | ✅ Статус «Новая» |
| 4 | Вход под `Conf2027 / Demo77` | ✅ Админ-панель |
| 5 | Смена статуса: Новая → Мероприятие назначено | ✅ Обновлено |
| 6 | Смена статуса: Мероприятие назначено → Завершено | ✅ Обновлено |
| 7 | Вход под `testuser2027` | ✅ Dashboard |
| 8 | Кнопка «Оставить отзыв» | ✅ Форма отзыва |
| 9 | Оценка ★★★★★ + текст | ✅ Отзыв сохранён |
| 10 | Вход под `Conf2027` → админ-панель | ✅ Отзыв отображается |

### 9.3. Проверка данных через psql

```bash
PGPASSWORD='Conf2027!' psql -U conference_user -d conference_db -h 127.0.0.1 <<EOF
SELECT '--- Пользователи ---';
SELECT id, login, role FROM "user";
SELECT '--- Заявки ---';
SELECT r.id, u.login, e.name, s.name AS status
FROM request r
JOIN "user" u ON r.user_id = u.id
JOIN event e ON r.event_id = e.id
JOIN status s ON r.status_id = s.id;
SELECT '--- Отзывы ---';
SELECT rv.id, u.login, rv.rating, rv.text
FROM review rv
JOIN request r ON rv.request_id = r.id
JOIN "user" u ON r.user_id = u.id;
EOF
```

### 9.4. Проверка CRUD через psql

```sql
-- Create
INSERT INTO event (name, date, place, description)
VALUES ('Тестовый зал', '2027-12-01', 'Аудитория', 'Для теста')
RETURNING id;

-- Read
SELECT * FROM event ORDER BY id DESC LIMIT 3;

-- Update
UPDATE event SET description = 'Обновлено' WHERE id = 6;

-- Delete
DELETE FROM event WHERE id = 6;
```

---


# Полный проект «Конференции.РФ» на PostgreSQL

---

## Структура проекта

```
conference_portal/
├── backend/
│   ├── __init__.py
│   ├── app.py
│   ├── database.py
│   ├── auth/
│   │   ├── __init__.py
│   │   └── decorators.py
│   ├── models/
│   │   ├── __init__.py
│   │   ├── base_model.py
│   │   ├── user.py
│   │   ├── event.py
│   │   ├── request.py
│   │   ├── status.py
│   │   └── review.py
│   ├── validators/
│   │   ├── __init__.py
│   │   └── validators.py
│   └── routes/
│       ├── __init__.py
│       ├── auth_routes.py
│       ├── request_routes.py
│       ├── admin_routes.py
│       └── review_routes.py
├── frontend/
│   ├── static/
│   │   ├── css/
│   │   │   └── style.css
│   │   └── js/
│   │       ├── register.js
│   │       ├── login.js
│   │       └── request.js
│   └── templates/
│       ├── register.html
│       ├── login.html
│       ├── dashboard.html
│       ├── create_request.html
│       ├── admin.html
│       └── review.html
├── schema_postgres.sql
├── seed_events.py
├── test_pg_injection.py
├── requirements.txt
├── .env
├── .gitignore
└── README.md
```

---

## 1. Корневые файлы

### 1.1. `requirements.txt`

```
Flask==2.3.3
Werkzeug==2.3.7
psycopg2-binary==2.9.9
python-dotenv==1.0.0
```

### 1.2. `.env`

```env
DATABASE_URL=postgresql://conference_user:Conf2027!@127.0.0.1:5432/conference_db
FLASK_SECRET_KEY=a3f8b2c1d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1
FLASK_DEBUG=False
```

### 1.3. `.gitignore`

```
venv/
__pycache__/
*.pyc
*.pyo
.env
*.log
.pytest_cache/
```

### 1.4. `schema_postgres.sql`

```sql
-- =====================================================================
-- Схема базы данных портала «Конференции.РФ» (PostgreSQL)
-- =====================================================================

-- Таблица «Пользователь»
CREATE TABLE IF NOT EXISTS "user" (
    id SERIAL PRIMARY KEY,
    login VARCHAR(50) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    phone VARCHAR(18) NOT NULL,
    role VARCHAR(20) NOT NULL DEFAULT 'user'
);

-- Таблица «Мероприятие»
CREATE TABLE IF NOT EXISTS event (
    id SERIAL PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    date DATE NOT NULL,
    place VARCHAR(200) NOT NULL,
    description TEXT
);

-- Таблица «Статус»
CREATE TABLE IF NOT EXISTS status (
    id SERIAL PRIMARY KEY,
    name VARCHAR(50) UNIQUE NOT NULL
);

-- Наполнение справочника статусов
INSERT INTO status (name) VALUES
    ('Новая'),
    ('Мероприятие назначено'),
    ('Завершено')
ON CONFLICT (name) DO NOTHING;

-- Таблица «Заявка»
CREATE TABLE IF NOT EXISTS request (
    id SERIAL PRIMARY KEY,
    user_id INTEGER NOT NULL REFERENCES "user"(id) ON DELETE CASCADE,
    event_id INTEGER NOT NULL REFERENCES event(id) ON DELETE CASCADE,
    status_id INTEGER NOT NULL DEFAULT 1 REFERENCES status(id),
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- Индексы для ускорения запросов
CREATE INDEX IF NOT EXISTS idx_request_user ON request(user_id);
CREATE INDEX IF NOT EXISTS idx_request_status ON request(status_id);
CREATE INDEX IF NOT EXISTS idx_request_event ON request(event_id);

-- Таблица «Отзыв»
CREATE TABLE IF NOT EXISTS review (
    id SERIAL PRIMARY KEY,
    request_id INTEGER UNIQUE NOT NULL REFERENCES request(id) ON DELETE CASCADE,
    text TEXT NOT NULL,
    rating INTEGER CHECK (rating BETWEEN 1 AND 5),
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

### 1.5. `seed_events.py`

```python
"""Наполнение БД тестовыми мероприятиями."""
from backend.models.event import Event
from backend.database import Database


def seed_events():
    db = Database()
    existing = db.fetch_one("SELECT COUNT(*) AS cnt FROM event")
    if existing and existing['cnt'] > 0:
        print(f"✓ Мероприятия уже есть: {existing['cnt']} записей")
        return

    events = [
        {
            'name': 'Аудитория №101',
            'date': '2027-06-15',
            'place': 'Аудитория',
            'description': 'Большая аудитория на 100 человек'
        },
        {
            'name': 'Коворкинг «Прогресс»',
            'date': '2027-06-20',
            'place': 'Коворкинг',
            'description': 'Современное пространство для командной работы'
        },
        {
            'name': 'Кинозал «Октябрь»',
            'date': '2027-07-01',
            'place': 'Кинозал',
            'description': 'Зал с проектором и звуком Dolby Atmos'
        },
        {
            'name': 'Конференц-зал «Восток»',
            'date': '2027-07-10',
            'place': 'Аудитория',
            'description': 'Зал с видеоконференцсвязью'
        },
        {
            'name': 'Коворкинг «Цифра»',
            'date': '2027-08-05',
            'place': 'Коворкинг',
            'description': 'Пространство для IT-специалистов'
        },
    ]

    for data in events:
        Event(**data).save()
        print(f"  + Добавлено: {data['name']}")

    print(f"✓ Загружено {len(events)} мероприятий")


if __name__ == '__main__':
    seed_events()
```

### 1.6. `test_pg_injection.py`

```python
"""Тест защиты от SQL-инъекций в PostgreSQL."""
from backend.database import Database

db = Database()

attacks = [
    "' OR '1'='1",
    "'; DROP TABLE \"user\"; --",
    "admin'--",
    "' UNION SELECT * FROM \"user\" --",
    "1' OR '1'='1' /*",
]

print("=" * 60)
print("ТЕСТ SQL-ИНЪЕКЦИЙ (PostgreSQL)")
print("=" * 60)

for attack in attacks:
    result = db.fetch_all(
        'SELECT * FROM "user" WHERE login = %s',
        (attack,)
    )
    status = "✅ ЗАЩИЩЕНО" if len(result) == 0 else "❌ УЯЗВИМО"
    print(f"{status}: '{attack}' → {len(result)} записей")

# Проверка, что таблица user не удалена
check = db.fetch_all(
    """SELECT table_name FROM information_schema.tables
       WHERE table_schema='public' AND table_name='user'"""
)
print("=" * 60)
if check:
    print("✅ Таблица user существует — инъекции не сработали")
else:
    print("❌ Таблица user удалена!")
print("=" * 60)
```

### 1.7. `README.md`

```markdown
# Портал «Конференции.РФ»

Информационная система для бронирования помещений под Всероссийские конференции.

## Технологии

- **Backend:** Python 3.8, Flask 2.3.3
- **Database:** PostgreSQL 12+
- **Frontend:** HTML5, CSS3, JavaScript (fetch API)
- **Security:** Werkzeug (pbkdf2:sha256), prepared statements

## Быстрый старт

### 1. Установка PostgreSQL (ROSA Linux)

```bash
sudo dnf install postgresql-server postgresql-contrib
sudo su - postgres
initdb -D /var/lib/pgsql/data
exit
sudo systemctl start postgresql12.service
sudo systemctl enable postgresql12.service
```

### 2. Создание БД

```bash
sudo -u postgres psql <<EOF
CREATE USER conference_user WITH PASSWORD 'Conf2027!';
CREATE DATABASE conference_db OWNER conference_user;
GRANT ALL PRIVILEGES ON DATABASE conference_db TO conference_user;
EOF
```

### 3. Применение схемы

```bash
PGPASSWORD='Conf2027!' psql -U conference_user -d conference_db -h 127.0.0.1 -f schema_postgres.sql
```

### 4. Виртуальное окружение

```bash
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

### 5. Наполнение БД

```bash
python seed_events.py
```

### 6. Запуск

```bash
python -m backend.app
```

Откройте http://127.0.0.1:5000

## Учётные данные администратора

- **Логин:** `Conf2027`
- **Пароль:** `Demo77`

## Тестирование

```bash
python test_pg_injection.py
```
```

---

## 2. Пакет `backend/`

### 2.1. `backend/__init__.py`

```python
# Пакет backend
```

### 2.2. `backend/app.py`

```python
"""Точка входа Flask-приложения."""
import os
from flask import Flask, render_template
from dotenv import load_dotenv

load_dotenv()

app = Flask(
    __name__,
    template_folder='../frontend/templates',
    static_folder='../frontend/static'
)

# =====================================================================
# БЕЗОПАСНАЯ НАСТРОЙКА СЕССИЙ И COOKIES
# =====================================================================
app.secret_key = os.environ.get(
    'FLASK_SECRET_KEY',
    'dev-fallback-key-change-in-production-32bytes-min'
)

app.config.update(
    SESSION_COOKIE_SECURE=False,       # True для production (HTTPS)
    SESSION_COOKIE_HTTPONLY=True,      # Защита от XSS
    SESSION_COOKIE_SAMESITE='Lax',     # Защита от CSRF
    PERMANENT_SESSION_LIFETIME=1800,   # 30 минут
    SESSION_COOKIE_NAME='conf_session',
)

# =====================================================================
# РЕГИСТРАЦИЯ BLUEPRINTS
# =====================================================================
from backend.routes.auth_routes import auth_bp
from backend.routes.request_routes import request_bp
from backend.routes.admin_routes import admin_bp
from backend.routes.review_routes import review_bp
from backend.models.user import User

app.register_blueprint(auth_bp, url_prefix='/api/auth')
app.register_blueprint(request_bp)
app.register_blueprint(admin_bp)
app.register_blueprint(review_bp)


@app.route('/')
def index():
    return render_template('login.html')


def init_admin():
    """Создание администратора по умолчанию."""
    admin = User(
        login='Conf2027',
        password='Demo77',
        email='admin@conf2027.ru',
        phone='8(999)000-00-00',
        role='admin'
    )
    if not admin.login_exists():
        ok, result = admin.save()
        if ok:
            print(f"✓ Администратор Conf2027 создан (id={result})")
        else:
            print(f"✗ Ошибка создания администратора: {result}")
    else:
        print("✓ Администратор уже существует")


if __name__ == '__main__':
    init_admin()
    app.run(
        debug=os.environ.get('FLASK_DEBUG', 'False') == 'True',
        host='127.0.0.1',
        port=5000
    )
```

### 2.3. `backend/database.py`

```python
"""Singleton-класс для работы с PostgreSQL."""
import os
import psycopg2
import psycopg2.extras
from dotenv import load_dotenv

load_dotenv()


class Database:
    """Singleton-класс для работы с PostgreSQL."""
    _instance = None

    def __new__(cls, db_url=None):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
            cls._instance._initialized = False
        return cls._instance

    def __init__(self, db_url=None):
        if self._initialized:
            return
        if db_url is None:
            db_url = os.environ.get(
                'DATABASE_URL',
                'postgresql://conference_user:Conf2027!@127.0.0.1:5432/conference_db'
            )
        self.db_url = db_url
        self._initialized = True

    def get_connection(self):
        """Создать новое подключение к PostgreSQL."""
        return psycopg2.connect(self.db_url)

    def execute(self, query, params=()):
        """
        Выполнить INSERT / UPDATE / DELETE.
        Для INSERT с RETURNING id возвращает id.
        """
        with self.get_connection() as conn:
            with conn.cursor() as cursor:
                cursor.execute(query, params)
                conn.commit()
                try:
                    result = cursor.fetchone()
                    return result[0] if result else None
                except psycopg2.ProgrammingError:
                    return None

    def fetch_one(self, query, params=()):
        """Получить одну запись как словарь."""
        with self.get_connection() as conn:
            with conn.cursor(
                cursor_factory=psycopg2.extras.RealDictCursor
            ) as cursor:
                cursor.execute(query, params)
                row = cursor.fetchone()
                return dict(row) if row else None

    def fetch_all(self, query, params=()):
        """Получить все записи как список словарей."""
        with self.get_connection() as conn:
            with conn.cursor(
                cursor_factory=psycopg2.extras.RealDictCursor
            ) as cursor:
                cursor.execute(query, params)
                return [dict(row) for row in cursor.fetchall()]

    def execute_script(self, script_path: str):
        """Выполнить SQL-скрипт из файла."""
        with open(script_path, 'r', encoding='utf-8') as f:
            sql = f.read()
        with self.get_connection() as conn:
            with conn.cursor() as cursor:
                cursor.execute(sql)
                conn.commit()
```

### 2.4. `backend/auth/__init__.py`

```python
# Пакет auth
```

### 2.5. `backend/auth/decorators.py`

```python
"""Декораторы для проверки прав доступа."""
from functools import wraps
from flask import session, jsonify, redirect, request


def login_required(f):
    """Декоратор: требует авторизации."""
    @wraps(f)
    def decorated_function(*args, **kwargs):
        if 'user_id' not in session:
            if request.path.startswith('/api/'):
                return jsonify({
                    'success': False,
                    'message': 'Требуется авторизация'
                }), 401
            return redirect('/api/auth/login')
        return f(*args, **kwargs)
    return decorated_function


def admin_required(f):
    """Декоратор: требует прав администратора."""
    @wraps(f)
    def decorated_function(*args, **kwargs):
        if 'user_id' not in session:
            return jsonify({
                'success': False,
                'message': 'Требуется авторизация'
            }), 401
        if session.get('role') != 'admin':
            return jsonify({
                'success': False,
                'message': 'Доступ запрещён. Требуются права администратора'
            }), 403
        return f(*args, **kwargs)
    return decorated_function


def participant_required(f):
    """Декоратор: требует роли участника (user) или администратора."""
    @wraps(f)
    def decorated_function(*args, **kwargs):
        if 'user_id' not in session:
            return jsonify({
                'success': False,
                'message': 'Требуется авторизация'
            }), 401
        if session.get('role') not in ('user', 'admin'):
            return jsonify({
                'success': False,
                'message': 'Доступ запрещён. Требуется роль участника'
            }), 403
        return f(*args, **kwargs)
    return decorated_function
```

---

## 3. Пакет `backend/models/`

### 3.1. `backend/models/__init__.py`

```python
# Пакет models
```

### 3.2. `backend/models/base_model.py`

```python
"""Базовый класс для всех моделей."""
from backend.database import Database


class BaseModel:
    """Базовый класс для всех моделей. Демонстрирует наследование."""
    table_name = None

    def __init__(self, **kwargs):
        self.db = Database()
        for key, value in kwargs.items():
            setattr(self, key, value)

    def save(self):
        raise NotImplementedError("Метод save() должен быть реализован")

    @classmethod
    def get_all(cls):
        db = Database()
        return db.fetch_all(f'SELECT * FROM {cls.table_name}')

    @classmethod
    def get_by_id(cls, record_id: int):
        db = Database()
        return db.fetch_one(
            f'SELECT * FROM {cls.table_name} WHERE id = %s',
            (record_id,)
        )
```

### 3.3. `backend/models/user.py`

```python
"""Модель «Пользователь»."""
from backend.models.base_model import BaseModel
from backend.validators.validators import Validator
from backend.database import Database
from werkzeug.security import generate_password_hash, check_password_hash


class User(BaseModel):
    table_name = '"user"'

    def __init__(self, login=None, password=None, email=None,
                 phone=None, role='user', **kwargs):
        super().__init__(**kwargs)
        self.login = login
        self.password = password
        self.email = email
        self.phone = phone
        self.role = role

    def validate(self) -> tuple:
        """Полная валидация полей пользователя."""
        checks = [
            Validator.validate_login(self.login),
            Validator.validate_password(self.password),
            Validator.validate_email(self.email),
            Validator.validate_phone(self.phone),
        ]
        for ok, msg in checks:
            if not ok:
                return False, msg
        return True, ""

    def login_exists(self) -> bool:
        row = self.db.fetch_one(
            'SELECT id FROM "user" WHERE login = %s', (self.login,)
        )
        return row is not None

    def email_exists(self) -> bool:
        row = self.db.fetch_one(
            'SELECT id FROM "user" WHERE email = %s', (self.email,)
        )
        return row is not None

    def save(self) -> tuple:
        """Сохранение нового пользователя с хешированием пароля."""
        ok, msg = self.validate()
        if not ok:
            return False, msg
        if self.login_exists():
            return False, "Пользователь с таким логином уже существует"
        if self.email_exists():
            return False, "Пользователь с таким email уже существует"

        password_hash = generate_password_hash(
            self.password,
            method='pbkdf2:sha256:600000',
            salt_length=16
        )
        user_id = self.db.execute(
            '''INSERT INTO "user" (login, password_hash, email, phone, role)
               VALUES (%s, %s, %s, %s, %s)
               RETURNING id''',
            (self.login, password_hash, self.email, self.phone, self.role)
        )
        self.id = user_id
        return True, user_id

    @staticmethod
    def authenticate(login: str, password: str) -> dict:
        """Аутентификация: сравнение пароля с хешем."""
        db = Database()
        user = db.fetch_one(
            'SELECT * FROM "user" WHERE login = %s', (login,)
        )
        if user and check_password_hash(user['password_hash'], password):
            return user
        return None
```

### 3.4. `backend/models/event.py`

```python
"""Модель «Мероприятие»."""
from backend.models.base_model import BaseModel


class Event(BaseModel):
    table_name = "event"

    def __init__(self, name=None, date=None, place=None,
                 description=None, **kwargs):
        super().__init__(**kwargs)
        self.name = name
        self.date = date
        self.place = place
        self.description = description

    def save(self) -> int:
        event_id = self.db.execute(
            """INSERT INTO event (name, date, place, description)
               VALUES (%s, %s, %s, %s)
               RETURNING id""",
            (self.name, self.date, self.place, self.description)
        )
        self.id = event_id
        return event_id
```

### 3.5. `backend/models/status.py`

```python
"""Модель «Статус»."""
from backend.models.base_model import BaseModel


class Status(BaseModel):
    table_name = "status"

    def __init__(self, name=None, **kwargs):
        super().__init__(**kwargs)
        self.name = name
```

### 3.6. `backend/models/request.py`

```python
"""Модель «Заявка»."""
from backend.database import Database
from backend.models.base_model import BaseModel


class Request(BaseModel):
    table_name = "request"

    def __init__(self, user_id=None, event_id=None,
                 status_id=1, **kwargs):
        super().__init__(**kwargs)
        self.user_id = user_id
        self.event_id = event_id
        self.status_id = status_id

    def save(self) -> int:
        request_id = self.db.execute(
            """INSERT INTO request (user_id, event_id, status_id)
               VALUES (%s, %s, %s)
               RETURNING id""",
            (self.user_id, self.event_id, self.status_id)
        )
        self.id = request_id
        return request_id

    @classmethod
    def get_by_user(cls, user_id: int):
        """Все заявки конкретного пользователя с деталями."""
        db = Database()
        return db.fetch_all(
            """SELECT r.id, e.name AS event_name, e.date,
                      e.place, s.name AS status_name,
                      r.status_id, r.created_at
               FROM request r
               JOIN event e ON r.event_id = e.id
               JOIN status s ON r.status_id = s.id
               WHERE r.user_id = %s
               ORDER BY r.created_at DESC""",
            (user_id,)
        )

    @classmethod
    def get_all_with_details(cls):
        """Все заявки всех пользователей (для администратора)."""
        db = Database()
        return db.fetch_all(
            """SELECT r.id, u.login, u.email, e.name AS event_name,
                      e.date, e.place, s.name AS status_name,
                      r.status_id, r.created_at
               FROM request r
               JOIN "user" u ON r.user_id = u.id
               JOIN event e ON r.event_id = e.id
               JOIN status s ON r.status_id = s.id
               ORDER BY r.created_at DESC"""
        )

    @classmethod
    def get_by_id(cls, request_id: int):
        """Одна заявка с деталями."""
        db = Database()
        return db.fetch_one(
            """SELECT r.id, r.user_id, r.event_id, r.status_id,
                      u.login, e.name AS event_name, e.date, e.place,
                      s.name AS status_name, r.created_at
               FROM request r
               JOIN "user" u ON r.user_id = u.id
               JOIN event e ON r.event_id = e.id
               JOIN status s ON r.status_id = s.id
               WHERE r.id = %s""",
            (request_id,)
        )

    @classmethod
    def update_status(cls, request_id: int, status_id: int) -> bool:
        db = Database()
        db.execute(
            "UPDATE request SET status_id = %s WHERE id = %s",
            (status_id, request_id)
        )
        return True

    @classmethod
    def delete(cls, request_id: int) -> bool:
        db = Database()
        db.execute("DELETE FROM request WHERE id = %s", (request_id,))
        return True

    @classmethod
    def can_be_reviewed(cls, request_id: int) -> bool:
        db = Database()
        row = db.fetch_one(
            "SELECT status_id FROM request WHERE id = %s",
            (request_id,)
        )
        return row and row['status_id'] == 3
```

### 3.7. `backend/models/review.py`

```python
"""Модель «Отзыв»."""
from backend.database import Database
from backend.models.base_model import BaseModel


class Review(BaseModel):
    table_name = "review"

    def __init__(self, request_id=None, text=None,
                 rating=None, **kwargs):
        super().__init__(**kwargs)
        self.request_id = request_id
        self.text = text
        self.rating = rating

    def validate(self) -> tuple:
        if not self.text or len(self.text.strip()) < 5:
            return False, "Текст отзыва должен содержать минимум 5 символов"
        if len(self.text) > 2000:
            return False, "Текст отзыва слишком длинный (максимум 2000)"
        if not self.rating or not (1 <= int(self.rating) <= 5):
            return False, "Оценка должна быть от 1 до 5"
        return True, ""

    def save(self) -> tuple:
        """Сохранение отзыва с валидацией."""
        ok, msg = self.validate()
        if not ok:
            return False, msg

        from backend.models.request import Request
        req = Request.get_by_id(self.request_id)
        if not req:
            return False, "Заявка не найдена"
        if req['status_id'] != 3:
            return False, "Отзыв можно оставить только после завершения"
        if self.exists_for_request(self.request_id):
            return False, "Отзыв для этой заявки уже оставлен"

        review_id = self.db.execute(
            """INSERT INTO review (request_id, text, rating)
               VALUES (%s, %s, %s)
               RETURNING id""",
            (self.request_id, self.text.strip(), int(self.rating))
        )
        self.id = review_id
        return True, review_id

    @staticmethod
    def exists_for_request(request_id: int) -> bool:
        db = Database()
        row = db.fetch_one(
            "SELECT id FROM review WHERE request_id = %s",
            (request_id,)
        )
        return row is not None

    @classmethod
    def get_by_request(cls, request_id: int):
        db = Database()
        return db.fetch_one(
            """SELECT id, text, rating, created_at
               FROM review WHERE request_id = %s""",
            (request_id,)
        )

    @classmethod
    def get_all_with_details(cls):
        db = Database()
        return db.fetch_all(
            """SELECT rv.id, rv.text, rv.rating, rv.created_at,
                      u.login, e.name AS event_name
               FROM review rv
               JOIN request r ON rv.request_id = r.id
               JOIN "user" u ON r.user_id = u.id
               JOIN event e ON r.event_id = e.id
               ORDER BY rv.created_at DESC"""
        )

    @classmethod
    def delete(cls, review_id: int) -> bool:
        db = Database()
        db.execute("DELETE FROM review WHERE id = %s", (review_id,))
        return True
```

---

## 4. Пакет `backend/validators/`

### 4.1. `backend/validators/__init__.py`

```python
# Пакет validators
```

### 4.2. `backend/validators/validators.py`

```python
"""Валидация пользовательских данных."""
import re


class Validator:
    """Класс для валидации данных."""

    LOGIN_PATTERN = re.compile(r'^[A-Za-z0-9]{6,50}$')
    EMAIL_PATTERN = re.compile(
        r'^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$'
    )
    PHONE_PATTERN = re.compile(r'^8\(\d{3}\)\d{3}-\d{2}-\d{2}$')
    FIO_PATTERN = re.compile(r'^[А-Яа-яЁё\s]{2,150}$')

    @staticmethod
    def sanitize_string(value: str, max_length: int = 255) -> str:
        """Очистка строки от опасных символов."""
        if not value:
            return ""
        cleaned = re.sub(r'[\x00-\x1f\x7f]', '', value)
        return cleaned[:max_length].strip()

    @staticmethod
    def validate_login(login: str) -> tuple:
        login = Validator.sanitize_string(login, 50)
        if not Validator.LOGIN_PATTERN.match(login):
            return False, "Логин: от 6 до 50 символов, латиница и цифры"
        return True, ""

    @staticmethod
    def validate_password(password: str) -> tuple:
        if not password or len(password) < 8:
            return False, "Пароль должен содержать минимум 8 символов"
        if len(password) > 128:
            return False, "Пароль слишком длинный (максимум 128)"
        return True, ""

    @staticmethod
    def validate_email(email: str) -> tuple:
        email = Validator.sanitize_string(email, 100)
        if not Validator.EMAIL_PATTERN.match(email):
            return False, "Некорректный email"
        return True, ""

    @staticmethod
    def validate_phone(phone: str) -> tuple:
        phone = Validator.sanitize_string(phone, 18)
        if not Validator.PHONE_PATTERN.match(phone):
            return False, "Телефон в формате 8(XXX)XXX-XX-XX"
        return True, ""

    @staticmethod
    def validate_fio(fio: str) -> tuple:
        fio = Validator.sanitize_string(fio, 150)
        if not Validator.FIO_PATTERN.match(fio):
            return False, "ФИО: только кириллица и пробелы"
        return True, ""

    @staticmethod
    def validate_rating(rating) -> tuple:
        try:
            rating = int(rating)
            if 1 <= rating <= 5:
                return True, ""
            return False, "Оценка должна быть от 1 до 5"
        except (ValueError, TypeError):
            return False, "Оценка должна быть числом"
```

---

## 5. Пакет `backend/routes/`

### 5.1. `backend/routes/__init__.py`

```python
# Пакет routes
```

### 5.2. `backend/routes/auth_routes.py`

```python
"""Маршруты регистрации и авторизации."""
from flask import Blueprint, request, jsonify, render_template, session
from backend.models.user import User

auth_bp = Blueprint('auth', __name__)


@auth_bp.route('/register', methods=['GET', 'POST'])
def register():
    if request.method == 'GET':
        return render_template('register.html')

    data = request.get_json()
    user = User(
        login=data.get('login', '').strip(),
        password=data.get('password', ''),
        email=data.get('email', '').strip(),
        phone=data.get('phone', '').strip(),
        role='user'
    )
    ok, result = user.save()
    if not ok:
        return jsonify({'success': False, 'message': result}), 400
    return jsonify({'success': True, 'user_id': result}), 201


@auth_bp.route('/login', methods=['GET', 'POST'])
def login():
    if request.method == 'GET':
        return render_template('login.html')

    data = request.get_json()
    user = User.authenticate(
        data.get('login', ''),
        data.get('password', '')
    )
    if not user:
        return jsonify({
            'success': False,
            'message': 'Неверный логин или пароль'
        }), 401

    # Очистка старой сессии (защита от fixation)
    session.clear()
    session['user_id'] = user['id']
    session['role'] = user['role']
    session['login'] = user['login']
    session.permanent = True

    return jsonify({
        'success': True,
        'role': user['role'],
        'redirect': '/admin' if user['role'] == 'admin' else '/dashboard'
    }), 200


@auth_bp.route('/logout', methods=['POST'])
def logout():
    session.clear()
    return jsonify({'success': True}), 200
```

### 5.3. `backend/routes/request_routes.py`

```python
"""Маршруты заявок."""
from flask import Blueprint, request, jsonify, render_template, session
from backend.auth.decorators import login_required, participant_required
from backend.models.request import Request
from backend.models.event import Event

request_bp = Blueprint('requests', __name__)


@request_bp.route('/dashboard')
@login_required
def dashboard():
    return render_template('dashboard.html')


@request_bp.route('/api/requests', methods=['GET'])
@login_required
def get_my_requests():
    data = Request.get_by_user(session['user_id'])
    return jsonify({'success': True, 'requests': data}), 200


@request_bp.route('/create_request', methods=['GET', 'POST'])
@participant_required
def create_request():
    if request.method == 'GET':
        events = Event.get_all()
        return render_template('create_request.html', events=events)

    data = request.get_json()
    event_id = data.get('event_id')

    if not event_id:
        return jsonify({
            'success': False,
            'message': 'Выберите помещение'
        }), 400

    new_request = Request(
        user_id=session['user_id'],
        event_id=int(event_id),
        status_id=1
    )
    request_id = new_request.save()
    return jsonify({
        'success': True,
        'request_id': request_id,
        'message': 'Заявка отправлена администратору'
    }), 201


@request_bp.route('/api/requests/<int:request_id>', methods=['DELETE'])
@login_required
def delete_my_request(request_id):
    """Удаление своей заявки (только если статус «Новая»)."""
    req = Request.get_by_id(request_id)
    if not req:
        return jsonify({'success': False, 'message': 'Заявка не найдена'}), 404
    if req['user_id'] != session['user_id']:
        return jsonify({'success': False, 'message': 'Доступ запрещён'}), 403
    if req['status_id'] != 1:
        return jsonify({
            'success': False,
            'message': 'Можно удалить только заявку со статусом «Новая»'
        }), 400

    Request.delete(request_id)
    return jsonify({'success': True}), 200
```

### 5.4. `backend/routes/admin_routes.py`

```python
"""Маршруты администратора."""
from flask import Blueprint, request, jsonify, render_template
from backend.auth.decorators import admin_required
from backend.models.request import Request

admin_bp = Blueprint('admin', __name__)


@admin_bp.route('/admin')
@admin_required
def admin_panel():
    return render_template('admin.html')


@admin_bp.route('/api/admin/requests', methods=['GET'])
@admin_required
def get_all_requests():
    data = Request.get_all_with_details()
    return jsonify({'success': True, 'requests': data}), 200


@admin_bp.route('/api/admin/requests/<int:request_id>', methods=['PUT'])
@admin_required
def update_request_status(request_id):
    data = request.get_json()
    status_id = data.get('status_id')

    if status_id not in (1, 2, 3):
        return jsonify({
            'success': False,
            'message': 'Некорректный статус'
        }), 400

    req = Request.get_by_id(request_id)
    if not req:
        return jsonify({
            'success': False,
            'message': 'Заявка не найдена'
        }), 404

    current = req['status_id']
    new_status = int(status_id)

    allowed_transitions = {
        1: [2],   # Новая → Мероприятие назначено
        2: [3],   # Мероприятие назначено → Завершено
        3: [],    # Завершено — финальный статус
    }

    if new_status not in allowed_transitions.get(current, []):
        return jsonify({
            'success': False,
            'message': f'Недопустимый переход: {current} → {new_status}'
        }), 400

    Request.update_status(request_id, new_status)
    return jsonify({'success': True}), 200


@admin_bp.route('/api/admin/requests/<int:request_id>', methods=['DELETE'])
@admin_required
def delete_request(request_id):
    """Удаление заявки администратором."""
    req = Request.get_by_id(request_id)
    if not req:
        return jsonify({
            'success': False,
            'message': 'Заявка не найдена'
        }), 404

    Request.delete(request_id)
    return jsonify({'success': True}), 200
```

### 5.5. `backend/routes/review_routes.py`

```python
"""Маршруты отзывов."""
from flask import Blueprint, request, jsonify, render_template, session
from backend.auth.decorators import login_required, admin_required
from backend.models.review import Review
from backend.models.request import Request

review_bp = Blueprint('reviews', __name__)


@review_bp.route('/review/<int:request_id>', methods=['GET'])
@login_required
def review_form(request_id):
    """Форма оставления отзыва."""
    req = Request.get_by_id(request_id)
    if not req:
        return render_template('dashboard.html'), 404
    if req['user_id'] != session['user_id']:
        return render_template('dashboard.html'), 403
    if req['status_id'] != 3:
        return render_template('dashboard.html'), 400

    existing = Review.get_by_request(request_id)
    return render_template('review.html',
                           request_id=request_id,
                           req=req,
                           review=existing)


@review_bp.route('/api/review', methods=['POST'])
@login_required
def create_review():
    """Создание отзыва."""
    data = request.get_json()
    request_id = data.get('request_id')

    if not request_id:
        return jsonify({
            'success': False,
            'message': 'Не указан ID заявки'
        }), 400

    req = Request.get_by_id(int(request_id))
    if not req:
        return jsonify({
            'success': False,
            'message': 'Заявка не найдена'
        }), 404
    if req['user_id'] != session['user_id']:
        return jsonify({
            'success': False,
            'message': 'Доступ запрещён'
        }), 403

    review = Review(
        request_id=int(request_id),
        text=data.get('text', '').strip(),
        rating=data.get('rating')
    )
    ok, result = review.save()
    if not ok:
        return jsonify({'success': False, 'message': result}), 400

    return jsonify({
        'success': True,
        'review_id': result,
        'message': 'Спасибо за отзыв!'
    }), 201


@review_bp.route('/api/reviews', methods=['GET'])
@admin_required
def get_all_reviews():
    """Все отзывы (для админа)."""
    data = Review.get_all_with_details()
    return jsonify({'success': True, 'reviews': data}), 200


@review_bp.route('/api/reviews/<int:review_id>', methods=['DELETE'])
@admin_required
def delete_review(review_id):
    """Удаление отзыва администратором."""
    Review.delete(review_id)
    return jsonify({'success': True}), 200
```

---

## 6. Пакет `frontend/`

### 6.1. `frontend/templates/login.html`

```html
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <title>Вход — Конференции.РФ</title>
    <link rel="stylesheet" href="{{ url_for('static', filename='css/style.css') }}">
</head>
<body>
<div class="container">
    <h1>Авторизация</h1>
    <form id="loginForm">
        <input type="text" id="login" placeholder="Логин" required>
        <input type="password" id="password" placeholder="Пароль" required>
        <button type="submit">Войти</button>
    </form>
    <div id="message"></div>
    <p>Ещё не зарегистрированы? <a href="/api/auth/register">Регистрация</a></p>
</div>
<script src="{{ url_for('static', filename='js/login.js') }}"></script>
</body>
</html>
```

### 6.2. `frontend/templates/register.html`

```html
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <title>Регистрация — Конференции.РФ</title>
    <link rel="stylesheet" href="{{ url_for('static', filename='css/style.css') }}">
</head>
<body>
<div class="container">
    <h1>Регистрация</h1>
    <form id="registerForm">
        <input type="text" id="login" placeholder="Логин (латиница, ≥ 6 симв.)" required>
        <input type="password" id="password" placeholder="Пароль (≥ 8 симв.)" required>
        <input type="text" id="phone" placeholder="8(XXX)XXX-XX-XX" required>
        <input type="email" id="email" placeholder="Email" required>
        <button type="submit">Создать пользователя</button>
    </form>
    <div id="message"></div>
    <p>Уже зарегистрированы? <a href="/api/auth/login">Войти</a></p>
</div>
<script src="{{ url_for('static', filename='js/register.js') }}"></script>
</body>
</html>
```

### 6.3. `frontend/templates/dashboard.html`

```html
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <title>Мои заявки — Конференции.РФ</title>
    <link rel="stylesheet" href="{{ url_for('static', filename='css/style.css') }}">
</head>
<body>
<div class="container">
    <header>
        <h1>Мои заявки</h1>
        <div class="actions">
            <a href="/create_request" class="btn btn-primary">+ Создать заявку</a>
            <button onclick="logout()" class="btn btn-secondary">Выйти</button>
        </div>
    </header>

    <table id="requestsTable">
        <thead>
        <tr>
            <th>№</th>
            <th>Мероприятие</th>
            <th>Дата</th>
            <th>Место</th>
            <th>Статус</th>
            <th>Создана</th>
            <th>Действия</th>
        </tr>
        </thead>
        <tbody id="requestsBody">
        <tr><td colspan="7">Загрузка...</td></tr>
        </tbody>
    </table>
</div>

<script>
async function loadRequests() {
    const res = await fetch('/api/requests');
    const data = await res.json();
    const tbody = document.getElementById('requestsBody');
    tbody.innerHTML = '';

    if (!data.success || data.requests.length === 0) {
        tbody.innerHTML = '<tr><td colspan="7">Заявок пока нет</td></tr>';
        return;
    }

    data.requests.forEach(r => {
        let actions = '—';

        if (r.status_id === 1) {
            actions = `<button onclick="deleteRequest(${r.id})"
                       class="btn btn-danger btn-sm">Удалить</button>`;
        } else if (r.status_id === 3) {
            actions = `<a href="/review/${r.id}"
                       class="btn btn-success btn-sm">Оставить отзыв</a>`;
        }

        tbody.insertAdjacentHTML('beforeend', `
            <tr>
                <td>${r.id}</td>
                <td>${r.event_name}</td>
                <td>${r.date}</td>
                <td>${r.place}</td>
                <td><span class="status status-${r.status_id}">${r.status_name}</span></td>
                <td>${r.created_at}</td>
                <td>${actions}</td>
            </tr>
        `);
    });
}

async function deleteRequest(id) {
    if (!confirm('Удалить заявку?')) return;
    const res = await fetch(`/api/requests/${id}`, { method: 'DELETE' });
    const data = await res.json();
    if (data.success) {
        loadRequests();
    } else {
        alert(data.message);
    }
}

async function logout() {
    await fetch('/api/auth/logout', { method: 'POST' });
    window.location.href = '/api/auth/login';
}

loadRequests();
</script>
</body>
</html>
```

### 6.4. `frontend/templates/create_request.html`

```html
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <title>Создание заявки</title>
    <link rel="stylesheet" href="{{ url_for('static', filename='css/style.css') }}">
</head>
<body>
<div class="container">
    <h1>Формирование заявки</h1>
    <form id="requestForm">
        <select id="event_id" required>
            <option value="">— Выберите помещение —</option>
            {% for e in events %}
                <option value="{{ e.id }}">{{ e.name }} ({{ e.place }})</option>
            {% endfor %}
        </select>
        <button type="submit">Отправить</button>
        <a href="/dashboard" class="btn btn-secondary">Отмена</a>
    </form>
    <div id="message"></div>
</div>
<script src="{{ url_for('static', filename='js/request.js') }}"></script>
</body>
</html>
```

### 6.5. `frontend/templates/admin.html`

```html
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <title>Панель администратора</title>
    <link rel="stylesheet" href="{{ url_for('static', filename='css/style.css') }}">
</head>
<body>
<div class="container container-wide">
    <header>
        <h1>Панель администратора</h1>
        <button onclick="logout()" class="btn btn-secondary">Выйти</button>
    </header>

    <h2>Заявки пользователей</h2>
    <table id="adminTable">
        <thead>
        <tr>
            <th>№</th>
            <th>Пользователь</th>
            <th>Мероприятие</th>
            <th>Дата</th>
            <th>Статус</th>
            <th>Действия</th>
        </tr>
        </thead>
        <tbody id="adminBody">
        <tr><td colspan="6">Загрузка...</td></tr>
        </tbody>
    </table>

    <h2>Отзывы</h2>
    <table id="reviewsTable">
        <thead>
        <tr>
            <th>№</th>
            <th>Пользователь</th>
            <th>Мероприятие</th>
            <th>Оценка</th>
            <th>Текст</th>
            <th>Дата</th>
            <th>Действия</th>
        </tr>
        </thead>
        <tbody id="reviewsBody">
        <tr><td colspan="7">Загрузка...</td></tr>
        </tbody>
    </table>
</div>

<script>
const STATUS_NAMES = {
    1: 'Новая',
    2: 'Мероприятие назначено',
    3: 'Завершено'
};

async function loadRequests() {
    const res = await fetch('/api/admin/requests');
    const data = await res.json();
    const tbody = document.getElementById('adminBody');
    tbody.innerHTML = '';

    if (!data.success) return;

    data.requests.forEach(r => {
        let statusOptions = '';
        for (const [id, name] of Object.entries(STATUS_NAMES)) {
            const selected = r.status_id == id ? 'selected' : '';
            statusOptions += `<option value="${id}" ${selected}>${name}</option>`;
        }
        const disabled = r.status_id === 3 ? 'disabled' : '';

        tbody.insertAdjacentHTML('beforeend', `
            <tr>
                <td>${r.id}</td>
                <td>${r.login}</td>
                <td>${r.event_name} (${r.place})</td>
                <td>${r.date}</td>
                <td><span class="status status-${r.status_id}">${r.status_name}</span></td>
                <td>
                    <select onchange="changeStatus(${r.id}, this.value)" ${disabled}>
                        ${statusOptions}
                    </select>
                    <button onclick="deleteRequest(${r.id})"
                            class="btn btn-danger btn-sm">Удалить</button>
                </td>
            </tr>
        `);
    });
}

async function loadReviews() {
    const res = await fetch('/api/reviews');
    const data = await res.json();
    const tbody = document.getElementById('reviewsBody');
    tbody.innerHTML = '';

    if (!data.success || data.reviews.length === 0) {
        tbody.innerHTML = '<tr><td colspan="7">Отзывов пока нет</td></tr>';
        return;
    }

    data.reviews.forEach(r => {
        const stars = '★'.repeat(r.rating) + '☆'.repeat(5 - r.rating);
        tbody.insertAdjacentHTML('beforeend', `
            <tr>
                <td>${r.id}</td>
                <td>${r.login}</td>
                <td>${r.event_name}</td>
                <td>${stars} (${r.rating}/5)</td>
                <td>${r.text}</td>
                <td>${r.created_at}</td>
                <td>
                    <button onclick="deleteReview(${r.id})"
                            class="btn btn-danger btn-sm">Удалить</button>
                </td>
            </tr>
        `);
    });
}

async function changeStatus(requestId, statusId) {
    const res = await fetch(`/api/admin/requests/${requestId}`, {
        method: 'PUT',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ status_id: parseInt(statusId) })
    });
    const data = await res.json();
    if (!data.success) alert(data.message);
    loadRequests();
}

async function deleteRequest(id) {
    if (!confirm('Удалить заявку?')) return;
    const res = await fetch(`/api/admin/requests/${id}`, { method: 'DELETE' });
    const data = await res.json();
    if (data.success) {
        loadRequests();
        loadReviews();
    }
}

async function deleteReview(id) {
    if (!confirm('Удалить отзыв?')) return;
    const res = await fetch(`/api/reviews/${id}`, { method: 'DELETE' });
    const data = await res.json();
    if (data.success) loadReviews();
}

async function logout() {
    await fetch('/api/auth/logout', { method: 'POST' });
    window.location.href = '/api/auth/login';
}

loadRequests();
loadReviews();
</script>
</body>
</html>
```

### 6.6. `frontend/templates/review.html`

```html
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <title>Отзыв о мероприятии</title>
    <link rel="stylesheet" href="{{ url_for('static', filename='css/style.css') }}">
</head>
<body>
<div class="container">
    <h1>Отзыв о мероприятии</h1>

    <div class="info-block">
        <p><strong>Мероприятие:</strong> {{ req.event_name }}</p>
        <p><strong>Дата:</strong> {{ req.date }}</p>
        <p><strong>Место:</strong> {{ req.place }}</p>
    </div>

    {% if review %}
        <div class="alert alert-info">
            <p>Вы уже оставили отзыв:</p>
            <p><strong>Оценка:</strong>
                {{ '★' * review.rating }}{{ '☆' * (5 - review.rating) }}
            </p>
            <p><strong>Текст:</strong> {{ review.text }}</p>
            <p><strong>Дата:</strong> {{ review.created_at }}</p>
        </div>
        <a href="/dashboard" class="btn">← Вернуться к заявкам</a>
    {% else %}
        <form id="reviewForm">
            <input type="hidden" id="request_id" value="{{ request_id }}">

            <label>Оценка:</label>
            <div class="rating">
                <label><input type="radio" name="rating" value="5" required> ★★★★★ Отлично</label>
                <label><input type="radio" name="rating" value="4"> ★★★★ Хорошо</label>
                <label><input type="radio" name="rating" value="3"> ★★★ Удовлетворительно</label>
                <label><input type="radio" name="rating" value="2"> ★★ Плохо</label>
                <label><input type="radio" name="rating" value="1"> ★ Очень плохо</label>
            </div>

            <label for="text">Текст отзыва:</label>
            <textarea id="text" rows="5"
                      placeholder="Расскажите о мероприятии..."
                      minlength="5" maxlength="2000" required></textarea>

            <button type="submit" class="btn btn-primary">Отправить отзыв</button>
            <a href="/dashboard" class="btn btn-secondary">Отмена</a>
        </form>

        <div id="message"></div>
    {% endif %}
</div>

<script>
document.getElementById('reviewForm')?.addEventListener('submit', async (e) => {
    e.preventDefault();

    const rating = document.querySelector('input[name="rating"]:checked')?.value;
    const text = document.getElementById('text').value.trim();
    const request_id = document.getElementById('request_id').value;
    const msg = document.getElementById('message');

    if (!rating) {
        msg.className = 'error';
        msg.textContent = 'Поставьте оценку';
        return;
    }
    if (text.length < 5) {
        msg.className = 'error';
        msg.textContent = 'Текст отзыва минимум 5 символов';
        return;
    }

    const res = await fetch('/api/review', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
            request_id: parseInt(request_id),
            text: text,
            rating: parseInt(rating)
        })
    });
    const data = await res.json();

    if (data.success) {
        msg.className = 'success';
        msg.textContent = data.message;
        setTimeout(() => window.location.href = '/dashboard', 1200);
    } else {
        msg.className = 'error';
        msg.textContent = data.message;
    }
});
</script>
</body>
</html>
```

### 6.7. `frontend/static/js/register.js`

```javascript
document.getElementById('registerForm').addEventListener('submit', async (e) => {
    e.preventDefault();

    const login = document.getElementById('login').value.trim();
    const password = document.getElementById('password').value;
    const email = document.getElementById('email').value.trim();
    const phone = document.getElementById('phone').value.trim();

    const msg = document.getElementById('message');
    msg.textContent = '';
    msg.className = '';

    // Клиентская валидация
    if (!/^[A-Za-z0-9]{6,50}$/.test(login)) {
        return showError('Логин: от 6 до 50 символов, только латиница и цифры');
    }
    if (password.length < 8) {
        return showError('Пароль должен быть не короче 8 символов');
    }
    if (!/^8\(\d{3}\)\d{3}-\d{2}-\d{2}$/.test(phone)) {
        return showError('Телефон в формате 8(XXX)XXX-XX-XX');
    }
    if (!/^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$/.test(email)) {
        return showError('Некорректный email');
    }

    try {
        const res = await fetch('/api/auth/register', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ login, password, email, phone })
        });
        const data = await res.json();

        if (data.success) {
            msg.className = 'success';
            msg.textContent = 'Пользователь успешно создан!';
            setTimeout(() => window.location.href = '/api/auth/login', 1000);
        } else {
            showError(data.message);
        }
    } catch (err) {
        showError('Ошибка сервера: ' + err.message);
    }
});

function showError(text) {
    const msg = document.getElementById('message');
    msg.className = 'error';
    msg.textContent = text;
}
```

### 6.8. `frontend/static/js/login.js`

```javascript
document.getElementById('loginForm').addEventListener('submit', async (e) => {
    e.preventDefault();

    const payload = {
        login: document.getElementById('login').value.trim(),
        password: document.getElementById('password').value
    };
    const msg = document.getElementById('message');

    try {
        const res = await fetch('/api/auth/login', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify(payload)
        });
        const data = await res.json();
        if (data.success) {
            window.location.href = data.redirect;
        } else {
            msg.className = 'error';
            msg.textContent = data.message;
        }
    } catch (err) {
        msg.className = 'error';
        msg.textContent = 'Ошибка сервера';
    }
});
```

### 6.9. `frontend/static/js/request.js`

```javascript
document.getElementById('requestForm').addEventListener('submit', async (e) => {
    e.preventDefault();
    const eventId = document.getElementById('event_id').value;
    const msg = document.getElementById('message');

    if (!eventId) {
        msg.className = 'error';
        msg.textContent = 'Выберите помещение';
        return;
    }

    const res = await fetch('/create_request', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ event_id: eventId })
    });
    const data = await res.json();

    if (data.success) {
        msg.className = 'success';
        msg.textContent = data.message;
        setTimeout(() => window.location.href = '/dashboard', 1200);
    } else {
        msg.className = 'error';
        msg.textContent = data.message || 'Ошибка';
    }
});
```

### 6.10. `frontend/static/css/style.css`

```css
/* =====================================================================
   ОБЩИЕ СТИЛИ
   ===================================================================== */
body {
    font-family: Arial, sans-serif;
    background: #f4f6f8;
    margin: 0;
    padding: 0;
}

.container {
    max-width: 800px;
    margin: 40px auto;
    background: #fff;
    padding: 30px;
    border-radius: 10px;
    box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
}

.container-wide {
    max-width: 1200px;
}

h1 {
    color: #2c3e50;
}

h2 {
    color: #34495e;
    margin-top: 30px;
}

/* =====================================================================
   ФОРМЫ
   ===================================================================== */
input, select, textarea, button {
    display: block;
    width: 100%;
    padding: 10px;
    margin: 10px 0;
    font-size: 16px;
    box-sizing: border-box;
}

textarea {
    font-family: inherit;
    border: 1px solid #ddd;
    border-radius: 5px;
    resize: vertical;
}

button {
    background: #2980b9;
    color: #fff;
    border: none;
    cursor: pointer;
    border-radius: 5px;
}

button:hover {
    background: #1c5980;
}

/* =====================================================================
   ЗАГОЛОВОК
   ===================================================================== */
header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 20px;
    padding-bottom: 15px;
    border-bottom: 2px solid #ecf0f1;
}

header .actions {
    display: flex;
    gap: 10px;
}

/* =====================================================================
   КНОПКИ
   ===================================================================== */
.btn {
    display: inline-block;
    padding: 8px 16px;
    border-radius: 5px;
    text-decoration: none;
    border: none;
    cursor: pointer;
    font-size: 14px;
    text-align: center;
}

.btn-primary   { background: #2980b9; color: #fff; }
.btn-secondary { background: #95a5a6; color: #fff; }
.btn-danger    { background: #e74c3c; color: #fff; }
.btn-success   { background: #27ae60; color: #fff; }
.btn-sm        { padding: 4px 10px; font-size: 12px; width: auto; display: inline-block; }

.btn:hover { opacity: 0.9; }

/* =====================================================================
   ТАБЛИЦЫ
   ===================================================================== */
table {
    width: 100%;
    border-collapse: collapse;
    margin-top: 20px;
}

th, td {
    border: 1px solid #ddd;
    padding: 8px;
    text-align: left;
}

th {
    background: #ecf0f1;
}

/* =====================================================================
   СТАТУСЫ
   ===================================================================== */
.status {
    display: inline-block;
    padding: 4px 10px;
    border-radius: 12px;
    font-size: 12px;
    font-weight: bold;
}

.status-1 { background: #f39c12; color: #fff; }
.status-2 { background: #3498db; color: #fff; }
.status-3 { background: #27ae60; color: #fff; }

/* =====================================================================
   СООБЩЕНИЯ
   ===================================================================== */
.error {
    color: #c0392b;
    margin-top: 10px;
}

.success {
    color: #27ae60;
    margin-top: 10px;
}

/* =====================================================================
   ФОРМА ОТЗЫВА
   ===================================================================== */
.info-block {
    background: #ecf0f1;
    padding: 15px;
    border-radius: 5px;
    margin-bottom: 20px;
}

.rating {
    display: flex;
    flex-direction: column;
    gap: 8px;
    margin: 15px 0;
}

.rating label {
    cursor: pointer;
    padding: 5px;
    border-radius: 3px;
    display: flex;
    align-items: center;
    gap: 8px;
}

.rating label:hover {
    background: #f4f6f8;
}

.rating input[type="radio"] {
    width: auto;
    margin: 0;
}

.alert {
    padding: 15px;
    border-radius: 5px;
    margin: 15px 0;
}

.alert-info {
    background: #d1ecf1;
    border: 1px solid #bee5eb;
    color: #0c5460;
}
```

---

## 7. Порядок запуска проекта

### Шаг 1. Установка PostgreSQL в ROSA Linux

```bash
# Установка
sudo dnf install postgresql-server postgresql-contrib

# Инициализация БД
sudo su - postgres
initdb -D /var/lib/pgsql/data
exit

# Запуск службы
sudo systemctl start postgresql12.service
sudo systemctl enable postgresql12.service

# Проверка
sudo systemctl status postgresql12.service
```

### Шаг 2. Создание пользователя и базы данных

```bash
sudo -u postgres psql <<EOF
CREATE USER conference_user WITH PASSWORD 'Conf2027!';
CREATE DATABASE conference_db OWNER conference_user;
GRANT ALL PRIVILEGES ON DATABASE conference_db TO conference_user;
EOF
```

### Шаг 3. Применение схемы

```bash
PGPASSWORD='Conf2027!' psql -U conference_user -d conference_db -h 127.0.0.1 -f schema_postgres.sql
```

### Шаг 4. Виртуальное окружение Python

```bash
cd ~/conference_portal
python3 -m venv venv
source venv/bin/activate
pip install --upgrade pip
pip install -r requirements.txt
```

### Шаг 5. Наполнение БД мероприятиями

```bash
python seed_events.py
```

### Шаг 6. Запуск приложения

```bash
python -m backend.app
```

**Ожидаемый вывод:**
```
✓ Администратор Conf2027 создан (id=1)
 * Running on http://127.0.0.1:5000
```

### Шаг 7. Проверка через браузер

1. Откройте http://127.0.0.1:5000
2. Войдите как администратор: `Conf2027 / Demo77`
3. Создайте пользователя через `/api/auth/register`
4. Создайте заявку, смените статус, оставьте отзыв

### Шаг 8. Тест защиты от SQL-инъекций

```bash
python test_pg_injection.py
```

---

## 8. Итоговый чек-лист

| № | Действие | ☐ |
|---|----------|---|
| 1 | PostgreSQL установлен и запущен | ☐ |
| 2 | Созданы `conference_user` и `conference_db` | ☐ |
| 3 | Применён `schema_postgres.sql` | ☐ |
| 4 | Виртуальное окружение активировано | ☐ |
| 5 | Установлены зависимости из `requirements.txt` | ☐ |
| 6 | Создан `.env` с `DATABASE_URL` | ☐ |
| 7 | Запущен `seed_events.py` | ☐ |
| 8 | Приложение стартует (`python -m backend.app`) | ☐ |
| 9 | Администратор `Conf2027` создан | ☐ |
| 10 | Регистрация нового пользователя работает | ☐ |
| 11 | Создание заявки работает | ☐ |
| 12 | Смена статусов работает | ☐ |
| 13 | Отзывы работают | ☐ |
| 14 | Тест SQL-инъекций пройден | ☐ |
| 15 | Коммиты в Git сделаны (≥ 5) | ☐ |

---

## 9. Возможные проблемы и решения
| Проблема | Решение |
|----------|---------|
| `psycopg2.OperationalError: could not connect` | `sudo systemctl start postgresql12.service` |
| `relation "user" does not exist` | Использовать `"user"` в кавычках |
| `syntax error at or near "?"` | Заменить `?` на `%s` |
| `password authentication failed` | Проверить `DATABASE_URL` в `.env` |
| `column "id" does not exist` | Добавить `RETURNING id` в INSERT |
| `duplicate key value violates unique constraint` | Использовать `ON CONFLICT DO NOTHING` |
| `ModuleNotFoundError: psycopg2` | `pip install psycopg2-binary` |
| `ModuleNotFoundError: dotenv` | `pip install python-dotenv` |
| `database "conference_db" does not exist` | `sudo -u postgres createdb -O conference_user conference_db` |

---
## Заключение

Проект «Конференции.РФ» **полностью переписан под PostgreSQL**:

- ✅ Использует `psycopg2-binary` вместо `sqlite3`
- ✅ Плейсхолдеры `%s` вместо `?`
- ✅ `SERIAL PRIMARY KEY` и `RETURNING id`
- ✅ `RealDictCursor` для удобной работы со словарями
- ✅ `ON DELETE CASCADE` для каскадного удаления
- ✅ Индексы для ускорения запросов
- ✅ Безопасное подключение через `.env`
- ✅ Полный функционал: CRUD, смена статусов, отзывы

Доработайте проект - напиши собственные фронтенд шаблоны для данного проекта.

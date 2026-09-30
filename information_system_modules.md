# Практическая работа № 3
## Разработка модулей информационной системы в соответствии с техническим заданием

**Дисциплина:** Информационные системы и программирование  
**Специальность:** 09.02.07 «Информационные системы и программирование»  
**Тема:** Разработка модулей ИС «Конференции.РФ» на Python 3.8 + SQLite  
**Время выполнения:** 4 академических часа (180 минут)  
**ОС:** РОСА Линукс

---

## Цель работы

Разработать основные модули информационной системы для портала «Конференции.РФ»:

1. Модуль регистрации с полной валидацией полей.
2. Модуль авторизации.
3. Модуль создания заявки на бронирование помещения.
4. Модуль просмотра заявок пользователя.
5. Панель администратора.
6. Разделить логику на фронтенд, бекенд и слой работы с БД.
7. Применить принципы ООП.
8. Сохранить результаты в Git-репозиторий.

---

## Оснащение рабочего места

| Компонент | Версия / Примечание |
|-----------|---------------------|
| ОС | РОСА Линукс |
| Python | 3.8+ |
| SQLite | 3.x |
| Веб-фреймворк | Flask 2.x (или FastAPI) |
| Фронтенд | HTML5, CSS3, JavaScript |
| Редактор | VS Code / PyCharm |
| Git | 2.x |

---

## Краткие теоретические сведения

### Архитектура приложения

Проект строится по **трёхслойной архитектуре**:

```
┌───────────────────────────────────────────┐
│         ФРОНТЕНД (Frontend)               │
│  HTML-шаблоны, CSS, JavaScript            │
│  Отображение данных, формы, валидация     │
└──────────────────┬────────────────────────┘
                   │ HTTP-запросы (JSON)
┌──────────────────▼────────────────────────┐
│         БЕКЕНД (Backend)                  │
│  Flask, маршруты (routes), бизнес-логика  │
│  Классы-модели, валидация на сервере      │
└──────────────────┬────────────────────────┘
                   │ SQL-запросы
┌──────────────────▼────────────────────────┐
│         БАЗА ДАННЫХ (SQLite)              │
│  Таблицы: user, event, request, status,   │
│  review                                   │
└───────────────────────────────────────────┘
```

### Принципы ООП, применяемые в работе

| Принцип | Применение |
|---------|------------|
| **Инкапсуляция** | Классы-модели скрывают работу с БД |
| **Наследование** | Базовый класс `BaseModel` → дочерние `User`, `Request` и т. д. |
| **Полиморфизм** | Методы `save()`, `validate()` переопределяются в дочерних классах |
| **Абстракция** | Пользователь работает с методами `create()`, `get_all()`, не зная деталей SQL |

---

## Структура проекта

Создайте следующую структуру папок и файлов:

```
conference_portal/
├── backend/
│   ├── __init__.py
│   ├── app.py                  # Точка входа, Flask-приложение
│   ├── database.py             # Класс Database (Singleton)
│   ├── models/
│   │   ├── __init__.py
│   │   ├── base_model.py       # Базовый класс BaseModel
│   │   ├── user.py             # Класс User
│   │   ├── event.py            # Класс Event
│   │   ├── request.py          # Класс Request
│   │   ├── status.py           # Класс Status
│   │   └── review.py           # Класс Review
│   ├── validators/
│   │   ├── __init__.py
│   │   └── validators.py       # Функции валидации
│   └── routes/
│       ├── __init__.py
│       ├── auth_routes.py      # Регистрация / авторизация
│       ├── request_routes.py   # Создание / просмотр заявок
│       └── admin_routes.py     # Панель администратора
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
│       └── admin.html
├── database/
│   └── conference.db           # Файл базы данных SQLite
├── schema.sql                  # SQL-скрипт создания БД
├── requirements.txt
├── .gitignore
└── README.md
```

---

## Шаг 1. Подготовка окружения 

### 1.1. Установка Python 3.8 и pip

В РОСА Линукс выполните в терминале:

```bash
sudo dnf install python3 python3-pip python3-venv
python3 --version   # Должно быть 3.8 или выше
```

### 1.2. Создание виртуального окружения

```bash
mkdir conference_portal
cd conference_portal
python3 -m venv venv
source venv/bin/activate
```

### 1.3. Установка зависимостей

Создайте файл `requirements.txt`:

```
Flask==2.3.3
Werkzeug==2.3.7
```

Установите:

```bash
pip install -r requirements.txt
```

---

## Шаг 2. Создание базы данных 

### 2.1. Файл `schema.sql`

```sql
PRAGMA foreign_keys = ON;

CREATE TABLE IF NOT EXISTS user (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    login VARCHAR(50) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    phone VARCHAR(18) NOT NULL,
    role VARCHAR(20) NOT NULL DEFAULT 'user'
);

CREATE TABLE IF NOT EXISTS event (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name VARCHAR(200) NOT NULL,
    date DATE NOT NULL,
    place VARCHAR(200) NOT NULL,
    description TEXT
);

CREATE TABLE IF NOT EXISTS status (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name VARCHAR(50) UNIQUE NOT NULL
);

INSERT OR IGNORE INTO status (name) VALUES
    ('Новая'),
    ('Мероприятие назначено'),
    ('Завершено');

CREATE TABLE IF NOT EXISTS request (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id INTEGER NOT NULL,
    event_id INTEGER NOT NULL,
    status_id INTEGER NOT NULL DEFAULT 1,
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES user(id),
    FOREIGN KEY (event_id) REFERENCES event(id),
    FOREIGN KEY (status_id) REFERENCES status(id)
);

CREATE TABLE IF NOT EXISTS review (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    request_id INTEGER UNIQUE NOT NULL,
    text TEXT NOT NULL,
    rating INTEGER CHECK (rating BETWEEN 1 AND 5),
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (request_id) REFERENCES request(id)
);
```

### 2.2. Применение скрипта

```bash
mkdir database
sqlite3 database/conference.db < schema.sql
```

Проверьте:

```bash
sqlite3 database/conference.db ".tables"
```

Должны отобразиться таблицы: `event  request  review  status  user`.

---

## Шаг 3. Реализация слоя работы с БД 

### 3.1. Файл `backend/database.py`

```python
import sqlite3
import os

class Database:
    """Singleton-класс для работы с SQLite."""
    _instance = None

    def __new__(cls, db_path=None):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
            cls._instance._initialized = False
        return cls._instance

    def __init__(self, db_path=None):
        if self._initialized:
            return
        if db_path is None:
            db_path = os.path.join(
                os.path.dirname(__file__), '..', 'database', 'conference.db'
            )
        self.db_path = os.path.abspath(db_path)
        self._initialized = True

    def get_connection(self):
        conn = sqlite3.connect(self.db_path)
        conn.row_factory = sqlite3.Row
        conn.execute("PRAGMA foreign_keys = ON")
        return conn

    def execute(self, query, params=()):
        """Выполнить INSERT / UPDATE / DELETE."""
        with self.get_connection() as conn:
            cursor = conn.execute(query, params)
            conn.commit()
            return cursor.lastrowid

    def fetch_one(self, query, params=()):
        with self.get_connection() as conn:
            cursor = conn.execute(query, params)
            row = cursor.fetchone()
            return dict(row) if row else None

    def fetch_all(self, query, params=()):
        with self.get_connection() as conn:
            cursor = conn.execute(query, params)
            return [dict(row) for row in cursor.fetchall()]
```

**Что демонстрирует этот класс:**
- **Инкапсуляция** — работа с соединением скрыта.
- **Singleton** — единая точка подключения к БД.
- Удобные методы `execute`, `fetch_one`, `fetch_all`.

---

## Шаг 4. Реализация валидаторов

### 4.1. Файл `backend/validators/validators.py`

```python
import re

class Validator:
    """Класс для валидации пользовательских данных."""

    @staticmethod
    def validate_login(login: str) -> tuple:
        """Логин: латиница + цифры, минимум 6 символов."""
        if not login or len(login) < 6:
            return False, "Логин должен содержать минимум 6 символов"
        if not re.match(r'^[A-Za-z0-9]+$', login):
            return False, "Логин может содержать только латиницу и цифры"
        return True, ""

    @staticmethod
    def validate_password(password: str) -> tuple:
        """Пароль: минимум 8 символов."""
        if not password or len(password) < 8:
            return False, "Пароль должен содержать минимум 8 символов"
        return True, ""

    @staticmethod
    def validate_email(email: str) -> tuple:
        """Формат email."""
        pattern = r'^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$'
        if not email or not re.match(pattern, email):
            return False, "Некорректный email"
        return True, ""

    @staticmethod
    def validate_phone(phone: str) -> tuple:
        """Телефон в формате 8(XXX)XXX-XX-XX."""
        pattern = r'^8\(\d{3}\)\d{3}-\d{2}-\d{2}$'
        if not phone or not re.match(pattern, phone):
            return False, "Телефон в формате 8(XXX)XXX-XX-XX"
        return True, ""

    @staticmethod
    def validate_fio(fio: str) -> tuple:
        """ФИО: кириллица и пробелы."""
        pattern = r'^[А-Яа-яЁё\s]+$'
        if not fio or not re.match(pattern, fio):
            return False, "ФИО должно содержать только кириллицу и пробелы"
        return True, ""
```

**Что демонстрирует этот класс:**
- **Инкапсуляция логики** валидации в одном месте.
- Использование **статических методов** (можно вызывать без создания экземпляра).
- Возврат кортежа `(bool, сообщение)` — удобно для отображения ошибок.

---

## Шаг 5. Реализация моделей (ООП) 

### 5.1. Базовый класс `backend/models/base_model.py`

```python
from backend.database import Database

class BaseModel:
    """Базовый класс для всех моделей. Демонстрирует наследование."""
    table_name = None

    def __init__(self, **kwargs):
        self.db = Database()
        for key, value in kwargs.items():
            setattr(self, key, value)

    def save(self):
        """Метод должен быть переопределён в дочерних классах."""
        raise NotImplementedError("Метод save() должен быть реализован")

    @classmethod
    def get_all(cls):
        db = Database()
        return db.fetch_all(f"SELECT * FROM {cls.table_name}")

    @classmethod
    def get_by_id(cls, record_id: int):
        db = Database()
        return db.fetch_one(
            f"SELECT * FROM {cls.table_name} WHERE id = ?",
            (record_id,)
        )
```

**Что демонстрирует этот класс:**
- **Наследование** — дочерние классы получают общие методы.
- **Полиморфизм** — метод `save()` переопределяется.
- **Абстракция** — работа с БД скрыта.

### 5.2. Модель `User` — `backend/models/user.py`

```python
from backend.models.base_model import BaseModel
from backend.validators.validators import Validator
from werkzeug.security import generate_password_hash, check_password_hash


class User(BaseModel):
    table_name = "user"

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
        """Проверка уникальности логина."""
        row = self.db.fetch_one(
            "SELECT id FROM user WHERE login = ?", (self.login,)
        )
        return row is not None

    def email_exists(self) -> bool:
        row = self.db.fetch_one(
            "SELECT id FROM user WHERE email = ?", (self.email,)
        )
        return row is not None

    def save(self) -> tuple:
        """Сохранение нового пользователя с валидацией."""
        ok, msg = self.validate()
        if not ok:
            return False, msg
        if self.login_exists():
            return False, "Пользователь с таким логином уже существует"
        if self.email_exists():
            return False, "Пользователь с таким email уже существует"

        password_hash = generate_password_hash(self.password)
        user_id = self.db.execute(
            """INSERT INTO user (login, password_hash, email, phone, role)
               VALUES (?, ?, ?, ?, ?)""",
            (self.login, password_hash, self.email, self.phone, self.role)
        )
        self.id = user_id
        return True, user_id

    @staticmethod
    def authenticate(login: str, password: str) -> dict:
        """Аутентификация по логину и паролю."""
        db = Database()
        user = db.fetch_one("SELECT * FROM user WHERE login = ?", (login,))
        if user and check_password_hash(user['password_hash'], password):
            return user
        return None
```

### 5.3. Модель `Status` — `backend/models/status.py`

```python
from backend.models.base_model import BaseModel


class Status(BaseModel):
    table_name = "status"

    def __init__(self, name=None, **kwargs):
        super().__init__(**kwargs)
        self.name = name
```

### 5.4. Модель `Event` — `backend/models/event.py`

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
        return self.db.execute(
            """INSERT INTO event (name, date, place, description)
               VALUES (?, ?, ?, ?)""",
            (self.name, self.date, self.place, self.description)
        )
```

### 5.5. Модель `Request` — `backend/models/request.py`

```python
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
               VALUES (?, ?, ?)""",
            (self.user_id, self.event_id, self.status_id)
        )
        self.id = request_id
        return request_id

    @classmethod
    def get_by_user(cls, user_id: int):
        """Все заявки конкретного пользователя с названиями статусов."""
        db = Database()
        return db.fetch_all(
            """SELECT r.id, e.name AS event_name, e.date,
                      e.place, s.name AS status_name,
                      r.created_at
               FROM request r
               JOIN event e ON r.event_id = e.id
               JOIN status s ON r.status_id = s.id
               WHERE r.user_id = ?
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
                      r.created_at
               FROM request r
               JOIN user u ON r.user_id = u.id
               JOIN event e ON r.event_id = e.id
               JOIN status s ON r.status_id = s.id
               ORDER BY r.created_at DESC"""
        )

    @classmethod
    def update_status(cls, request_id: int, status_id: int) -> bool:
        db = Database()
        db.execute(
            "UPDATE request SET status_id = ? WHERE id = ?",
            (status_id, request_id)
        )
        return True
```

### 5.6. Модель `Review` — `backend/models/review.py`

```python
from backend.models.base_model import BaseModel


class Review(BaseModel):
    table_name = "review"

    def __init__(self, request_id=None, text=None,
                 rating=None, **kwargs):
        super().__init__(**kwargs)
        self.request_id = request_id
        self.text = text
        self.rating = rating

    def save(self) -> int:
        return self.db.execute(
            """INSERT INTO review (request_id, text, rating)
               VALUES (?, ?, ?)""",
            (self.request_id, self.text, self.rating)
        )
```

---

## Шаг 6. Реализация маршрутов (Routes) 

### 6.1. Регистрация и авторизация — `backend/routes/auth_routes.py`

```python
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

    session['user_id'] = user['id']
    session['role'] = user['role']
    session['login'] = user['login']
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

### 6.2. Заявки — `backend/routes/request_routes.py`

```python
from flask import Blueprint, request, jsonify, render_template, session
from backend.models.request import Request
from backend.models.event import Event

request_bp = Blueprint('requests', __name__)


def _check_auth():
    if 'user_id' not in session:
        return False
    return True


@request_bp.route('/dashboard')
def dashboard():
    if not _check_auth():
        return render_template('login.html')
    return render_template('dashboard.html')


@request_bp.route('/api/requests', methods=['GET'])
def get_my_requests():
    if not _check_auth():
        return jsonify({'success': False, 'message': 'Не авторизован'}), 401
    data = Request.get_by_user(session['user_id'])
    return jsonify({'success': True, 'requests': data}), 200


@request_bp.route('/create_request', methods=['GET', 'POST'])
def create_request():
    if not _check_auth():
        return render_template('login.html')

    if request.method == 'GET':
        events = Event.get_all()
        return render_template('create_request.html', events=events)

    data = request.get_json()
    new_request = Request(
        user_id=session['user_id'],
        event_id=int(data.get('event_id')),
        status_id=1  # «Новая»
    )
    request_id = new_request.save()
    return jsonify({'success': True, 'request_id': request_id}), 201
```

### 6.3. Панель администратора — `backend/routes/admin_routes.py`

```python
from flask import Blueprint, request, jsonify, render_template, session
from backend.models.request import Request

admin_bp = Blueprint('admin', __name__)


def _check_admin():
    return session.get('role') == 'admin'


@admin_bp.route('/admin')
def admin_panel():
    if not _check_admin():
        return render_template('login.html')
    return render_template('admin.html')


@admin_bp.route('/api/admin/requests', methods=['GET'])
def get_all_requests():
    if not _check_admin():
        return jsonify({'success': False, 'message': 'Доступ запрещён'}), 403
    data = Request.get_all_with_details()
    return jsonify({'success': True, 'requests': data}), 200


@admin_bp.route('/api/admin/requests/<int:request_id>', methods=['PUT'])
def update_request_status(request_id):
    if not _check_admin():
        return jsonify({'success': False, 'message': 'Доступ запрещён'}), 403
    data = request.get_json()
    status_id = int(data.get('status_id'))
    if status_id not in (1, 2, 3):
        return jsonify({'success': False, 'message': 'Некорректный статус'}), 400
    Request.update_status(request_id, status_id)
    return jsonify({'success': True}), 200
```

### 6.4. Точка входа — `backend/app.py`

```python
from flask import Flask
from backend.routes.auth_routes import auth_bp
from backend.routes.request_routes import request_bp
from backend.routes.admin_routes import admin_bp
from backend.models.user import User

app = Flask(
    __name__,
    template_folder='../frontend/templates',
    static_folder='../frontend/static'
)
app.secret_key = 'conf2027_secret_key_demo'

# Регистрация blueprints
app.register_blueprint(auth_bp, url_prefix='/api/auth')
app.register_blueprint(request_bp)
app.register_blueprint(admin_bp)


@app.route('/')
def index():
    return render_template('login.html')


def init_admin():
    """Создание администратора по умолчанию (Conf2027 / Demo77)."""
    admin = User(
        login='Conf2027',
        password='Demo77',
        email='admin@conf2027.ru',
        phone='8(999)000-00-00',
        role='admin'
    )
    if not admin.login_exists():
        admin.save()
        print("✓ Администратор Conf2027 создан")


if __name__ == '__main__':
    init_admin()
    app.run(debug=True, host='127.0.0.1', port=5000)
```

---

## Шаг 7. Реализация фронтенда

### 7.1. Шаблон `frontend/templates/register.html`

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
        <input type="text" id="fio" placeholder="ФИО (кириллица)" required>
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

### 7.2. JavaScript `frontend/static/js/register.js`

```javascript
document.getElementById('registerForm').addEventListener('submit', async (e) => {
    e.preventDefault();

    const payload = {
        login:    document.getElementById('login').value.trim(),
        password: document.getElementById('password').value,
        email:    document.getElementById('email').value.trim(),
        phone:    document.getElementById('phone').value.trim()
    };

    const msg = document.getElementById('message');
    msg.textContent = '';
    msg.className = '';

    // Клиентская валидация
    if (!/^[A-Za-z0-9]{6,}$/.test(payload.login)) {
        return showError('Логин: минимум 6 символов (латиница и цифры)');
    }
    if (payload.password.length < 8) {
        return showError('Пароль должен быть не короче 8 символов');
    }
    if (!/^8\(\d{3}\)\d{3}-\d{2}-\d{2}$/.test(payload.phone)) {
        return showError('Телефон в формате 8(XXX)XXX-XX-XX');
    }
    if (!/^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$/.test(payload.email)) {
        return showError('Некорректный email');
    }

    try {
        const res = await fetch('/api/auth/register', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify(payload)
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

### 7.3. Шаблон `frontend/templates/login.html`

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

### 7.4. JavaScript `frontend/static/js/login.js`

```javascript
document.getElementById('loginForm').addEventListener('submit', async (e) => {
    e.preventDefault();

    const payload = {
        login:    document.getElementById('login').value.trim(),
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

### 7.5. Шаблон `dashboard.html` (просмотр заявок)

```html
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <title>Мои заявки</title>
    <link rel="stylesheet" href="{{ url_for('static', filename='css/style.css') }}">
</head>
<body>
<div class="container">
    <h1>Мои заявки</h1>
    <a href="/create_request" class="btn">Создать заявку</a>
    <table id="requestsTable">
        <thead>
        <tr>
            <th>№</th><th>Мероприятие</th><th>Дата</th>
            <th>Место</th><th>Статус</th><th>Создана</th>
        </tr>
        </thead>
        <tbody></tbody>
    </table>
</div>
<script>
    fetch('/api/requests')
        .then(r => r.json())
        .then(data => {
            const tbody = document.querySelector('#requestsTable tbody');
            if (!data.success) return;
            data.requests.forEach(r => {
                tbody.insertAdjacentHTML('beforeend', `
                    <tr>
                        <td>${r.id}</td>
                        <td>${r.event_name}</td>
                        <td>${r.date}</td>
                        <td>${r.place}</td>
                        <td>${r.status_name}</td>
                        <td>${r.created_at}</td>
                    </tr>
                `);
            });
        });
</script>
</body>
</html>
```

### 7.6. Шаблон `create_request.html`

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
    </form>
    <div id="message"></div>
</div>
<script src="{{ url_for('static', filename='js/request.js') }}"></script>
</body>
</html>
```

### 7.7. JavaScript `request.js`

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
        msg.textContent = 'Заявка отправлена администратору!';
        setTimeout(() => window.location.href = '/dashboard', 1200);
    } else {
        msg.className = 'error';
        msg.textContent = data.message || 'Ошибка';
    }
});
```

### 7.8. Шаблон `admin.html` (панель администратора)

```html
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <title>Панель администратора</title>
    <link rel="stylesheet" href="{{ url_for('static', filename='css/style.css') }}">
</head>
<body>
<div class="container">
    <h1>Панель администратора</h1>
    <table id="adminTable">
        <thead>
        <tr>
            <th>№</th><th>Пользователь</th><th>Мероприятие</th>
            <th>Дата</th><th>Статус</th><th>Изменить</th>
        </tr>
        </thead>
        <tbody></tbody>
    </table>
</div>
<script>
    async function loadRequests() {
        const res = await fetch('/api/admin/requests');
        const data = await res.json();
        const tbody = document.querySelector('#adminTable tbody');
        tbody.innerHTML = '';
        if (!data.success) return;
        data.requests.forEach(r => {
            tbody.insertAdjacentHTML('beforeend', `
                <tr>
                    <td>${r.id}</td>
                    <td>${r.login}</td>
                    <td>${r.event_name}</td>
                    <td>${r.date}</td>
                    <td>${r.status_name}</td>
                    <td>
                        <select onchange="changeStatus(${r.id}, this.value)">
                            <option value="1" ${r.status_name === 'Новая' ? 'selected' : ''}>Новая</option>
                            <option value="2" ${r.status_name === 'Мероприятие назначено' ? 'selected' : ''}>Мероприятие назначено</option>
                            <option value="3" ${r.status_name === 'Завершено' ? 'selected' : ''}>Завершено</option>
                        </select>
                    </td>
                </tr>
            `);
        });
    }

    async function changeStatus(requestId, statusId) {
        await fetch(`/api/admin/requests/${requestId}`, {
            method: 'PUT',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ status_id: parseInt(statusId) })
        });
        loadRequests();
    }

    loadRequests();
</script>
</body>
</html>
```

### 7.9. CSS `frontend/static/css/style.css`

```css
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
    box-shadow: 0 2px 10px rgba(0,0,0,0.1);
}
h1 { color: #2c3e50; }
input, select, button {
    display: block;
    width: 100%;
    padding: 10px;
    margin: 10px 0;
    font-size: 16px;
    box-sizing: border-box;
}
button {
    background: #2980b9;
    color: #fff;
    border: none;
    cursor: pointer;
}
button:hover { background: #1c5980; }
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
th { background: #ecf0f1; }
.error   { color: #c0392b; margin-top: 10px; }
.success { color: #27ae60; margin-top: 10px; }
```

---

## Шаг 8. Запуск и проверка 

### 8.1. Запуск приложения

Из корневой папки проекта:

```bash
source venv/bin/activate
python -m backend.app
```

Должно появиться:

```
✓ Администратор Conf2027 создан
 * Running on http://127.0.0.1:5000
```

### 8.2. Проверка модулей

| Модуль | URL | Что проверить |
|--------|-----|--------------|
| Регистрация | http://127.0.0.1:5000/api/auth/register | Валидация полей, уникальность логина |
| Авторизация | http://127.0.0.1:5000/api/auth/login | Вход под `Conf2027 / Demo77` |
| Заявки пользователя | http://127.0.0.1:5000/dashboard | Отображение заявок |
| Создание заявки | http://127.0.0.1:5000/create_request | Выбор помещения, отправка |
| Панель администратора | http://127.0.0.1:5000/admin | Список всех заявок, смена статуса |

### 8.3. Тестовые сценарии

1. **Регистрация с некорректным логином** — должна появиться ошибка «Логин должен содержать минимум 6 символов».
2. **Регистрация с паролем «123»** — ошибка «Пароль должен содержать минимум 8 символов».
3. **Регистрация с телефоном «+79991234567»** — ошибка формата.
4. **Повторная регистрация с тем же логином** — ошибка «Пользователь с таким логином уже существует».
5. **Вход администратора** — перенаправление на `/admin`.

---

## Шаг 9. Сохранение в Git-репозиторий 

### 9.1. Создание `.gitignore`

```
venv/
__pycache__/
*.pyc
database/conference.db
.env
```

### 9.2. Коммиты 

```bash
git init
git add .
git commit -m "ПР №3: структура проекта, БД, модели ООП, валидация"

git add backend/routes/
git commit -m "ПР №3: маршруты регистрации, авторизации, заявок, админ-панель"

git add frontend/
git commit -m "ПР №3: фронтенд (HTML/CSS/JS) и валидация на клиенте"

git remote add origin <URL_вашего_репозитория>
git push -u origin main
```

**Рекомендуется сделать минимум 3 коммита** — поэтапно фиксировать создание слоёв.
**Документация по работе с git** - https://github.com/softboxdev/linux_deep_level/blob/main/practice/github.md 
---

## Требования к результату

1. Работающее приложение Flask, запускаемое командой `python -m backend.app`.
2. Разделение на три слоя: `frontend/`, `backend/`, `database/`.
3. Все 5 модулей работают:
   - регистрация с полной валидацией;
   - авторизация;
   - создание заявки;
   - просмотр заявок пользователя;
   - панель администратора со сменой статуса.
4. Использование ООП (наследование `BaseModel`, инкапсуляция в моделях).
5. Коммиты в Git-репозитории (минимум 3).


---

# Дополнительная часть. Инструкция по установке зависимостей для проекта на ROSA Linux

В этой инструкции описан полный процесс установки всех зависимостей, необходимых для запуска проекта «Конференции.РФ» на **Python 3.8** в **ROSA Linux**. Мы будем использовать **виртуальное окружение** (venv) — это изолированная среда Python, которая не затрагивает системные пакеты и предотвращает конфликты между проектами .

---

## Шаг 1. Проверка версии Python

В ROSA Linux (платформа 2021.1 и новее) Python 3.8 предустановлен . Проверьте его наличие:

```bash
python3 --version
```

**Ожидаемый результат:**
```
Python 3.8.x
```

Если Python не установлен или версия ниже 3.8, установите его:

```bash
sudo dnf install python3
```

---

## Шаг 2. Установка пакета python3-venv

Модуль `venv` входит в стандартную библиотеку Python, но в некоторых дистрибутивах он поставляется отдельным пакетом.

**Проверьте наличие venv:**

```bash
python3 -m venv --help
```

**Если команда не найдена**, установите пакет:

```bash
sudo dnf install python3-venv
```

---

## Шаг 3. Создание виртуального окружения

Перейдите в корневую папку вашего проекта:

```bash
cd ~/conference_portal
```

Создайте виртуальное окружение в папке `venv`:

```bash
python3 -m venv venv
```

**Что произойдёт:**
- В текущей папке появится директория `venv/`
- Внутри неё будет изолированная копия Python и pip 

> **Важно:** Директорию `venv/` необходимо добавить в `.gitignore`, чтобы не коммитить её в репозиторий .

---

## Шаг 4. Активация виртуального окружения

**Активируйте окружение:**

```bash
source venv/bin/activate
```

**Признак активации:** в приглашении терминала появится префикс `(venv)`:

```
(venv) user@rosa:~/conference_portal$
```

**Проверка, что вы внутри окружения:**

```bash
which python
```

**Ожидаемый результат:**
```
/home/user/conference_portal/venv/bin/python
```

Путь должен содержать `venv/` .

---

## Шаг 5. Обновление pip

Перед установкой пакетов обновите pip до последней версии:

```bash
pip install --upgrade pip
```

---

## Шаг 6. Установка Flask и Werkzeug

Установите необходимые пакеты:

```bash
pip install Flask==2.3.3 Werkzeug==2.3.7
```

**Пояснение:**
- **Flask** — веб-фреймворк для бекенда 
- **Werkzeug** — библиотека для хеширования паролей и работы с HTTP (устанавливается автоматически с Flask, но фиксируем версию)

**Проверка установки:**

```bash
pip list
```

**Ожидаемый вывод (среди прочих):**
```
Flask        2.3.3
Werkzeug     2.3.7
```

**Альтернативный способ проверки:**

```bash
python -m flask --version
```

**Ожидаемый результат:**
```
Flask 2.3.3
Python 3.8.x
```



---

## Шаг 7. Установка SQLite (если не установлен)

SQLite обычно предустановлен в ROSA Linux. Проверьте:

```bash
sqlite3 --version
```

**Если команда не найдена**, установите:

```bash
sudo dnf install sqlite3
```

> **Примечание:** Python 3.8 включает встроенный модуль `sqlite3`, поэтому дополнительно ничего устанавливать не нужно.

---

## Шаг 8. Установка Git (для коммитов)

Для работы с репозиторием установите Git:

```bash
sudo dnf install git
```

**Проверка:**

```bash
git --version
```

**Ожидаемый результат:**
```
git version 2.x.x
```



---

## Шаг 9. Создание файла requirements.txt

Для воспроизводимости окружения зафиксируйте зависимости в файле:

```bash
pip freeze > requirements.txt
```

**Содержимое `requirements.txt`:**
```
Flask==2.3.3
Werkzeug==2.3.7
```

Теперь любой разработчик может установить те же зависимости командой:

```bash
pip install -r requirements.txt
```

---

## Шаг 10. Проверка работы Flask

Создайте тестовый файл `test_flask.py`:

```python
from flask import Flask

app = Flask(__name__)

@app.route('/')
def hello():
    return 'Flask работает!'

if __name__ == '__main__':
    app.run(debug=True, port=5000)
```

**Запустите:**

```bash
python test_flask.py
```

**Ожидаемый вывод:**
```
 * Running on http://127.0.0.1:5000
```

**Откройте браузер** и перейдите по адресу `http://127.0.0.1:5000`. Вы должны увидеть сообщение «Flask работает!».

**Остановите сервер:** `Ctrl+C`

**Удалите тестовый файл:**

```bash
rm test_flask.py
```

---

## Шаг 11. Деактивация виртуального окружения

Когда вы закончили работу с проектом, деактивируйте окружение:

```bash
deactivate
```

Приглашение вернётся к обычному виду:

```
user@rosa:~/conference_portal$
```

При следующем запуске проекта снова активируйте окружение:

```bash
source venv/bin/activate
```

---

## Критерии оценивания

| № | Критерий | Баллы |
|---|----------|-------|
| 1 | Работоспособность всех 5 модулей | 3 |
| 2 | Наличие валидации (логин, пароль, email, телефон) | 2 |
| 3 | Использование ООП (наследование, инкапсуляция, полиморфизм) | 2 |
| 4 | Разделение фронтенда и бекенда | 2 |
| 5 | Коммиты в репозиторий (≥ 3) | 1 |
| **ИТОГО** | | **10** |

**Шкала перевода баллов:**

| Баллы | Оценка |
|-------|--------|
| 9–10 | «5» (отлично) |
| 7–8 | «4» (хорошо) |
| 5–6 | «3» (удовлетворительно) |
| 0–4 | «2» (неудовлетворительно) |

---

## Контрольные вопросы

1. Что такое трёхслойная архитектура и зачем она нужна?
2. Как реализован принцип наследования в проекте?
3. Почему пароль хешируется перед сохранением в БД?
4. Чем серверная валидация отличается от клиентской и зачем нужны обе?
5. Что такое Blueprint в Flask и для чего он используется?
6. Как реализована защита от SQL-инъекций в проекте?
7. Какие роли пользователей есть в системе и как они разграничивают доступ?

---

## Приложение. Полный чек-лист выполнения

| № | Действие | ☐ |
|---|----------|---|
| 1 | Создано виртуальное окружение | ☐ |
| 2 | Установлен Flask, Werkzeug | ☐ |
| 3 | Создана БД `conference.db` | ☐ |
| 4 | Реализован класс `Database` (Singleton) | ☐ |
| 5 | Реализованы валидаторы | ☐ |
| 6 | Реализованы модели с наследованием от `BaseModel` | ☐ |
| 7 | Реализованы маршруты регистрации и авторизации | ☐ |
| 8 | Реализованы маршруты заявок | ☐ |
| 9 | Реализованы маршруты администратора | ☐ |
| 10 | Созданы HTML-шаблоны | ☐ |
| 11 | Созданы CSS и JS | ☐ |
| 12 | Приложение запускается и работает | ☐ |
| 13 | Сделаны 3+ коммита в Git | ☐ |

---

## Возможные проблемы и решения

| Проблема | Решение |
|----------|---------|
| `python3: command not found` | `sudo dnf install python3` |
| `No module named venv` | `sudo dnf install python3-venv` |
| `pip: command not found` | `sudo dnf install python3-pip` |
| Ошибка при установке Flask | Убедитесь, что venv активирован: `source venv/bin/activate` |
| Порт 5000 занят | Измените порт в `app.run(port=5001)` |
| `Permission denied` | Используйте `sudo` только для системных пакетов, не для pip в venv |
| `ModuleNotFoundError: No module named 'backend'` | Запускать из корня проекта: `python -m backend.app` |
| `sqlite3.OperationalError: no such table` | Применить `schema.sql`: `sqlite3 database/conference.db < schema.sql` |
| `werkzeug` не найден | `pip install werkzeug` |
| Ошибка `FOREIGN KEY constraint failed` | Убедиться, что `PRAGMA foreign_keys = ON` в `get_connection()` |
| Порт 5000 занят | Изменить на `port=5001` в `app.py` |
| Ошибка при хешировании пароля | Установить `Werkzeug` свежей версии |

---


## Заключение

В ходе работы вы:
- освоили трёхслойную архитектуру веб-приложения;
- применили принципы ООП на практике;
- реализовали все модули ИС «Конференции.РФ» согласно техническому заданию;
- научились связывать фронтенд с бекендом через REST API;
- оформили результаты в Git-репозиторий.

Разработанный проект готов к дальнейшему расширению (например, добавление отзывов, оптимизация производительности, разработка адаптивного дизайна) в рамках следующих практических работ.


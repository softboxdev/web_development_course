# Практическая работа № 4
## Разработка подсистемы безопасности информационной системы

**Дисциплина:** Информационные системы и программирование
**Специальность:** 09.02.07 «Информационные системы и программирование»
**Тема:** Разработка подсистемы безопасности ИС «Конференции.РФ»
**Время выполнения:** 4 академических часа (180 минут)
**ОС:** РОСА Линукс

---

## Цель и Задачи работы

Обеспечить безопасность информационной системы путём реализации следующих механизмов:

1. **Хеширование паролей** — защита учётных данных пользователей.
2. **Защита от SQL-инъекций** — использование подготовленных выражений (prepared statements).
3. **Разграничение прав доступа** — роли «гость», «участник», «администратор».
4. **Валидация и фильтрация данных** — на стороне сервера и клиента.
5. **Защита сессий и cookies** — настройка безопасных флагов и секретного ключа.

---

## Оснащение рабочего места

| Компонент | Версия / Примечание |
|-----------|---------------------|
| ОС | РОСА Линукс |
| Python | 3.8+ |
| SQLite | 3.x |
| Flask | 2.3.3 |
| Werkzeug | 2.3.7 |
| Git | 2.x |

> **Важно!** Перед началом работы убедитесь, что виртуальное окружение активировано:
> ```bash
> source venv/bin/activate
> ```

---

## Краткие теоретические сведения

### Основные угрозы безопасности веб-приложений

| Угроза | Описание | Последствия |
|--------|----------|-------------|
| **SQL-инъекция** | Внедрение вредоносного SQL-кода через пользовательский ввод | Кража данных, удаление таблиц |
| **XSS** | Внедрение JavaScript в страницы | Кража cookies, сессий |
| **CSRF** | Подделка межсайтовых запросов | Неавторизованные действия |
| **Перехват сессии** | Кража идентификатора сессии | Имперсонация пользователя |
| **Брутфорс** | Подбор пароля | Несанкционированный доступ |

### Принцип наименьших привилегий (Least Privilege)

Каждый пользователь должен иметь **только те права**, которые необходимы ему для выполнения своих задач. Это ключевой принцип разграничения доступа.

### Хеширование vs Шифрование

| Хеширование | Шифрование |
|-------------|------------|
| Необратимо | Обратимо |
| Для паролей | Для данных |
| **bcrypt**, Argon2, PBKDF2 | AES, Fernet |

> **Правило:** Пароли **никогда** не хранятся в открытом виде и **никогда** не шифруются — только хешируются!

---

## Шаг 1. Хеширование паролей (25 мин)

### 1.1. Почему нельзя хранить пароли в открытом виде?

Если база данных будет скомпрометирована, злоумышленник получит все пароли пользователей. Хеширование делает восстановление пароля вычислительно сложным.

### 1.2. Выбор алгоритма хеширования

| Алгоритм | Безопасность | Рекомендация |
|----------|--------------|--------------|
| MD5 | ❌ Небезопасен | Не использовать |
| SHA-1 | ❌ Небезопасен | Не использовать |
| SHA-256 | ⚠️ Условно | Только с солью и итерациями |
| **bcrypt** | ✅ Безопасен | **Рекомендуется** |
| **Argon2** | ✅ Очень безопасен | Для высоких требований |

В нашем проекте используется **Werkzeug** с алгоритмом **bcrypt** (через `generate_password_hash`).

### 1.3. Обновление модели User

Откройте `backend/models/user.py` и убедитесь, что пароль хешируется:

```python
from werkzeug.security import generate_password_hash, check_password_hash

class User(BaseModel):
    # ... код инициализации ...

    def save(self) -> tuple:
        """Сохранение нового пользователя с хешированием пароля."""
        ok, msg = self.validate()
        if not ok:
            return False, msg
        if self.login_exists():
            return False, "Пользователь с таким логином уже существует"
        if self.email_exists():
            return False, "Пользователь с таким email уже существует"

        # ХЕШИРОВАНИЕ ПАРОЛЯ
        password_hash = generate_password_hash(
            self.password,
            method='pbkdf2:sha256:600000',  # или 'bcrypt'
            salt_length=16
        )

        user_id = self.db.execute(
            """INSERT INTO user (login, password_hash, email, phone, role)
               VALUES (?, ?, ?, ?, ?)""",
            (self.login, password_hash, self.email, self.phone, self.role)
        )
        self.id = user_id
        return True, user_id

    @staticmethod
    def authenticate(login: str, password: str) -> dict:
        """Аутентификация: сравнение введённого пароля с хешем."""
        db = Database()
        user = db.fetch_one("SELECT * FROM user WHERE login = ?", (login,))
        if user and check_password_hash(user['password_hash'], password):
            return user
        return None
```

### 1.4. Проверка хеширования

Создайте тестовый скрипт `test_hash.py`:

```python
from werkzeug.security import generate_password_hash, check_password_hash

# Хеширование
password = "Demo77"
hashed = generate_password_hash(password, method='pbkdf2:sha256:600000', salt_length=16)
print(f"Пароль: {password}")
print(f"Хеш: {hashed}")
print(f"Длина хеша: {len(hashed)}")

# Проверка
print(f"Верный пароль: {check_password_hash(hashed, 'Demo77')}")
print(f"Неверный пароль: {check_password_hash(hashed, 'wrong')}")
```

**Запустите:**

```bash
python test_hash.py
```

**Ожидаемый результат:** хеш начинается с `pbkdf2:sha256:600000$...`, проверка верного пароля — `True`, неверного — `False`.

---

## Шаг 2. Защита от SQL-инъекций (20 мин)

### 2.1. Что такое SQL-инъекция?

**SQL-инъекция** — это внедрение вредоносного SQL-кода через пользовательский ввод, который приложение некорректно обрабатывает.

**Уязвимый код (НЕ ИСПОЛЬЗОВАТЬ!):**

```python
# ❌ ОПАСНО! SQL-инъекция возможна
query = f"SELECT * FROM user WHERE login = '{login}'"
db.execute(query)
```

**Атака:** если пользователь введёт логин `' OR '1'='1`, запрос станет:
```sql
SELECT * FROM user WHERE login = '' OR '1'='1'
-- Вернёт ВСЕХ пользователей!
```

### 2.2. Защита через подготовленные выражения

**Правильный код (подготовленные выражения):**

```python
# ✅ БЕЗОПАСНО! Параметры передаются отдельно
query = "SELECT * FROM user WHERE login = ?"
db.execute(query, (login,))
```

Параметры **никогда не подставляются** в строку запроса. База данных обрабатывает их как **данные**, а не как SQL-код .

### 2.3. Проверка класса Database

Откройте `backend/database.py` и убедитесь, что все методы используют параметры:

```python
class Database:
    # ... код ...

    def execute(self, query, params=()):
        """Выполнить INSERT/UPDATE/DELETE с параметрами."""
        with self.get_connection() as conn:
            cursor = conn.execute(query, params)
            conn.commit()
            return cursor.lastrowid

    def fetch_one(self, query, params=()):
        """Получить одну запись с параметрами."""
        with self.get_connection() as conn:
            cursor = conn.execute(query, params)
            row = cursor.fetchone()
            return dict(row) if row else None

    def fetch_all(self, query, params=()):
        """Получить все записи с параметрами."""
        with self.get_connection() as conn:
            cursor = conn.execute(query, params)
            return [dict(row) for row in cursor.fetchall()]
```

### 2.4. Проверка всех запросов

Пройдитесь по всем моделям и убедитесь, что **нигде** нет `f-строк` или конкатенации в SQL-запросах:

| Файл | Проверить |
|------|-----------|
| `user.py` | ✅ Все запросы с `?` |
| `request.py` | ✅ Все запросы с `?` |
| `event.py` | ✅ Все запросы с `?` |
| `review.py` | ✅ Все запросы с `?` |

### 2.5. Тест на SQL-инъекцию

Создайте `test_sql_injection.py`:

```python
from backend.database import Database

db = Database()

# Попытка SQL-инъекции
malicious_login = "' OR '1'='1"
result = db.fetch_all(
    "SELECT * FROM user WHERE login = ?",
    (malicious_login,)
)
print(f"Результат инъекции: {len(result)} записей")
# Ожидается: 0 записей (инъекция не сработала)
```

**Запустите:**

```bash
python test_sql_injection.py
```

**Ожидаемый результат:** `Результат инъекции: 0 записей` — атака предотвращена.

---

## Шаг 3. Разграничение прав доступа (30 мин)

### 3.1. Роли и права

| Роль | Просмотр заявок | Создание заявок | Управление заявками |
|------|-----------------|-----------------|---------------------|
| **Гость** (неавторизован) | ❌ | ❌ | ❌ |
| **Участник** (user) | ✅ Только свои | ✅ | ❌ |
| **Администратор** (admin) | ✅ Все | ❌ | ✅ Смена статуса |

### 3.2. Создание декораторов для проверки прав

Создайте файл `backend/auth/decorators.py`:

```python
from functools import wraps
from flask import session, jsonify, redirect, url_for


def login_required(f):
    """Декоратор: требует авторизации."""
    @wraps(f)
    def decorated_function(*args, **kwargs):
        if 'user_id' not in session:
            # Для API-запросов — JSON-ответ
            if request.path.startswith('/api/'):
                return jsonify({
                    'success': False,
                    'message': 'Требуется авторизация'
                }), 401
            # Для страниц — редирект на логин
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

> **Примечание:** не забудьте добавить `from flask import request` в начало файла.

### 3.3. Применение декораторов к маршрутам

Обновите `backend/routes/request_routes.py`:

```python
from flask import Blueprint, request, jsonify, render_template, session
from backend.auth.decorators import (
    login_required, participant_required
)
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
    new_request = Request(
        user_id=session['user_id'],
        event_id=int(data.get('event_id')),
        status_id=1
    )
    request_id = new_request.save()
    return jsonify({'success': True, 'request_id': request_id}), 201
```

Обновите `backend/routes/admin_routes.py`:

```python
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
    status_id = int(data.get('status_id'))
    if status_id not in (1, 2, 3):
        return jsonify({
            'success': False,
            'message': 'Некорректный статус'
        }), 400
    Request.update_status(request_id, status_id)
    return jsonify({'success': True}), 200
```

### 3.4. Проверка разграничения прав

| Сценарий | Ожидаемый результат |
|----------|---------------------|
| Гость открывает `/dashboard` | Редирект на `/api/auth/login` |
| Гость запрашивает `/api/requests` | JSON: `{"success": false}`, 401 |
| Участник открывает `/admin` | JSON: `{"success": false}`, 403 |
| Администратор открывает `/admin` | Панель администратора |
| Участник создаёт заявку | ✅ Успешно |
| Участник пытается сменить статус | JSON: 403 Доступ запрещён |

---

## Шаг 4. Валидация и фильтрация данных (25 мин)

### 4.1. Валидация на сервере

Серверная валидация — **основной рубеж защиты**. Клиентскую валидацию можно обойти, отключив JavaScript или используя инструменты разработчика .

Обновите `backend/validators/validators.py`:

```python
import re

class Validator:
    """Валидация пользовательских данных."""

    # Белый список допустимых символов
    LOGIN_PATTERN = re.compile(r'^[A-Za-z0-9]{6,50}$')
    EMAIL_PATTERN = re.compile(r'^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$')
    PHONE_PATTERN = re.compile(r'^8\(\d{3}\)\d{3}-\d{2}-\d{2}$')
    FIO_PATTERN = re.compile(r'^[А-Яа-яЁё\s]{2,150}$')

    @staticmethod
    def sanitize_string(value: str, max_length: int = 255) -> str:
        """Очистка строки от потенциально опасных символов."""
        if not value:
            return ""
        # Удаление управляющих символов
        cleaned = re.sub(r'[\x00-\x1f\x7f]', '', value)
        # Ограничение длины
        return cleaned[:max_length].strip()

    @staticmethod
    def validate_login(login: str) -> tuple:
        """Логин: латиница и цифры, от 6 до 50 символов."""
        login = Validator.sanitize_string(login, 50)
        if not Validator.LOGIN_PATTERN.match(login):
            return False, "Логин: от 6 до 50 символов, только латиница и цифры"
        return True, ""

    @staticmethod
    def validate_password(password: str) -> tuple:
        """Пароль: минимум 8 символов, максимум 128."""
        if not password or len(password) < 8:
            return False, "Пароль должен содержать минимум 8 символов"
        if len(password) > 128:
            return False, "Пароль слишком длинный (максимум 128 символов)"
        return True, ""

    @staticmethod
    def validate_email(email: str) -> tuple:
        """Email: проверка формата."""
        email = Validator.sanitize_string(email, 100)
        if not Validator.EMAIL_PATTERN.match(email):
            return False, "Некорректный email"
        return True, ""

    @staticmethod
    def validate_phone(phone: str) -> tuple:
        """Телефон: формат 8(XXX)XXX-XX-XX."""
        phone = Validator.sanitize_string(phone, 18)
        if not Validator.PHONE_PATTERN.match(phone):
            return False, "Телефон в формате 8(XXX)XXX-XX-XX"
        return True, ""

    @staticmethod
    def validate_fio(fio: str) -> tuple:
        """ФИО: кириллица и пробелы."""
        fio = Validator.sanitize_string(fio, 150)
        if not Validator.FIO_PATTERN.match(fio):
            return False, "ФИО: только кириллица и пробелы"
        return True, ""

    @staticmethod
    def validate_rating(rating) -> tuple:
        """Оценка: целое число от 1 до 5."""
        try:
            rating = int(rating)
            if 1 <= rating <= 5:
                return True, ""
            return False, "Оценка должна быть от 1 до 5"
        except (ValueError, TypeError):
            return False, "Оценка должна быть числом"
```

### 4.2. Валидация на клиенте

Клиентская валидация — **первый рубеж** (для удобства пользователя) .

Обновите `frontend/static/js/register.js`:

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

    // Клиентская валидация (белый список)
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

    // Отправка на сервер (серверная валидация обязательна!)
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
});

function showError(text) {
    const msg = document.getElementById('message');
    msg.className = 'error';
    msg.textContent = text;
}
```

### 4.3. Проверка серверной валидации

Даже если клиентская валидация пропущена (например, через отключение JavaScript), сервер должен отклонить некорректные данные:

```python
# backend/routes/auth_routes.py

@auth_bp.route('/register', methods=['POST'])
def register():
    data = request.get_json()

    # Серверная валидация через модель User
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
```

---

## Шаг 5. Защита сессий и cookies (25 мин)

### 5.1. Настройка безопасных cookies

Flask по умолчанию подписывает cookies с помощью `SECRET_KEY`, но **не устанавливает** флаги `Secure`, `SameSite` . Их нужно настроить вручную.

Обновите `backend/app.py`:

```python
import os
from flask import Flask

app = Flask(
    __name__,
    template_folder='../frontend/templates',
    static_folder='../frontend/static'
)

# ═══════════════════════════════════════════════════════════
# БЕЗОПАСНАЯ НАСТРОЙКА СЕССИЙ И COOKIES
# ═══════════════════════════════════════════════════════════

# Секретный ключ: длинный, случайный, из переменной окружения
app.secret_key = os.environ.get(
    'FLASK_SECRET_KEY',
    'dev-fallback-key-change-in-production-32bytes-min'
)

# Флаги безопасности cookies
app.config.update(
    # Только HTTPS (в разработке False, в продакшене True)
    SESSION_COOKIE_SECURE=False,  # Для production: True

    # Запрет доступа из JavaScript (защита от XSS)
    SESSION_COOKIE_HTTPONLY=True,

    # Защита от CSRF: cookies не отправляются при cross-site запросах
    SESSION_COOKIE_SAMESITE='Lax',

    # Время жизни сессии: 30 минут
    PERMANENT_SESSION_LIFETIME=1800,

    # Имя cookie
    SESSION_COOKIE_NAME='conf_session',
)

# ═══════════════════════════════════════════════════════════
# РЕГИСТРАЦИЯ BLUEPRINTS
# ═══════════════════════════════════════════════════════════

from backend.routes.auth_routes import auth_bp
from backend.routes.request_routes import request_bp
from backend.routes.admin_routes import admin_bp
from backend.models.user import User

app.register_blueprint(auth_bp, url_prefix='/api/auth')
app.register_blueprint(request_bp)
app.register_blueprint(admin_bp)


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
        admin.save()
        print("✓ Администратор Conf2027 создан")


if __name__ == '__main__':
    init_admin()
    app.run(debug=False, host='127.0.0.1', port=5000)
```

### 5.2. Генерация надёжного SECRET_KEY

**Никогда** не используйте `SECRET_KEY` по умолчанию в продакшене! Сгенерируйте его:

```bash
python -c "import secrets; print(secrets.token_hex(32))"
```

**Пример результата:**
```
a3f8b2c1d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1
```

Сохраните ключ в файл `.env`:

```bash
echo "FLASK_SECRET_KEY=<ваш_сгенерированный_ключ>" > .env
```

И загружайте его:

```python
from dotenv import load_dotenv
load_dotenv()
app.secret_key = os.environ['FLASK_SECRET_KEY']
```

> **Установка python-dotenv:**
> ```bash
> pip install python-dotenv
> ```

### 5.3. Настройка сессии при входе

Обновите `backend/routes/auth_routes.py`:

```python
from flask import session

@auth_bp.route('/login', methods=['POST'])
def login():
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

    # Создание новой сессии
    session['user_id'] = user['id']
    session['role'] = user['role']
    session['login'] = user['login']

    # Сессия постоянная (с ограниченным временем жизни)
    session.permanent = True

    return jsonify({
        'success': True,
        'role': user['role'],
        'redirect': '/admin' if user['role'] == 'admin' else '/dashboard'
    }), 200
```

### 5.4. Защита от CSRF (дополнительно)

Для форм, отправляемых через HTML (не JSON), рекомендуется использовать CSRF-токены.

**Установите Flask-WTF:**

```bash
pip install Flask-WTF
```

**Обновите `app.py`:**

```python
from flask_wtf.csrf import CSRFProtect

app = Flask(__name__)
app.secret_key = os.environ.get('FLASK_SECRET_KEY', 'dev-key')

# Включение CSRF-защиты
csrf = CSRFProtect(app)
```

Теперь все POST/PUT/DELETE запросы должны содержать CSRF-токен. Для JSON-API можно исключить маршруты:

```python
csrf.exempt(auth_bp)
```

### 5.5. Проверка защиты сессий

| Проверка | Ожидаемый результат |
|----------|---------------------|
| Cookie `HttpOnly` | JavaScript не может прочитать `document.cookie` |
| Cookie `SameSite=Lax` | Cookies не отправляются при cross-site запросах |
| Время жизни 30 минут | Сессия истекает после 30 минут бездействия |
| `session.clear()` при входе | Старая сессия не может быть переиспользована |

**Проверка через браузер:**
1. Откройте DevTools (F12)
2. Перейдите на вкладку **Application** → **Cookies**
3. Найдите cookie `conf_session`
4. Убедитесь, что флаг **HttpOnly** установлен ✅

---

## Шаг 6. Коммиты в репозиторий (10 мин)

Сделайте коммиты по этапам работы:

```bash
# Коммит 1: Хеширование паролей
git add backend/models/user.py
git commit -m "ПР №4: хеширование паролей через Werkzeug (pbkdf2:sha256)"

# Коммит 2: Защита от SQL-инъекций
git add backend/database.py
git commit -m "ПР №4: подготовленные выражения во всех методах Database"

# Коммит 3: Разграничение прав доступа
git add backend/auth/decorators.py
git add backend/routes/request_routes.py
git add backend/routes/admin_routes.py
git commit -m "ПР №4: декораторы login_required, admin_required, participant_required"

# Коммит 4: Валидация данных
git add backend/validators/validators.py
git add frontend/static/js/register.js
git commit -m "ПР №4: серверная и клиентская валидация (белый список)"

# Коммит 5: Защита сессий и cookies
git add backend/app.py
git add backend/routes/auth_routes.py
git add .env
git commit -m "ПР №4: безопасные cookies, SECRET_KEY из env, session.clear()"

# Отправка на удалённый репозиторий
git push origin main
```

---

## Требования к результату

1. **Хеширование паролей** — пароли в БД хранятся в виде хешей (`pbkdf2:sha256` или `bcrypt`).
2. **Защита от SQL-инъекций** — все SQL-запросы используют параметры `?`.
3. **Разграничение прав** — декораторы проверяют роль перед доступом.
4. **Валидация** — на сервере (модели) и на клиенте (JavaScript).
5. **Защита сессий** — `HttpOnly`, `SameSite`, ограниченное время жизни, `session.clear()`.
6. **Коммиты** — минимум 5 коммитов в Git.

---

## Критерии оценивания

| № | Критерий | Баллы |
|---|----------|-------|
| 1 | Хеширование паролей (Werkzeug/bcrypt) | 2 |
| 2 | Защита от SQL-инъекций (подготовленные выражения) | 2 |
| 3 | Разграничение прав (декораторы, роли) | 2 |
| 4 | Валидация на сервере и клиенте | 2 |
| 5 | Защита сессий (флаги cookies, SECRET_KEY) | 1 |
| 6 | Коммиты (≥ 5) | 1 |
| **ИТОГО** | | **10** |

---

## Контрольные вопросы

1. Почему пароли нельзя хранить в открытом виде или шифровать?
2. Как подготовленные выражения защищают от SQL-инъекций?
3. Чем отличается валидация на клиенте от валидации на сервере?
4. Зачем нужен флаг `HttpOnly` для cookies?
5. Что такое CSRF и как от него защититься?
6. Почему `SECRET_KEY` должен храниться в переменной окружения?
7. Какие роли есть в системе и как разграничиваются их права?

---

## Возможные проблемы и решения

| Проблема | Решение |
|----------|---------|
| `SECRET_KEY` не загружается из `.env` | Установите `python-dotenv`, добавьте `load_dotenv()` |
| Cookie `Secure` не работает | В разработке используйте `SESSION_COOKIE_SECURE=False` |
| CSRF-токен отклоняется | Убедитесь, что `Flask-WTF` установлен и `CSRFProtect` инициализирован |
| Ошибка `ForeignKey constraint` | Включите `PRAGMA foreign_keys = ON` в `get_connection()`  |
| SQL-инъекция всё ещё возможна | Проверьте ВСЕ запросы на наличие `?` и параметров |

---

## Заключение

В ходе работы вы реализовали полноценную подсистему безопасности ИС «Конференции.РФ»:

- **Пароли** защищены надёжным хешированием.
- **SQL-запросы** неуязвимы для инъекций.
- **Права доступа** разграничены по ролям.
- **Данные** валидируются на двух уровнях.
- **Сессии** защищены от перехвата и XSS.


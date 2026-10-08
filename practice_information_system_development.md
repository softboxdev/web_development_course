# Практическая работа № 5

## Разработка веб-приложения в соответствии с техническим заданием

**Дисциплина:** Информационные системы и программирование  
**Специальность:** 09.02.07 «Информационные системы и программирование»  
**Тема:** Разработка полного веб-приложения ИС «Конференции.РФ»  
**Время выполнения:** 4 академических часа (180 минут)  
**ОС:** РОСА Линукс

---

## Цель работы

Разработать **полнофункциональное веб-приложение** для портала «Конференции.РФ», объединив все модули из предыдущих практических работ:

1. Интегрировать фронтенд с бекендом.
2. Реализовать полный CRUD (Create, Read, Update, Delete) для работы с БД.
3. Реализовать смену статусов заявок: **Новая → Мероприятие назначено → Завершено**.
4. Реализовать отзывы после завершения мероприятия.

---

## Оснащение рабочего места

| Компонент | Версия / Примечание |
|-----------|---------------------|
| ОС | РОСА Линукс |
| Python | 3.8+ |
| SQLite / PostgreSQL | 3.x / 12+ |
| Flask | 2.3.3 |
| Werkzeug | 2.3.7 |
| Git | 2.x |

> **Важно!** Перед началом работы зайдите в папку рабочего проекта и активируйте виртуальное окружение:
> ```bash
> cd ~/conference_portal
> source venv/bin/activate
> ```

---

## Краткие теоретические сведения

### Что такое CRUD?

**CRUD** — это четыре базовые операции работы с данными:

| Операция | SQL-команда | HTTP-метод | Пример в проекте |
|----------|-------------|-----------|------------------|
| **C**reate | `INSERT` | POST | Создание заявки |
| **R**ead | `SELECT` | GET | Просмотр заявок |
| **U**pdate | `UPDATE` | PUT | Смена статуса |
| **D**elete | `DELETE` | DELETE | Удаление отзыва |

### Жизненный цикл заявки

```
┌─────────────┐    ┌─────────────────────┐    ┌────────────┐
│   НОВАЯ     │───►│ МЕРОПРИЯТИЕ         │───►│ ЗАВЕРШЕНО  │
│  (status=1) │    │ НАЗНАЧЕНО (status=2)│    │ (status=3) │
└─────────────┘    └─────────────────────┘    └────────────┘
                                                      │
                                                      ▼
                                              ┌──────────────┐
                                              │   ОТЗЫВ      │
                                              │  (review)    │
                                              └──────────────┘
```

### Интеграция фронтенда с бекендом

```
ФРОНТЕНД (HTML/JS)          БЕКЕНД (Flask)           БД (SQLite)
     │                            │                       │
     │  fetch('/api/...')         │                       │
     ├───────────────────────────►│                       │
     │                            │  SQL-запрос           │
     │                            ├──────────────────────►│
     │                            │                       │
     │                            │◄──────────────────────┤
     │  JSON-ответ                │                       │
     │◄───────────────────────────┤                       │
     │                            │                       │
     │  Обновление DOM            │                       │
     ▼                            ▼                       ▼
```

---

## Шаг 1. Проверка готовности проекта (10 мин)

### 1.1. Проверка структуры

Убедитесь, что структура проекта соответствует ПР № 3:

```bash
tree -L 3 -I 'venv|__pycache__|*.pyc'
```

**Ожидаемая структура:**

```
conference_portal/
├── backend/
│   ├── app.py
│   ├── database.py
│   ├── auth/
│   │   └── decorators.py
│   ├── models/
│   │   ├── base_model.py
│   │   ├── user.py
│   │   ├── event.py
│   │   ├── request.py
│   │   ├── status.py
│   │   └── review.py
│   ├── validators/
│   │   └── validators.py
│   └── routes/
│       ├── auth_routes.py
│       ├── request_routes.py
│       ├── admin_routes.py
│       └── review_routes.py   ← добавим в этой ПР
├── frontend/
│   ├── static/
│   │   ├── css/style.css
│   │   └── js/
│   │       ├── register.js
│   │       ├── login.js
│   │       ├── request.js
│   │       ├── admin.js
│   │       └── review.js       ← добавим
│   └── templates/
│       ├── register.html
│       ├── login.html
│       ├── dashboard.html
│       ├── create_request.html
│       ├── admin.html
│       └── review.html         ← добавим
├── database/conference.db
├── schema.sql
└── requirements.txt
```

### 1.2. Проверка базы данных

```bash
sqlite3 database/conference.db ".tables"
```

**Ожидаемый результат:**
```
event  request  review  status  user
```

### 1.3. Запуск приложения

```bash
python -m backend.app
```

Откройте браузер: http://127.0.0.1:5000

**Проверьте:**
- Регистрация работает
- Авторизация работает
- Вход под `Conf2027 / Demo77` открывает админ-панель

**Остановите:** `Ctrl+C`

---

## Шаг 2. Наполнение БД тестовыми мероприятиями (15 мин)

Для полноценной работы приложения нужны помещения (аудитории, коворкинги, кинозалы).

### 2.1. Создание скрипта `seed_events.py`

В корне проекта создайте файл:

```python
from backend.models.event import Event
from backend.database import Database


def seed_events():
    """Наполнение БД тестовыми мероприятиями."""
    db = Database()
    
    # Проверка: есть ли уже мероприятия
    existing = db.fetch_all("SELECT COUNT(*) as cnt FROM event")
    if existing[0]['cnt'] > 0:
        print(f"✓ Мероприятия уже есть: {existing[0]['cnt']} записей")
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
        event = Event(**data)
        event.save()
        print(f"  + Добавлено: {data['name']}")
    
    print(f"✓ Загружено {len(events)} мероприятий")


if __name__ == '__main__':
    seed_events()
```

### 2.2. Запуск скрипта

```bash
python seed_events.py
```

**Ожидаемый вывод:**

```
  + Добавлено: Аудитория №101
  + Добавлено: Коворкинг «Прогресс»
  + Добавлено: Кинозал «Октябрь»
  + Добавлено: Конференц-зал «Восток»
  + Добавлено: Коворкинг «Цифра»
✓ Загружено 5 мероприятий
```

### 2.3. Проверка

```bash
sqlite3 database/conference.db "SELECT id, name, place FROM event;"
```

---

## Шаг 3. Полный CRUD для заявок (40 мин)

### 3.1. Создание заявки (Create) — уже реализовано

Метод `Request.save()` из ПР № 3 создаёт новую заявку со статусом «Новая» (status_id=1).

### 3.2. Чтение заявок (Read) — расширим

Откройте `backend/models/request.py` и **добавьте** методы:

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
               VALUES (?, ?, ?)""",
            (self.user_id, self.event_id, self.status_id)
        )
        self.id = request_id
        return request_id

    @classmethod
    def get_by_user(cls, user_id: int):
        """Все заявки пользователя с деталями."""
        db = Database()
        return db.fetch_all(
            """SELECT r.id, e.name AS event_name, e.date,
                      e.place, s.name AS status_name,
                      r.status_id, r.created_at
               FROM request r
               JOIN event e ON r.event_id = e.id
               JOIN status s ON r.status_id = s.id
               WHERE r.user_id = ?
               ORDER BY r.created_at DESC""",
            (user_id,)
        )

    @classmethod
    def get_all_with_details(cls):
        """Все заявки (для админа)."""
        db = Database()
        return db.fetch_all(
            """SELECT r.id, u.login, u.email, e.name AS event_name,
                      e.date, e.place, s.name AS status_name,
                      r.status_id, r.created_at
               FROM request r
               JOIN user u ON r.user_id = u.id
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
               JOIN user u ON r.user_id = u.id
               JOIN event e ON r.event_id = e.id
               JOIN status s ON r.status_id = s.id
               WHERE r.id = ?""",
            (request_id,)
        )

    @classmethod
    def update_status(cls, request_id: int, status_id: int) -> bool:
        """Смена статуса заявки."""
        db = Database()
        db.execute(
            "UPDATE request SET status_id = ? WHERE id = ?",
            (status_id, request_id)
        )
        return True

    @classmethod
    def delete(cls, request_id: int) -> bool:
        """Удаление заявки (только для администратора)."""
        db = Database()
        db.execute("DELETE FROM request WHERE id = ?", (request_id,))
        return True

    @classmethod
    def can_be_reviewed(cls, request_id: int) -> bool:
        """Проверка: можно ли оставить отзыв (статус «Завершено»)."""
        db = Database()
        row = db.fetch_one(
            """SELECT status_id FROM request WHERE id = ?""",
            (request_id,)
        )
        if not row:
            return False
        return row['status_id'] == 3  # Завершено
```

### 3.3. Обновление маршрутов заявок

Откройте `backend/routes/request_routes.py`:

```python
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

    # Валидация
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

### 3.4. Обновление админ-маршрутов

Откройте `backend/routes/admin_routes.py`:

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
    status_id = data.get('status_id')
    
    if status_id not in (1, 2, 3):
        return jsonify({
            'success': False,
            'message': 'Некорректный статус'
        }), 400
    
    # Проверка существования заявки
    req = Request.get_by_id(request_id)
    if not req:
        return jsonify({'success': False, 'message': 'Заявка не найдена'}), 404
    
    # Проверка допустимых переходов статусов
    current = req['status_id']
    new_status = int(status_id)
    
    allowed_transitions = {
        1: [2],       # Новая → Мероприятие назначено
        2: [3],       # Мероприятие назначено → Завершено
        3: [],        # Завершено — финальный статус
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
        return jsonify({'success': False, 'message': 'Заявка не найдена'}), 404
    
    Request.delete(request_id)
    return jsonify({'success': True}), 200
```

---

## Шаг 4. Реализация отзывов (30 мин)

### 4.1. Модель Review

Откройте `backend/models/review.py` и **дополните**:

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
        """Валидация отзыва."""
        if not self.text or len(self.text.strip()) < 5:
            return False, "Текст отзыва должен содержать минимум 5 символов"
        if len(self.text) > 2000:
            return False, "Текст отзыва слишком длинный (максимум 2000 символов)"
        if not self.rating or not (1 <= int(self.rating) <= 5):
            return False, "Оценка должна быть от 1 до 5"
        return True, ""

    def save(self) -> tuple:
        """Сохранение отзыва с валидацией."""
        ok, msg = self.validate()
        if not ok:
            return False, msg
        
        # Проверка: заявка существует и завершена
        from backend.models.request import Request
        req = Request.get_by_id(self.request_id)
        if not req:
            return False, "Заявка не найдена"
        if req['status_id'] != 3:
            return False, "Отзыв можно оставить только после завершения мероприятия"
        
        # Проверка: отзыв ещё не оставлен
        if self.exists_for_request(self.request_id):
            return False, "Отзыв для этой заявки уже оставлен"
        
        review_id = self.db.execute(
            """INSERT INTO review (request_id, text, rating)
               VALUES (?, ?, ?)""",
            (self.request_id, self.text.strip(), int(self.rating))
        )
        self.id = review_id
        return True, review_id

    @staticmethod
    def exists_for_request(request_id: int) -> bool:
        """Есть ли уже отзыв для заявки."""
        db = Database()
        row = db.fetch_one(
            "SELECT id FROM review WHERE request_id = ?",
            (request_id,)
        )
        return row is not None

    @classmethod
    def get_by_request(cls, request_id: int):
        """Отзыв по заявке."""
        db = Database()
        return db.fetch_one(
            """SELECT id, text, rating, created_at
               FROM review WHERE request_id = ?""",
            (request_id,)
        )

    @classmethod
    def get_all_with_details(cls):
        """Все отзывы с деталями (для админа)."""
        db = Database()
        return db.fetch_all(
            """SELECT rv.id, rv.text, rv.rating, rv.created_at,
                      u.login, e.name AS event_name
               FROM review rv
               JOIN request r ON rv.request_id = r.id
               JOIN user u ON r.user_id = u.id
               JOIN event e ON r.event_id = e.id
               ORDER BY rv.created_at DESC"""
        )

    @classmethod
    def delete(cls, review_id: int) -> bool:
        db = Database()
        db.execute("DELETE FROM review WHERE id = ?", (review_id,))
        return True
```

### 4.2. Маршруты отзывов

Создайте файл `backend/routes/review_routes.py`:

```python
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
    
    # Проверка принадлежности заявки пользователю
    req = Request.get_by_id(int(request_id))
    if not req:
        return jsonify({'success': False, 'message': 'Заявка не найдена'}), 404
    if req['user_id'] != session['user_id']:
        return jsonify({'success': False, 'message': 'Доступ запрещён'}), 403
    
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

### 4.3. Подключение blueprint в `app.py`

Откройте `backend/app.py` и **добавьте**:

```python
from backend.routes.review_routes import review_bp

# Регистрация blueprints
app.register_blueprint(auth_bp, url_prefix='/api/auth')
app.register_blueprint(request_bp)
app.register_blueprint(admin_bp)
app.register_blueprint(review_bp)  # ← добавить
```

---

## Шаг 5. Обновление фронтенда (30 мин)

### 5.1. Обновлённый `dashboard.html`

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
        
        // Кнопка «Удалить» — только для статуса «Новая»
        if (r.status_id === 1) {
            actions = `<button onclick="deleteRequest(${r.id})" 
                       class="btn btn-danger btn-sm">Удалить</button>`;
        }
        // Кнопка «Оставить отзыв» — только для статуса «Завершено»
        else if (r.status_id === 3) {
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

### 5.2. Обновлённый `admin.html`

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
        // Формируем список доступных статусов
        let statusOptions = '';
        for (const [id, name] of Object.entries(STATUS_NAMES)) {
            const selected = r.status_id == id ? 'selected' : '';
            statusOptions += `<option value="${id}" ${selected}>${name}</option>`;
        }
        
        // Блокировка смены статуса, если уже «Завершено»
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
    if (!data.success) {
        alert(data.message);
    }
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

### 5.3. Шаблон `review.html`

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
            <p><strong>Оценка:</strong> {{ '★' * review.rating }}{{ '☆' * (5 - review.rating) }}</p>
            <p><strong>Текст:</strong> {{ review.text }}</p>
            <p><strong>Дата:</strong> {{ review.created_at }}</p>
        </div>
        <a href="/dashboard" class="btn">← Вернуться к заявкам</a>
    {% else %}
        <form id="reviewForm">
            <input type="hidden" id="request_id" value="{{ request_id }}">
            
            <label>Оценка:</label>
            <div class="rating">
                <input type="radio" name="rating" value="5" id="r5" required>
                <label for="r5">★★★★★ Отлично</label>
                <input type="radio" name="rating" value="4" id="r4">
                <label for="r4">★★★★ Хорошо</label>
                <input type="radio" name="rating" value="3" id="r3">
                <label for="r3">★★★ Удовлетворительно</label>
                <input type="radio" name="rating" value="2" id="r2">
                <label for="r2">★★ Плохо</label>
                <input type="radio" name="rating" value="1" id="r1">
                <label for="r1">★ Очень плохо</label>
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

### 5.4. Дополнение CSS

Откройте `frontend/static/css/style.css` и **добавьте**:

```css
.container-wide {
    max-width: 1200px;
}

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

.btn {
    display: inline-block;
    padding: 8px 16px;
    border-radius: 5px;
    text-decoration: none;
    border: none;
    cursor: pointer;
    font-size: 14px;
}

.btn-primary   { background: #2980b9; color: #fff; }
.btn-secondary { background: #95a5a6; color: #fff; }
.btn-danger    { background: #e74c3c; color: #fff; }
.btn-success   { background: #27ae60; color: #fff; }
.btn-sm        { padding: 4px 10px; font-size: 12px; }

.btn:hover { opacity: 0.9; }

/* Статусы */
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

/* Форма отзыва */
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
}

.rating label:hover {
    background: #f4f6f8;
}

textarea {
    width: 100%;
    padding: 10px;
    font-family: inherit;
    font-size: 14px;
    border: 1px solid #ddd;
    border-radius: 5px;
    box-sizing: border-box;
    resize: vertical;
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

## Шаг 6. Тестирование полного функционала (20 мин)

### 6.1. Сценарий 1: Регистрация нового пользователя

1. Откройте http://127.0.0.1:5000/api/auth/register
2. Заполните форму:
   - Логин: `testuser2027`
   - Пароль: `password123`
   - Email: `testuser@example.ru`
   - Телефон: `8(999)111-22-33`
3. Нажмите **«Создать пользователя»**

**Ожидаемый результат:** сообщение «Пользователь успешно создан», редирект на логин.

### 6.2. Сценарий 2: Создание заявки

1. Войдите под `testuser2027 / password123`
2. Нажмите **«+ Создать заявку»**
3. Выберите **«Аудитория №101»**
4. Нажмите **«Отправить»**

**Ожидаемый результат:** сообщение «Заявка отправлена администратору», редирект на dashboard.

### 6.3. Сценарий 3: Смена статуса администратором

1. Войдите под `Conf2027 / Demo77`
2. Откройте `/admin`
3. Найдите заявку от `testuser2027`
4. Смените статус: **Новая → Мероприятие назначено**

**Ожидаемый результат:** статус в таблице обновился.

5. Смените статус: **Мероприятие назначено → Завершено**

### 6.4. Сценарий 4: Оставление отзыва

1. Войдите под `testuser2027`
2. Откройте `/dashboard`
3. Найдите заявку со статусом «Завершено»
4. Нажмите **«Оставить отзыв»**
5. Поставьте оценку ★★★★★
6. Напишите текст: «Отличное мероприятие, всё понравилось!»
7. Нажмите **«Отправить отзыв»**

**Ожидаемый результат:** сообщение «Спасибо за отзыв!», редирект на dashboard.

### 6.5. Сценарий 5: Просмотр отзыва администратором

1. Войдите под `Conf2027 / Demo77`
2. Откройте `/admin`
3. Прокрутите вниз до таблицы **«Отзывы»**

**Ожидаемый результат:** отзыв от `testuser2027` с оценкой ★★★★★.

### 6.6. Проверка через БД

```bash
sqlite3 database/conference.db <<EOF
SELECT 'Заявки:';
SELECT id, user_id, event_id, status_id FROM request;
SELECT 'Отзывы:';
SELECT id, request_id, rating FROM review;
EOF
```

---

## Шаг 7. Коммиты в Git (10 мин)

```bash
# Коммит 1: Тестовые данные
git add seed_events.py
git commit -m "ПР №5: наполнение БД тестовыми мероприятиями"

# Коммит 2: CRUD для заявок
git add backend/models/request.py
git add backend/routes/request_routes.py
git commit -m "ПР №5: полный CRUD для заявок (чтение, удаление)"

# Коммит 3: Смена статусов
git add backend/routes/admin_routes.py
git commit -m "ПР №5: проверка допустимых переходов статусов"

# Коммит 4: Отзывы
git add backend/models/review.py
git add backend/routes/review_routes.py
git add backend/app.py
git commit -m "ПР №5: модуль отзывов (создание, чтение, удаление)"

# Коммит 5: Фронтенд
git add frontend/
git commit -m "ПР №5: интеграция фронтенда с бекендом (dashboard, admin, review)"

# Отправка
git push origin main
```

---

## Требования к результату

1. **Работающее приложение** по адресу http://127.0.0.1:5000
2. **Полный CRUD** для заявок:
   - ✅ Create — создание заявки
   - ✅ Read — просмотр своих / всех заявок
   - ✅ Update — смена статуса
   - ✅ Delete — удаление заявки
3. **Смена статусов** с проверкой допустимых переходов:
   - Новая (1) → Мероприятие назначено (2) → Завершено (3)
4. **Отзывы** после завершения:
   - Оценка ★1–★★★★★
   - Текст от 5 до 2000 символов
   - Один отзыв на заявку
5. **Интеграция фронтенда с бекендом** через fetch API
6. **Коммиты** — минимум 5


---

## Возможные проблемы и решения

| Проблема | Решение |
|----------|---------|
| Отзыв не создаётся | Проверьте, что статус заявки = 3 (Завершено) |
| Смена статуса не работает | Проверьте допустимые переходы (1→2, 2→3) |
| `405 Method Not Allowed` | Проверьте HTTP-метод (GET/POST/PUT/DELETE) |
| `403 Forbidden` | Проверьте роль пользователя и владельца заявки |
| `ImportError: review_bp` | Убедитесь, что `review_routes.py` создан |
| Оценка не передаётся | Проверьте `name="rating"` в HTML |

---

## Заключение

В ходе работы вы:

- ✅ Интегрировали фронтенд с бекендом через REST API
- ✅ Реализовали полный CRUD для заявок
- ✅ Реализовали смену статусов с валидацией переходов
- ✅ Реализовали модуль отзывов
- ✅ Протестировали все сценарии работы
- ✅ Зафиксировали результаты в Git

**Приложение «Конференции.РФ» готово к использованию** и полностью соответствует требованиям технического задания. Следующие практические работы могут быть посвящены оптимизации производительности, разработке адаптивного дизайна или развёртыванию на production-сервере.
# Задание для самостоятельной разработки фронтенда

## Введение

В предыдущих практических работах вы разработали **полноценный бекенд** на Flask + PostgreSQL с REST API. Однако фронтенд был минимальным: базовые HTML-страницы с минимальным CSS и простыми JavaScript-запросами через `fetch`.

**Задача:** самостоятельно разработать **современный, профессиональный фронтенд** для портала «Конференции.РФ».

---

## Что уже реализовано на бекенде

Вам НЕ нужно переписывать бекенд — он полностью готов. Ниже — список всех доступных эндпоинтов, с которыми будет работать ваш фронтенд.

### 1. Аутентификация (`/api/auth/`)

| Метод | URL | Тело запроса | Ответ |
|-------|-----|--------------|-------|
| POST | `/api/auth/register` | `{login, password, email, phone}` | `{success, user_id}` или `{success: false, message}` |
| POST | `/api/auth/login` | `{login, password}` | `{success, role, redirect}` |
| POST | `/api/auth/logout` | — | `{success: true}` |
| GET | `/api/auth/login` | — | HTML-страница (можно не использовать) |
| GET | `/api/auth/register` | — | HTML-страница (можно не использовать) |

### 2. Заявки (`/`)

| Метод | URL | Тело запроса | Ответ |
|-------|-----|--------------|-------|
| GET | `/api/requests` | — | `{success, requests: [...]}` — заявки текущего пользователя |
| POST | `/create_request` | `{event_id}` | `{success, request_id, message}` |
| DELETE | `/api/requests/<id>` | — | `{success}` — только для статуса «Новая» |
| GET | `/dashboard` | — | HTML (можно не использовать) |

### 3. Администратор (`/api/admin/`)

| Метод | URL | Тело запроса | Ответ |
|-------|-----|--------------|-------|
| GET | `/api/admin/requests` | — | `{success, requests: [...]}` — все заявки |
| PUT | `/api/admin/requests/<id>` | `{status_id}` | `{success}` |
| DELETE | `/api/admin/requests/<id>` | — | `{success}` |
| GET | `/admin` | — | HTML (можно не использовать) |

### 4. Отзывы (`/api/`)

| Метод | URL | Тело запроса | Ответ |
|-------|-----|--------------|-------|
| POST | `/api/review` | `{request_id, text, rating}` | `{success, review_id, message}` |
| GET | `/api/reviews` | — | `{success, reviews: [...]}` — только для админа |
| DELETE | `/api/reviews/<id>` | — | `{success}` — только для админа |
| GET | `/review/<id>` | — | HTML (можно не использовать) |

### 5. Мероприятия

Мероприятия уже наполнены в БД через `seed_events.py`. Для получения списка на странице создания заявки используйте шаблон `create_request.html` (или сделайте отдельный эндпоинт `GET /api/events` при необходимости).

---

## Что нужно разработать самостоятельно

### Общие требования

| Требование | Описание |
|------------|----------|
| **Современный дизайн** | Чистый, адаптивный интерфейс (mobile-first) |
| **Разделение файлов** | CSS, JS, шаблоны — по отдельным файлам |
| **SPA-подход** | Использовать `fetch` API без перезагрузки страниц |
| **Валидация** | На клиенте — мгновенная обратная связь |
| **Обработка ошибок** | Понятные сообщения пользователю |
| **Адаптивность** | Работа на экранах 390×844 (смартфон) и выше |
| **Анимации** | Плавные переходы, микроанимации |

---

## Задание 1. Стилизация страницы регистрации

### Что нужно сделать

1. Создайте красивую форму регистрации с полями:
   - **Логин** — с подсказкой «латиница и цифры, от 6 символов».
   - **Пароль** — с кнопкой «показать/скрыть» и индикатором надёжности.
   - **Email** — с валидацией в реальном времени.
   - **Телефон** — с автоформатированием по маске `8(XXX)XXX-XX-XX`.
2. Показывайте ошибки валидации **под каждым полем**, а не одним сообщением.
3. Кнопка **«Создать пользователя»** должна быть неактивна, пока форма не валидна.
4. При успехе — плавный переход на страницу входа.

### Пример структуры

```
frontend/
├── static/
│   ├── css/
│   │   ├── base.css        ← общие стили (reset, шрифты, кнопки)
│   │   ├── auth.css        ← стили для регистрации/входа
│   │   └── dashboard.css   ← стили для личного кабинета
│   └── js/
│       ├── api.js          ← обёртка над fetch
│       ├── validators.js   ← функции валидации
│       ├── register.js     ← логика регистрации
│       └── ...
```

### Пример обёртки для API (`api.js`)

```javascript
class ApiClient {
    static async request(url, options = {}) {
        const defaultOptions = {
            headers: { 'Content-Type': 'application/json' }
        };
        const response = await fetch(url, { ...defaultOptions, ...options });
        if (!response.ok) {
            const error = await response.json();
            throw new Error(error.message || 'Ошибка запроса');
        }
        return response.json();
    }

    static get(url) {
        return this.request(url);
    }

    static post(url, data) {
        return this.request(url, { method: 'POST', body: JSON.stringify(data) });
    }

    static put(url, data) {
        return this.request(url, { method: 'PUT', body: JSON.stringify(data) });
    }

    static delete(url) {
        return this.request(url, { method: 'DELETE' });
    }
}
```

### Пример автоформатирования телефона

```javascript
document.getElementById('phone').addEventListener('input', (e) => {
    let value = e.target.value.replace(/\D/g, ''); // только цифры
    if (value.startsWith('8')) {
        value = value.substring(0, 11);
        let formatted = '8';
        if (value.length > 1) formatted += '(' + value.substring(1, 4);
        if (value.length >= 4) formatted += ')' + value.substring(4, 7);
        if (value.length >= 7) formatted += '-' + value.substring(7, 9);
        if (value.length >= 9) formatted += '-' + value.substring(9, 11);
        e.target.value = formatted;
    }
});
```

---

## Задание 2. Стилизация страницы входа

### Что нужно сделать

1. Форма входа с полями **логин** и **пароль**.
2. Кнопка **«Войти»** — с индикатором загрузки (spinner).
3. Показывайте ошибки **«Неверный логин или пароль»** под формой.
4. Ссылка **«Ещё не зарегистрированы? Регистрация»**.
5. После успеха — редирект в зависимости от роли:
   - `user` → `/dashboard`
   - `admin` → `/admin`

---

## Задание 3. Личный кабинет пользователя (Dashboard)

### Что нужно сделать

1. **Карточки заявок** (вместо таблицы) — для лучшей адаптивности на смартфонах:
   - Номер заявки
   - Название мероприятия
   - Дата
   - Место
   - **Цветной бейдж статуса** (Новая — жёлтый, Назначено — синий, Завершено — зелёный)
   - Кнопки действий в зависимости от статуса
2. **Фильтрация** заявок по статусу:
   - Все
   - Новые
   - Назначенные
   - Завершённые
3. **Кнопка «+ Создать заявку»** в правом верхнем углу.
4. **Аватар пользователя** (заглушка) с выпадающим меню:
   - Профиль
   - Выйти
5. **Пустое состояние** — если заявок нет, покажите иллюстрацию и кнопку «Создать первую заявку».

### Пример структуры карточки

```html
<div class="card request-card" data-status="1">
    <div class="card-header">
        <span class="request-id">#42</span>
        <span class="status status-1">Новая</span>
    </div>
    <h3 class="card-title">Аудитория №101</h3>
    <div class="card-info">
        <p>📅 2027-06-15</p>
        <p>📍 Аудитория</p>
    </div>
    <div class="card-actions">
        <button class="btn btn-danger btn-sm">Удалить</button>
    </div>
</div>
```

---

## Задание 4. Форма создания заявки

### Что нужно сделать

1. **Выпадающий список помещений** — сгруппирован по типу:
   - Аудитории
   - Коворкинги
   - Кинозалы
2. **Предпросмотр выбранного помещения** — карточка справа/снизу.
3. **Кнопка «Отправить»** — с индикатором загрузки.
4. **Подтверждение** — модальное окно перед отправкой.
5. **Уведомление об успехе** (toast) — исчезает через 3 секунды.

### Пример группировки `<select>`

```html
<select id="event_id" required>
    <option value="">— Выберите помещение —</option>
    <optgroup label="Аудитории">
        <option value="1">Аудитория №101 (100 чел.)</option>
        <option value="4">Конференц-зал «Восток»</option>
    </optgroup>
    <optgroup label="Коворкинги">
        <option value="2">Коворкинг «Прогресс»</option>
        <option value="5">Коворкинг «Цифра»</option>
    </optgroup>
    <optgroup label="Кинозалы">
        <option value="3">Кинозал «Октябрь»</option>
    </optgroup>
</select>
```

---

## Задание 5. Панель администратора

### Что нужно сделать

1. **Дашборд со статистикой**:
   - Всего заявок
   - Новых
   - Назначенных
   - Завершённых
   - Средний рейтинг отзывов
2. **Таблица заявок** с фильтрами:
   - По статусу
   - По пользователю
   - По дате
3. **Смена статуса** через **выпадающий список** (не через select в таблице, а красиво оформленный):
   - Разрешить только допустимые переходы.
   - Заблокировать, если статус уже «Завершено».
4. **Удаление заявки** — с подтверждением.
5. **Таблица отзывов** с оценками в виде звёзд.
6. **Экспорт отзывов в CSV** (дополнительная задача).
7. **Адаптивность** — таблица должна превращаться в карточки на смартфоне.

### Пример бейджа со звездами

```html
<div class="rating-display">
    <span class="star filled">★</span>
    <span class="star filled">★</span>
    <span class="star filled">★</span>
    <span class="star filled">★</span>
    <span class="star empty">☆</span>
    <span class="rating-value">4.0</span>
</div>
```

---

## Задание 6. Форма отзыва

### Что нужно сделать

1. **Красивая форма** с карточкой мероприятия сверху.
2. **Оценка звёздами** с интерактивным выбором (при наведении подсвечиваются).
3. **Текст отзыва** с счётчиком символов (от 5 до 2000).
4. **Кнопка «Отправить»** — активна только при валидной форме.
5. **Если отзыв уже оставлен** — показать его в режиме чтения, без формы.

### Пример интерактивного рейтинга

```html
<div class="star-rating" id="starRating">
    <input type="radio" name="rating" value="5" id="star5">
    <label for="star5">★</label>
    <input type="radio" name="rating" value="4" id="star4">
    <label for="star4">★</label>
    <input type="radio" name="rating" value="3" id="star3">
    <label for="star3">★</label>
    <input type="radio" name="rating" value="2" id="star2">
    <label for="star2">★</label>
    <input type="radio" name="rating" value="1" id="star1">
    <label for="star1">★</label>
</div>
```

```css
.star-rating {
    display: flex;
    flex-direction: row-reverse;
    justify-content: flex-end;
}
.star-rating input { display: none; }
.star-rating label {
    font-size: 32px;
    color: #ddd;
    cursor: pointer;
    transition: color 0.2s;
}
.star-rating label:hover,
.star-rating label:hover ~ label,
.star-rating input:checked ~ label {
    color: #f39c12;
}
```

---

## Задание 7. Адаптивность (mobile-first)

### Что нужно сделать

1. **Медиазапросы** для трёх разрешений:
   - Мобильные: `max-width: 480px`
   - Планшеты: `481px — 1024px`
   - Десктоп: `1025px+`
2. **Мобильная навигация** — бургер-меню.
3. **Таблицы → карточки** на мобильных.
4. **Формы** — крупные поля, удобные для тапа.
5. **Модальные окна** — на весь экран на смартфоне.

### Пример медиазапроса

```css
/* Мобильные устройства по умолчанию */
.container { padding: 15px; }

/* Планшеты и десктоп */
@media (min-width: 768px) {
    .container { max-width: 720px; margin: 40px auto; }
}

@media (min-width: 1200px) {
    .container { max-width: 1140px; }
}
```

---

## Задание 8. Анимации и микроанимации

### Что нужно сделать

1. **Плавные переходы** между страницами.
2. **Анимация появления карточек** (fade-in + slide-up).
3. **Spinner** при загрузке данных.
4. **Toast-уведомления** в правом верхнем углу.
5. **Модальные окна** с плавным появлением.
6. **Пульсация** статуса «Новая».
7. **Hover-эффекты** на кнопках и карточках.

### Пример CSS-анимации

```css
@keyframes fadeInUp {
    from {
        opacity: 0;
        transform: translateY(20px);
    }
    to {
        opacity: 1;
        transform: translateY(0);
    }
}

.request-card {
    animation: fadeInUp 0.3s ease-out;
}

@keyframes pulse {
    0%, 100% { opacity: 1; }
    50% { opacity: 0.6; }
}

.status-1 {
    animation: pulse 2s infinite;
}

.spinner {
    display: inline-block;
    width: 20px;
    height: 20px;
    border: 3px solid rgba(255,255,255,0.3);
    border-radius: 50%;
    border-top-color: #fff;
    animation: spin 1s ease-in-out infinite;
}

@keyframes spin {
    to { transform: rotate(360deg); }
}
```

---

## Задание 9. Уведомления (Toast)

### Что нужно сделать

Создайте модуль `toast.js`:

```javascript
class Toast {
    static show(message, type = 'info', duration = 3000) {
        const toast = document.createElement('div');
        toast.className = `toast toast-${type}`;
        toast.textContent = message;
        document.body.appendChild(toast);

        setTimeout(() => toast.classList.add('show'), 10);
        setTimeout(() => {
            toast.classList.remove('show');
            setTimeout(() => toast.remove(), 300);
        }, duration);
    }

    static success(msg) { this.show(msg, 'success'); }
    static error(msg)   { this.show(msg, 'error'); }
    static info(msg)    { this.show(msg, 'info'); }
    static warning(msg) { this.show(msg, 'warning'); }
}
```

```css
.toast {
    position: fixed;
    top: 20px;
    right: 20px;
    padding: 15px 25px;
    border-radius: 8px;
    color: #fff;
    font-weight: bold;
    opacity: 0;
    transform: translateX(400px);
    transition: all 0.3s ease;
    z-index: 9999;
}
.toast.show {
    opacity: 1;
    transform: translateX(0);
}
.toast-success { background: #27ae60; }
.toast-error   { background: #e74c3c; }
.toast-info    { background: #3498db; }
.toast-warning { background: #f39c12; }
```

---

## Задание 10. Модальные окна

### Что нужно сделать

Создайте модуль `modal.js`:

```javascript
class Modal {
    static confirm(title, message) {
        return new Promise((resolve) => {
            const modal = document.createElement('div');
            modal.className = 'modal-overlay';
            modal.innerHTML = `
                <div class="modal">
                    <h3>${title}</h3>
                    <p>${message}</p>
                    <div class="modal-actions">
                        <button class="btn btn-secondary" data-action="cancel">Отмена</button>
                        <button class="btn btn-primary" data-action="confirm">Подтвердить</button>
                    </div>
                </div>
            `;
            document.body.appendChild(modal);
            modal.addEventListener('click', (e) => {
                const action = e.target.dataset.action;
                if (action) {
                    modal.remove();
                    resolve(action === 'confirm');
                }
            });
        });
    }
}
```

---

## Задание 11. Тёмная тема (дополнительно)

### Что нужно сделать

1. Кнопка переключения темы в правом верхнем углу.
2. Сохранение выбора в `localStorage`.
3. CSS-переменные для цветов.

```css
:root {
    --bg-primary: #ffffff;
    --text-primary: #2c3e50;
    --accent: #2980b9;
}

[data-theme="dark"] {
    --bg-primary: #1e1e1e;
    --text-primary: #ecf0f1;
    --accent: #3498db;
}

body {
    background: var(--bg-primary);
    color: var(--text-primary);
}
```

```javascript
const themeToggle = document.getElementById('themeToggle');
const currentTheme = localStorage.getItem('theme') || 'light';
document.body.dataset.theme = currentTheme;

themeToggle.addEventListener('click', () => {
    const newTheme = document.body.dataset.theme === 'dark' ? 'light' : 'dark';
    document.body.dataset.theme = newTheme;
    localStorage.setItem('theme', newTheme);
});
```

---

## Задание 12. Обработка ошибок и загрузки

### Что нужно сделать

1. **Индикатор загрузки** при каждом запросе к API.
2. **Skeleton-экраны** пока данные грузятся.
3. **Fallback-сообщения** при сетевых ошибках.
4. **Повторная попытка** (retry) при ошибке.

### Пример

```javascript
async function loadWithRetry(url, retries = 3) {
    for (let i = 0; i < retries; i++) {
        try {
            return await ApiClient.get(url);
        } catch (err) {
            if (i === retries - 1) throw err;
            await new Promise(r => setTimeout(r, 1000 * (i + 1)));
        }
    }
}
```

---

## Стек технологий: рекомендации

Вы можете использовать **чистый JavaScript** (как в текущем проекте) или подключить современные библиотеки:

| Технология | Для чего | Ссылка |
|------------|----------|--------|
| **Tailwind CSS** | Быстрая стилизация | https://tailwindcss.com |
| **Bootstrap 5** | Готовые компоненты | https://getbootstrap.com |
| **Alpine.js** | Реактивность без сборки | https://alpinejs.dev |
| **Vue 3 (CDN)** | Компонентный подход | https://vuejs.org |
| **Chart.js** | Графики на админ-панели | https://chartjs.org |
| **SweetAlert2** | Красивые модальные окна | https://sweetalert2.github.io |

> **Важно:** если используете библиотеки через CDN, помните, что интернет на рабочих местах **недоступен** (см. спецификацию КИМ). Поэтому **скачайте библиотеки заранее** и положите их в `frontend/static/vendor/`.

---

## Структура итогового фронтенда

```
frontend/
├── static/
│   ├── css/
│   │   ├── base.css           ← общие стили (reset, переменные)
│   │   ├── layout.css         ← сетка, header, footer
│   │   ├── components.css     ← кнопки, карточки, бейджи, формы
│   │   ├── auth.css           ← регистрация и вход
│   │   ├── dashboard.css      ← личный кабинет
│   │   ├── admin.css          ← админ-панель
│   │   ├── review.css         ← форма отзыва
│   │   ├── animations.css     ← анимации
│   │   └── responsive.css     ← медиазапросы
│   ├── js/
│   │   ├── api.js             ← обёртка над fetch
│   │   ├── validators.js      ← валидация на клиенте
│   │   ├── toast.js           ← уведомления
│   │   ├── modal.js           ← модальные окна
│   │   ├── theme.js           ← тёмная тема
│   │   ├── register.js
│   │   ├── login.js
│   │   ├── dashboard.js
│   │   ├── create_request.js
│   │   ├── admin.js
│   │   └── review.js
│   ├── img/
│   │   ├── logo.svg
│   │   ├── empty-state.svg
│   │   └── avatar.svg
│   └── vendor/                ← локальные библиотеки (если используете)
│       ├── alpine.min.js
│       └── chart.min.js
└── templates/
    ├── base.html              ← базовый шаблон с header/footer
    ├── register.html
    ├── login.html
    ├── dashboard.html
    ├── create_request.html
    ├── admin.html
    └── review.html
```

---

## Критерии оценивания фронтенда

| № | Критерий | Баллы |
|---|----------|-------|
| 1 | Современный дизайн (не «голый» HTML) | 2 |
| 2 | Адаптивность (mobile-first, 390×844) | 2 |
| 3 | Интеграция с бекендом через `fetch` | 2 |
| 4 | Клиентская валидация с сообщениями | 2 |
| 5 | Toast-уведомления | 1 |
| 6 | Модальные окна подтверждения | 1 |
| 7 | Анимации и переходы | 1 |
| 8 | Разделение CSS/JS по файлам | 1 |
| 9 | Обработка ошибок и загрузки | 1 |
| 10 | Тёмная тема (дополнительно) | +1 |
| **ИТОГО** | | **13 (+1)** |

---

## Полезные ссылки и ресурсы

| Ресурс | Ссылка |
|--------|--------|
| MDN: Fetch API | https://developer.mozilla.org/ru/docs/Web/API/Fetch_API |
| CSS-Tricks: Flexbox | https://css-tricks.com/snippets/css/a-guide-to-flexbox/ |
| CSS-Tricks: Grid | https://css-tricks.com/snippets/css/complete-guide-grid/ |
| Coolors (палитры) | https://coolors.co |
| Google Fonts | https://fonts.google.com |
| Heroicons | https://heroicons.com |
| Lucide Icons | https://lucide.dev |

---

## Порядок сдачи

1. Все файлы фронтенда сохранены в репозиторий.
2. Сделано **минимум 5 коммитов** (по одному на каждое задание).
3. В `README.md` добавлено описание фронтенда с примерами скриншотов.
4. Приложение работает на `http://127.0.0.1:5000`.
5. Все эндпоинты бекенда корректно используются.

---

## Итоговая проверка

| Сценарий | Ожидаемый результат |
|----------|---------------------|
| Открытие главной | Красивая страница входа |
| Регистрация | Мгновенная валидация, ошибки под полями |
| Вход участника | Карточки заявок, фильтры |
| Создание заявки | Модальное подтверждение, toast |
| Вход администратора | Дашборд со статистикой |
| Смена статуса | Плавное обновление, toast |
| Оставление отзыва | Интерактивные звёзды, счётчик символов |
| Мобильная версия | Бургер-меню, карточки вместо таблиц |
| Тёмная тема | Переключение, сохранение в localStorage |

---

## Заключение

Бекенд полностью готов и предоставляет **REST API**. Ваша задача — разработать **современный, адаптивный, анимированный фронтенд**, который:

- Красиво отображает все данные.
- Даёт мгновенную обратную связь пользователю.
- Корректно обрабатывает ошибки.
- Работает на смартфонах и десктопах.

**Проявите творческий подход** — этот проект станет вашим портфолио. Используйте современные техники CSS (Grid, Flexbox, переменные), JavaScript (fetch, async/await, классы) и UX-принципы (мгновенная валидация, toast, skeleton-экраны).

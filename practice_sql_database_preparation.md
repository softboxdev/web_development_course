# Практическая работа № 2. Подробная инструкция: создание базы данных SQLite в ОС Linux Rosa

## Введение

В этой инструкции пошагово описан процесс создания базы данных SQLite в операционной системе **РОСА Линукс**. Вы научитесь устанавливать SQLite, создавать базу данных, проектировать таблицы с учётом связей и проверять результат.

**SQLite** — это компактная встраиваемая реляционная база данных, которая не требует отдельного сервера. Вся база хранится в одном файле на диске (обычно с расширением `.db` или `.sqlite`). Это делает её идеальной для учебных проектов и небольших приложений.

---

## Шаг 1. Проверка наличия SQLite в системе

В РОСА Линукс SQLite обычно уже присутствует в дистрибутиве или доступен в репозиториях.

Откройте терминал и выполните команду:

```bash
sqlite3 --version
```

**Возможные результаты:**

| Результат | Что делать |
|-----------|------------|
| Показана версия (например, `3.40.1`) | SQLite установлен, переходите к шагу 3 |
| Команда не найдена (`command not found`) | Переходите к шагу 2 |

---

## Шаг 2. Установка SQLite

В РОСА Линукс для управления пакетами используется менеджер **dnf**.

Выполните в терминале:

```bash
sudo dnf install sqlite3
```

После ввода пароля администратора система скачает и установит SQLite.

**Проверка установки:**

```bash
sqlite3 --version
```

Убедитесь, что версия отображается.

---

## Шаг 3. Создание базы данных

SQLite не требует отдельной команды «создать базу». База создаётся автоматически при первом подключении к файлу, которого ещё не существует.

**Синтаксис:**

```bash
sqlite3 имя_базы.db
```

**Пример для портала «Конференции.РФ»:**

```bash
sqlite3 conference.db
```

**Что произойдёт:**
- Если файл `conference.db` не существует — он будет создан.
- Если файл существует — он откроется для работы.
- Вы увидите приглашение командной строки SQLite: `sqlite>`

> **Важно:** База создаётся в текущей директории. Чтобы создать её в конкретной папке, укажите полный путь: `sqlite3 /home/user/project/conference.db`

---

## Шаг 4. Создание таблиц

Теперь, находясь в приглашении `sqlite>`, создайте таблицы согласно спроектированной ER-диаграмме.

### 4.1. Таблица «Пользователь» (user)

```sql
CREATE TABLE user (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    login VARCHAR(50) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    phone VARCHAR(18) NOT NULL,
    role VARCHAR(20) NOT NULL DEFAULT 'user'
);
```

**Пояснения:**
- `INTEGER PRIMARY KEY AUTOINCREMENT` — автоматически увеличивающийся уникальный идентификатор.
- `VARCHAR(50) UNIQUE` — строка до 50 символов, значения уникальны.
- `NOT NULL` — поле обязательно для заполнения.
- `DEFAULT 'user'` — значение по умолчанию, если не указано иное.

### 4.2. Таблица «Мероприятие» (event)

```sql
CREATE TABLE event (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name VARCHAR(200) NOT NULL,
    date DATE NOT NULL,
    place VARCHAR(200) NOT NULL,
    description TEXT
);
```

### 4.3. Таблица «Статус» (status)

```sql
CREATE TABLE status (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name VARCHAR(50) UNIQUE NOT NULL
);
```

**Наполнение справочника статусов:**

```sql
INSERT INTO status (name) VALUES 
    ('Новая'),
    ('Мероприятие назначено'),
    ('Завершено');
```

### 4.4. Таблица «Заявка» (request) — с внешними ключами

```sql
CREATE TABLE request (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id INTEGER NOT NULL,
    event_id INTEGER NOT NULL,
    status_id INTEGER NOT NULL DEFAULT 1,
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES user(id),
    FOREIGN KEY (event_id) REFERENCES event(id),
    FOREIGN KEY (status_id) REFERENCES status(id)
);
```

**Пояснения по связям:**
- `FOREIGN KEY (user_id) REFERENCES user(id)` — внешний ключ, ссылающийся на таблицу `user`.
- Тип поля внешнего ключа должен совпадать с типом первичного ключа, на который он ссылается.
- `DEFAULT CURRENT_TIMESTAMP` — автоматическая подстановка текущей даты и времени.

### 4.5. Таблица «Отзыв» (review)

```sql
CREATE TABLE review (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    request_id INTEGER UNIQUE NOT NULL,
    text TEXT NOT NULL,
    rating INTEGER CHECK (rating BETWEEN 1 AND 5),
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (request_id) REFERENCES request(id)
);
```

**Пояснения:**
- `UNIQUE` на `request_id` — одна заявка может иметь только один отзыв (связь 1:1).
- `CHECK (rating BETWEEN 1 AND 5)` — оценка может быть только от 1 до 5.

---

## Шаг 5. Включение поддержки внешних ключей

**Важный момент:** В SQLite внешние ключи **отключены по умолчанию**. Чтобы они работали, нужно выполнить:

```sql
PRAGMA foreign_keys = ON;
```

Это необходимо делать **при каждом подключении** к базе данных. Можно добавить эту команду в начало SQL-скрипта.

---

## Шаг 6. Сохранение SQL-скрипта в файл

Чтобы не вводить все команды вручную каждый раз, сохраните их в файл.

### 6.1. Создание файла скрипта

Откройте текстовый редактор (например, `nano`):

```bash
nano schema.sql
```

### 6.2. Содержимое файла `schema.sql`

```sql
-- Включаем поддержку внешних ключей
PRAGMA foreign_keys = ON;

-- Таблица «Пользователь»
CREATE TABLE IF NOT EXISTS user (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    login VARCHAR(50) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    phone VARCHAR(18) NOT NULL,
    role VARCHAR(20) NOT NULL DEFAULT 'user'
);

-- Таблица «Мероприятие»
CREATE TABLE IF NOT EXISTS event (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name VARCHAR(200) NOT NULL,
    date DATE NOT NULL,
    place VARCHAR(200) NOT NULL,
    description TEXT
);

-- Таблица «Статус»
CREATE TABLE IF NOT EXISTS status (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name VARCHAR(50) UNIQUE NOT NULL
);

-- Наполнение справочника статусов
INSERT OR IGNORE INTO status (name) VALUES 
    ('Новая'),
    ('Мероприятие назначено'),
    ('Завершено');

-- Таблица «Заявка»
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

-- Таблица «Отзыв»
CREATE TABLE IF NOT EXISTS review (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    request_id INTEGER UNIQUE NOT NULL,
    text TEXT NOT NULL,
    rating INTEGER CHECK (rating BETWEEN 1 AND 5),
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (request_id) REFERENCES request(id)
);
```

**Пояснения:**
- `CREATE TABLE IF NOT EXISTS` — создаёт таблицу, только если её ещё нет (защита от повторного запуска).
- `INSERT OR IGNORE` — добавляет запись, игнорируя ошибку дублирования.

### 6.3. Выход из редактора

Нажмите `Ctrl+O`, затем `Enter` для сохранения. Затем `Ctrl+X` для выхода.

### 6.4. Применение скрипта

```bash
sqlite3 conference.db < schema.sql
```

Или изнутри SQLite:

```sql
.read schema.sql
```

---

## Шаг 7. Проверка созданных таблиц

### 7.1. Просмотр списка таблиц

```bash
sqlite3 conference.db ".tables"
```

**Ожидаемый вывод:**
```
event  request  review  status  user
```

### 7.2. Просмотр структуры таблицы

```sql
.schema user
```

**Или через SQL:**

```sql
PRAGMA table_info('user');
```

Эта команда покажет все колонки, их типы и ограничения.

### 7.3. Просмотр внешних ключей

```sql
PRAGMA foreign_key_list('request');
```

Покажет, какие внешние ключи определены в таблице `request`.

---

## Шаг 8. Вставка тестовых данных

Проверьте работоспособность базы, добавив тестовые данные.

### 8.1. Добавление пользователя

```sql
INSERT INTO user (login, password_hash, email, phone, role)
VALUES ('student2027', 'hash_example_123', 'student@example.ru', '8(999)123-45-67', 'user');
```

### 8.2. Добавление мероприятия

```sql
INSERT INTO event (name, date, place, description)
VALUES ('Аудитория №101', '2027-06-15', 'Аудитория', 'Большая аудитория для конференций');
```

### 8.3. Добавление заявки

```sql
INSERT INTO request (user_id, event_id, status_id)
VALUES (1, 1, 1);
```

### 8.4. Проверка данных

```sql
SELECT * FROM user;
SELECT * FROM event;
SELECT * FROM request;
```

**Ожидаемый вывод для `request`:**

```
id | user_id | event_id | status_id | created_at
1  | 1       | 1        | 1         | 2027-09-29 12:00:00
```

---

## Шаг 9. Полезные команды SQLite

### Команды приглашения SQLite (начинаются с точки)

| Команда | Назначение |
|---------|-----------|
| `.tables` | Показать все таблицы |
| `.schema` | Показать все SQL-скрипты создания |
| `.schema имя_таблицы` | Показать структуру конкретной таблицы |
| `.headers on` | Включить отображение заголовков колонок |
| `.mode column` | Красивый табличный вывод |
| `.mode line` | Вывод в виде «поле = значение» |
| `.exit` или `.quit` | Выход из SQLite |

**Пример настройки вывода:**

```sql
.headers on
.mode column
SELECT * FROM user;
```

---

## Шаг 10. Открытие базы для дальнейшей работы

Если вы вышли из SQLite, для повторного открытия базы выполните:

```bash
sqlite3 conference.db
```

База сохраняется в файле `conference.db` в текущей директории. Вы можете скопировать этот файл, передать его другому разработчику или загрузить в Git-репозиторий.

---

## Шаг 11. Построение ER-диаграммы для SQLite

### Вариант 1: Использование инструмента DbSchema

**DbSchema** — бесплатный инструмент для визуализации SQLite-баз (Community Edition).

**Пошаговая инструкция:**

1. Скачайте DbSchema: https://dbschema.com/download.html
2. Откройте программу → **Connect to Database**
3. Выберите **SQLite** из списка
4. Укажите путь к файлу `conference.db`
5. Нажмите **Connect**
6. DbSchema автоматически прочитает схему и нарисует ER-диаграмму

**Что вы увидите:**
- Таблицы в виде прямоугольников с полями
- Линии связей между таблицами (на основе FOREIGN KEY)
- Первичные и внешние ключи

### Вариант 2: Онлайн-инструменты

| Инструмент | Ссылка | Особенности |
|-----------|--------|-------------|
| **sqlite-erd** | npm-библиотека | React-компонент, локальная обработка |
| **dbdiagram.io** | https://dbdiagram.io | Ручное построение диаграмм |
| **draw.io** | https://app.diagrams.net | Универсальный редактор |

### Вариант 3: Через PRAGMA-команды (для проверки схемы)

```sql
-- Проверка структуры
PRAGMA table_info('user');
PRAGMA table_info('request');
PRAGMA foreign_key_list('request');
```

---

## Шаг 12. Сохранение в Git-репозиторий

Согласно требованиям практической работы, все результаты необходимо сохранить в репозиторий.

```bash
# Инициализация репозитория (если ещё не создан)
git init

# Добавление файлов
git add conference.db
git add schema.sql
git add er_diagram.png

# Коммит
git commit -m "Создана база данных SQLite для портала Конференции.РФ"

# Отправка на удалённый репозиторий
git push origin main
```

---

## Возможные проблемы и их решение

### Проблема 1: `sqlite3: command not found`

**Решение:** Установите SQLite через dnf:
```bash
sudo dnf install sqlite3
```

### Проблема 2: Ошибка `FOREIGN KEY constraint failed`

**Причина:** Внешние ключи отключены или данные нарушают целостность.

**Решение:**
```sql
PRAGMA foreign_keys = ON;
```
Убедитесь, что родительская запись (например, пользователь) существует перед созданием заявки.

### Проблема 3: Таблица уже существует

**Решение:** Используйте `DROP TABLE IF EXISTS` перед созданием:

```sql
DROP TABLE IF EXISTS request;
DROP TABLE IF EXISTS review;
DROP TABLE IF EXISTS status;
DROP TABLE IF EXISTS event;
DROP TABLE IF EXISTS user;
```

Или используйте `CREATE TABLE IF NOT EXISTS`.

### Проблема 4: Не отображаются внешние ключи в ER-диаграмме

**Причина:** SQLite не всегда сохраняет информацию о внешних ключах, если они не были объявлены явно в `CREATE TABLE`.

**Решение:** Убедитесь, что в скрипте присутствуют строки `FOREIGN KEY ... REFERENCES ...`.

---

## Итоговый чек-лист

| № | Действие | Статус |
|---|----------|--------|
| 1 | SQLite установлен | ☐ |
| 2 | База `conference.db` создана | ☐ |
| 3 | Все 5 таблиц созданы | ☐ |
| 4 | Внешние ключи определены | ☐ |
| 5 | Внешние ключи включены (`PRAGMA foreign_keys = ON`) | ☐ |
| 6 | Тестовые данные добавлены | ☐ |
| 7 | ER-диаграмма построена | ☐ |
| 8 | Результаты сохранены в Git | ☐ |

---

## Заключение

Вы научились:
- Устанавливать SQLite в РОСА Линукс через `dnf`;
- Создавать базу данных через командную строку;
- Проектировать таблицы с первичными и внешними ключами;
- Включать поддержку внешних ключей;
- Проверять структуру через `PRAGMA`;
- Строить ER-диаграмму.


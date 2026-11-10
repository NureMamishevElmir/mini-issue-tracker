# Mini Issue Tracker — API-контракт (ЛР1)

Базова адреса локально: `http://127.0.0.1:8000`. Формат — JSON, UTF-8. Поля, межі та бізнес-правила визначено в
[requirements.md](requirements.md); тут для кожної операції вказано лише вхід, вихід, коди та посилання на правила.

## Загальні угоди

- **Об'єкт `Issue`:** `{"id", "title", "description", "status", "priority", "assignee_id", "created_at"}` — розділ 6.
- **Об'єкт `Comment`:** `{"id", "issue_id", "author_id", "text", "created_at"}`.
- **Об'єкт `User`:** `{"id", "name"}`.
- **Час:** ISO 8601 UTC, `2026-11-10T09:15:30Z` (D-09). **Порядок списків:** за `id` зростанням (BR-04).
- **Порядок перевірок:** `422` → `404` → `409` (D-01). Будь-яка помилка не змінює дані (NFR-06).
- **Формат помилок:**
  - `422`: `{"detail": [{"loc": ["body", "title"], "msg": "...", "type": "..."}]}` — `loc` вказує поле (NFR-02);
  - `404`: `{"detail": "Issue 999 not found"}`;
  - `409`: `{"detail": "Transition CLOSED -> OPEN is not allowed"}` або `{"detail": "Issue 7 is CLOSED and cannot be edited"}`.
- `{id}` у шляху — додатне ціле; нечислове або ≤ 0 → `422`.

## Зведення маршрутів

| # | Метод і шлях | Призначення | Успіх | Помилки | FR |
|---|--------------|-------------|-------|---------|----|
| 1 | `GET /health` | Технічна перевірка запуску | `200` | — | NFR-01 |
| 2 | `GET /users` | Два тестові користувачі | `200` | — | FR-05, FR-06 |
| 3 | `POST /issues` | Створити заявку | `201` | `422` | FR-01 |
| 4 | `GET /issues` | Список із фільтрами `status`, `priority` (AND) | `200` | `422` | FR-02 |
| 5 | `GET /issues/{id}` | Одна заявка | `200` | `404`, `422` | FR-02 |
| 6 | `PATCH /issues/{id}` | Часткова зміна `title`, `description`, `priority`, `assignee_id` | `200` | `404`, `409`, `422` | FR-03, FR-05 |
| 7 | `PATCH /issues/{id}/status` | Зміна статусу | `200` | `404`, `409`, `422` | FR-04 |
| 8 | `POST /issues/{id}/comments` | Додати коментар | `201` | `404`, `422` | FR-06 |
| 9 | `GET /issues/{id}/comments` | Коментарі заявки | `200` | `404`, `422` | FR-06 |

---

### 1. `GET /health`

- **Вхід:** немає.
- **Успіх `200`:** `{"status": "ok"}`.

### 2. `GET /users`

- **Вхід:** немає.
- **Успіх `200`:** масив `User` у порядку `id`.

```json
[{"id": 1, "name": "Agent A"}, {"id": 2, "name": "Agent B"}]
```

### 3. `POST /issues`

- **Body:** `title` (обов'язкове), `description` (необов'язкове), `priority` (необов'язкове). Інші поля, зокрема
  `status`, `assignee_id`, `id`, `created_at`, → `422` (D-03, D-07).
- **Успіх `201`:** створений `Issue` зі `status = "OPEN"`, `assignee_id = null`, `description = ""` і
  `priority = "medium"`, якщо їх не передано.
- **Помилки:** `422` — назва порожня, коротша за 3 або довша за 120 символів після обрізання; опис > 2000 або `null`;
  недопустимий `priority`; невідоме поле.
- **Правила:** розділ 6, D-03, D-05–D-07, D-11. **Критерії:** AC-01-01 – AC-01-04.

Приклад успіху:

```http
POST /issues
{"title": "  Login fails  ", "description": "Crash after login"}
```
```http
201 Created
{"id": 1, "title": "Login fails", "description": "Crash after login", "status": "OPEN",
 "priority": "medium", "assignee_id": null, "created_at": "2026-11-10T09:15:30Z"}
```

Приклад помилки:

```http
POST /issues
{"title": " ab ", "priority": "High"}
```
```http
422 Unprocessable Content
{"detail": [
  {"loc": ["body", "title"], "msg": "Title must be 3-120 characters after trimming", "type": "string_length"},
  {"loc": ["body", "priority"], "msg": "Input should be 'low', 'medium' or 'high'", "type": "enum"}
]}
```

### 4. `GET /issues`

- **Query:** `status` (необов'язковий, `OPEN` | `IN_PROGRESS` | `CLOSED`), `priority` (необов'язковий,
  `low` | `medium` | `high`). Обидва разом — AND.
- **Успіх `200`:** масив `Issue` за зростанням `id`; нічого не знайдено → `[]`.
- **Помилки:** `422` — недопустиме значення фільтра (наприклад, `status=open`).
- **Правила:** BR-04. **Критерії:** AC-02-01 – AC-02-03, AC-02-05.

```http
GET /issues?status=OPEN&priority=high
200 OK
[{"id": 1, "title": "Login fails", "description": "Crash after login", "status": "OPEN",
  "priority": "high", "assignee_id": 2, "created_at": "2026-11-10T09:15:30Z"}]
```

### 5. `GET /issues/{id}`

- **Path:** `id`.
- **Успіх `200`:** `Issue`.
- **Помилки:** `404` — заявки немає; `422` — некоректний `id`.
- **Критерії:** AC-02-04.

### 6. `PATCH /issues/{id}`

- **Path:** `id`. **Body:** будь-яка непорожня підмножина `title`, `description`, `priority`, `assignee_id`.
- **Семантика:** BR-03 — змінюються лише передані поля; `assignee_id: null` знімає призначення; невдале оновлення
  атомарне.
- **Успіх `200`:** оновлений `Issue`.
- **Помилки:** `422` — порожнє тіло, невідоме поле (зокрема `status`), `null` у `title`/`description`/`priority`,
  порушені межі, `assignee_id` не `1`/`2`/`null`; `404` — заявки немає; `409` — заявка `CLOSED` (BR-02, D-04).
- **Критерії:** AC-03-01 – AC-03-04, AC-05-02, AC-05-03.

```http
PATCH /issues/1
{"priority": "high", "assignee_id": 2}
200 OK
{"id": 1, "title": "Login fails", "description": "Crash after login", "status": "OPEN",
 "priority": "high", "assignee_id": 2, "created_at": "2026-11-10T09:15:30Z"}
```

### 7. `PATCH /issues/{id}/status`

- **Path:** `id`. **Body:** `{"status": "<OPEN|IN_PROGRESS|CLOSED>"}`, інших полів немає.
- **Семантика:** матриця BR-01; той самий статус → `200` без змін.
- **Успіх `200`:** `Issue` з новим (або тим самим) статусом.
- **Помилки:** `422` — немає `status`, значення поза переліком, зайве поле; `404` — заявки немає; `409` — заборонений
  перехід.
- **Критерії:** AC-04-01 – AC-04-04.

```http
PATCH /issues/1/status
{"status": "IN_PROGRESS"}
200 OK
{"id": 1, "title": "Login fails", "description": "Crash after login", "status": "IN_PROGRESS",
 "priority": "high", "assignee_id": 2, "created_at": "2026-11-10T09:15:30Z"}
```

Приклад помилки (заявка вже `CLOSED`):

```http
PATCH /issues/7/status
{"status": "OPEN"}
409 Conflict
{"detail": "Transition CLOSED -> OPEN is not allowed"}
```

### 8. `POST /issues/{id}/comments`

- **Path:** `id` заявки. **Body:** `author_id` (`1` | `2`), `text` (1–1000 символів після обрізання).
- **Успіх `201`:** створений `Comment`. Дозволено й для заявки `CLOSED` (BR-02).
- **Помилки:** `422` — порожній або задовгий текст, невідомий автор, зайве поле; `404` — заявки немає.
- **Правила:** розділ 6, D-08. **Критерії:** AC-06-01 – AC-06-04.

```http
POST /issues/1/comments
{"author_id": 1, "text": "Reproduced on Chrome"}
201 Created
{"id": 1, "issue_id": 1, "author_id": 1, "text": "Reproduced on Chrome", "created_at": "2026-11-10T09:20:02Z"}
```

Приклад помилки:

```http
POST /issues/1/comments
{"author_id": 5, "text": "   "}
422 Unprocessable Content
{"detail": [
  {"loc": ["body", "author_id"], "msg": "Unknown user; allowed: 1, 2", "type": "value_error"},
  {"loc": ["body", "text"], "msg": "Text must be 1-1000 characters after trimming", "type": "string_length"}
]}
```

### 9. `GET /issues/{id}/comments`

- **Path:** `id` заявки.
- **Успіх `200`:** масив `Comment` за зростанням `id`; коментарів немає → `[]`.
- **Помилки:** `404` — заявки немає; `422` — некоректний `id`.
- **Критерії:** AC-06-01, AC-06-03.

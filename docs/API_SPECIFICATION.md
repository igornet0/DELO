# ДЕЛО — Спецификация HTTP API

> Статус документа: **Этап 0, проект контракта.** Ни один endpoint ещё не реализован.
> Колонка «Этап» указывает, на каком этапе [DEVELOPMENT_ROADMAP.md](DEVELOPMENT_ROADMAP.md) endpoint появляется. До этого этапа сервер возвращает `404`; заглушек, имитирующих работу, не будет.
> Источник правды после начала реализации — OpenAPI, генерируемый из кода (`utoipa`, **[D-11]**). Этот документ фиксирует соглашения и состав API до написания кода; расхождения разрешаются в пользу OpenAPI с обновлением этого файла.

Связанные документы: [ARCHITECTURE.md](ARCHITECTURE.md), [DATABASE_SCHEMA.md](DATABASE_SCHEMA.md), [DECISIONS.md](DECISIONS.md).

---

## 1. Общие соглашения

### 1.1. Базовый путь и версионирование

- Все пути начинаются с `/api/v1`. Несовместимые изменения — только в `/api/v2`; в пределах версии допускается лишь добавление полей и endpoint'ов.
- Ресурсы компании: `/api/v1/companies/{company_id}/...`. `company_id` из пути — **единственный** источник контекста компании. Middleware проверяет членство пользователя в компании и открывает транзакцию с `set_config('app.company_id', …)` (RLS). Если пользователь не состоит в компании, ответ — `404` (а не `403`), чтобы не раскрывать существование компании.
- Идентификаторы — UUID v7 (строка). Время — RFC 3339 в UTC (`2026-11-01T12:00:00Z`).
- Формат — `application/json; charset=utf-8`, поля в `snake_case`. Неизвестные поля во входящих DTO отклоняются (`deny_unknown_fields`), что защищает от mass assignment.

### 1.2. Аутентификация

| Способ | Для кого | Как передаётся |
|---|---|---|
| Сессия | Веб-интерфейс | cookie `delo_session` (`HttpOnly; Secure; SameSite=Lax; Path=/`) |
| API-токен | Интеграции, CLI | `Authorization: Bearer delo_<prefix>_<secret>` |

- **CSRF:** при входе сервер выставляет cookie `delo_csrf`, доступную JS. Каждый мутирующий запрос (`POST/PUT/PATCH/DELETE`) с сессионной аутентификацией обязан повторить её значение в заголовке `X-Delo-CSRF`. Без совпадения — `403 csrf_failed`. К запросам с Bearer-токеном CSRF-проверка не применяется.
- API-токен привязан к пользователю и (опционально) к одной компании, имеет scopes — подмножество разрешений пользователя. Итоговые права = права пользователя ∩ scopes токена.

### 1.3. Ошибки — RFC 7807 (`application/problem+json`)

```json
{
  "type": "https://delo.dev/problems/validation_failed",
  "title": "Некорректные данные",
  "status": 422,
  "code": "validation_failed",
  "detail": "Поле due_at не может быть в прошлом",
  "request_id": "01JC…",
  "errors": [{ "field": "due_at", "code": "in_past", "message": "Срок не может быть в прошлом" }]
}
```

| HTTP | `code` | Когда |
|---|---|---|
| 400 | `bad_request` | Синтаксически неверный запрос |
| 401 | `unauthenticated` | Нет сессии или токена, либо они истекли |
| 403 | `forbidden`, `csrf_failed` | Нет разрешения, или не пройдена CSRF-проверка |
| 404 | `not_found` | Ресурс отсутствует или недоступен в этой компании |
| 409 | `conflict`, `invalid_transition`, `version_mismatch` | Конфликт состояния, недопустимый переход статуса, устаревшая версия |
| 409 | `idempotency_key_reused` | Тот же ключ, но другое тело запроса |
| 413 | `payload_too_large` | Превышен размер тела или файла |
| 422 | `validation_failed` | Ошибка валидации DTO или доменного правила |
| 423 | `agents_paused` | Компания поставила агентов на паузу («стоп-кран») |
| 429 | `rate_limited` | Превышен лимит; заголовок `Retry-After` |
| 500 | `internal` | Внутренняя ошибка; детали только в логах (по `request_id`) |
| 503 | `unavailable` | Зависимость недоступна (БД, хранилище) |

Сообщения `title` и `message` — на языке из `Accept-Language` (по умолчанию `ru`). Frontend опирается на `code`, а не на текст.

### 1.4. Пагинация, сортировка, фильтры

- Курсорная пагинация: `?limit=50&cursor=<opaque>`; `limit` ≤ 200, по умолчанию 50.
- Ответ: `{ "items": [...], "next_cursor": "…" | null }`. Общее число записей не возвращается: `COUNT(*)` на больших таблицах дорог. Где счётчик нужен для UI, есть отдельный endpoint `/…/counts`.
- Сортировка — `?sort=created_at` или `?sort=-created_at`, только по полям из белого списка endpoint'а.
- Фильтры — именованные параметры (`?status=in_progress&assignee_type=agent`). Свободного языка запросов нет.

### 1.5. Идемпотентность

- Заголовок `Idempotency-Key` (UUID, 1–128 символов) принимается всеми мутирующими `POST`.
- Для endpoint'ов с пометкой **[IK]** заголовок **обязателен**: финансовые операции, решения по согласованиям, запуск агента. Без него ответ — `400`.
- Сервер сохраняет хеш тела и ответ в `idempotency_keys` (область: пользователь + компания + ключ, TTL 24 часа):
  - повтор с тем же телом возвращает сохранённый ответ и заголовок `Idempotent-Replayed: true`;
  - повтор с другим телом — `409 idempotency_key_reused`;
  - повтор, пока первый запрос ещё выполняется, — `409 conflict` с `Retry-After`.

### 1.6. Оптимистическая блокировка

Изменяемые ресурсы содержат поле `version` (целое). `PATCH`/`PUT` принимают заголовок `If-Match: "<version>"`. Если версия не совпадает — `409 version_mismatch`. Заголовок обязателен для оргструктуры, конфигурации агентов, политик инструментов и бюджетов.

### 1.7. Rate limiting

| Область | Лимит (начальные значения, настраиваются) |
|---|---|
| `/auth/login`, `/auth/password/*` | 5 в минуту на IP + email, 20 в час на IP |
| Аутентифицированный пользователь | 600 в минуту |
| API-токен | 1200 в минуту |
| Запуск агентов вручную | 30 в минуту на компанию |
| Загрузка файлов | 60 в минуту, 100 МБ на файл (настраивается) |

Ответы содержат заголовки `RateLimit-Limit`, `RateLimit-Remaining`, `RateLimit-Reset`. В MVP лимитер работает внутри процесса; с несколькими экземплярами API он переносится в Redis (**[D-06]**).

### 1.8. Акторы в ответах

Везде, где важно «кто сделал», возвращается объект актора, а не голый `user_id`:

```json
{ "type": "agent", "id": "…", "name": "Финансовый директор", "run_id": "…" }
```

Значения `type`: `user`, `agent`, `system`, `integration`. Поле `run_id` присутствует только у агента.

### 1.9. Деньги

Деньги передаются объектом `{ "amount_minor": 1250000, "currency": "RUB" }`: сумма — целое число в минимальных единицах, валюта — ISO 4217. Числа с плавающей точкой не используются.

---

## 2. Служебные endpoint'ы

| Метод | Путь | Описание | Этап |
|---|---|---|---|
| GET | `/health/live` | Процесс жив | 1 |
| GET | `/health/ready` | Готовность: БД, хранилище, миграции применены | 1 |
| GET | `/api/v1/openapi.json` | Сгенерированная спецификация (в production — только при флаге) | 1 |
| GET | `/metrics` | Метрики Prometheus; отдельный порт, наружу не публикуется | 1 |

---

## 3. Endpoint'ы по модулям

Обозначения в таблицах:
- «Разрешение» — требуемое разрешение из §4. Знак «—» означает, что достаточно аутентификации.
- **[IK]** — обязателен `Idempotency-Key`.
- Префикс `…` означает `/api/v1/companies/{company_id}`.

### 3.1. Аутентификация и профиль (`delo-iam`)

| Метод | Путь | Разрешение | Описание | Этап |
|---|---|---|---|---|
| POST | `/api/v1/auth/register` | публичный | Регистрация (может быть выключена флагом, тогда вход только по приглашению) | 1 |
| POST | `/api/v1/auth/login` | публичный | Вход по email и паролю → cookie сессии и CSRF | 1 |
| POST | `/api/v1/auth/logout` | — | Отзыв текущей сессии | 1 |
| POST | `/api/v1/auth/password/forgot` | публичный | Запрос сброса пароля. Ответ всегда `202` и не раскрывает, существует ли email | 1 |
| POST | `/api/v1/auth/password/reset` | публичный | Сброс по одноразовому токену | 1 |
| POST | `/api/v1/auth/email/verify` | публичный | Подтверждение email по токену | 1 |
| GET | `/api/v1/me` | — | Профиль, список компаний с ролями | 1 |
| PATCH | `/api/v1/me` | — | Имя, язык, часовой пояс | 1 |
| POST | `/api/v1/me/password` | — | Смена пароля (с текущим паролем); отзывает остальные сессии | 1 |
| GET | `/api/v1/me/sessions` | — | Активные сессии | 1 |
| DELETE | `/api/v1/me/sessions/{session_id}` | — | Отзыв сессии | 1 |
| GET | `/api/v1/me/api-tokens` | — | Список API-токенов (без секретов) | 6 |
| POST | `/api/v1/me/api-tokens` | — | Создание токена; секрет показывается **один раз** в ответе | 6 |
| DELETE | `/api/v1/me/api-tokens/{token_id}` | — | Отзыв токена | 6 |
| — | `/api/v1/me/mfa/*` | — | TOTP/WebAuthn — после MVP; таблица заложена | после MVP |

Пример `POST /api/v1/auth/login`:

```json
// запрос
{ "email": "owner@example.ru", "password": "…" }
// 200 + Set-Cookie: delo_session=…; delo_csrf=…
{ "user": { "id": "…", "email": "owner@example.ru", "display_name": "Иван", "locale": "ru" } }
```

При неверных данных ответ одинаковый — `401 unauthenticated` — независимо от того, существует ли пользователь. Время ответа выравнивается фиктивной проверкой Argon2.

### 3.2. Компании и доступ (`delo-tenancy`)

| Метод | Путь | Разрешение | Описание | Этап |
|---|---|---|---|---|
| GET | `/api/v1/companies` | — | Компании, где пользователь состоит | 1 |
| POST | `/api/v1/companies` | — | Создать компанию; создатель становится `owner` | 1 |
| GET | `…` | `company.read` | Профиль компании | 1 |
| PATCH | `…` | `company.manage` | Название, описание, отрасль, валюта, часовой пояс | 1 |
| DELETE | `…` | только `owner` | Мягкое удаление; требует подтверждения названием | 1 |
| POST | `…/transfer-ownership` | только `owner` | Передать владение участнику **[IK]** | 1 |
| POST | `…/agents-pause` | `agents.control` | «Стоп-кран»: `{ "paused": true, "reason": "…" }` | 3 |
| GET | `…/me/permissions` | — | Эффективные разрешения текущего пользователя (только для отображения в UI) | 1 |
| GET | `…/members` | `members.read` | Участники | 1 |
| PATCH | `…/members/{user_id}` | `members.manage` | Сменить роли | 1 |
| DELETE | `…/members/{user_id}` | `members.manage` | Исключить (владельца исключить нельзя) | 1 |
| GET | `…/invitations` | `members.manage` | Приглашения | 1 |
| POST | `…/invitations` | `members.manage` | Пригласить по email с ролью | 1 |
| DELETE | `…/invitations/{id}` | `members.manage` | Отозвать приглашение | 1 |
| POST | `/api/v1/invitations/{token}/accept` | — | Принять приглашение | 1 |
| GET | `…/roles` | `members.read` | Системные и собственные роли | 1 |
| POST | `…/roles` | `roles.manage` | Создать роль из набора разрешений | 1 |
| PATCH | `…/roles/{role_id}` | `roles.manage` | Изменить роль (системные — нельзя) | 1 |
| DELETE | `…/roles/{role_id}` | `roles.manage` | Удалить роль без участников | 1 |
| GET | `…/goals` | `company.read` | Цели компании | 2 |
| POST / PATCH / DELETE | `…/goals[/{id}]` | `company.manage` | Управление целями | 2 |

Пользователь не может выдать роль с разрешениями, которых нет у него самого: защита от эскалации через `roles.manage`.

### 3.3. Оргструктура (`delo-org`)

| Метод | Путь | Разрешение | Описание | Этап |
|---|---|---|---|---|
| GET | `…/org/tree` | `org.read` | Дерево подразделений с должностями и назначениями (людьми и агентами) | 2 |
| POST | `…/org/units` | `org.manage` | Создать подразделение (`parent_id`, название, руководящая должность) | 2 |
| PATCH | `…/org/units/{id}` | `org.manage` | Переименовать или переместить; проверяется отсутствие цикла **[If-Match]** | 2 |
| DELETE | `…/org/units/{id}` | `org.manage` | Только пустое подразделение | 2 |
| POST | `…/org/positions` | `org.manage` | Создать должность в подразделении | 2 |
| PATCH / DELETE | `…/org/positions/{id}` | `org.manage` | Изменить или удалить должность | 2 |
| POST | `…/org/positions/{id}/assignments` | `org.manage` | Назначить на должность пользователя или агента | 2 |
| DELETE | `…/org/positions/{id}/assignments/{assignment_id}` | `org.manage` | Снять назначение | 2 |

### 3.4. Агенты (`delo-agents`)

| Метод | Путь | Разрешение | Описание | Этап |
|---|---|---|---|---|
| GET | `/api/v1/agent-templates` | — | Системные шаблоны: CEO, BA, CFO, Marketing, Sales, Support, CTO, HR | 2 |
| GET | `…/agent-templates` | `agents.read` | Системные и собственные шаблоны компании | 2 |
| POST / PATCH / DELETE | `…/agent-templates[/{id}]` | `agents.manage` | Собственные шаблоны | 2 |
| GET | `…/agents` | `agents.read` | Список: статус, должность, текущий запуск, расход за период | 2 |
| POST | `…/agents` | `agents.manage` | Создать агента (из шаблона или с нуля) | 2 |
| GET | `…/agents/{id}` | `agents.read` | Карточка агента: текущая версия, гранты, лимиты, метрики | 2 |
| PATCH | `…/agents/{id}` | `agents.manage` | Название, описание, статус (`active`/`paused`/`disabled`) | 2 |
| POST | `…/agents/{id}/versions` | `agents.manage` | Новая версия конфигурации (инструкции, модель, параметры, `data_scopes`, лимиты) **[If-Match]** | 2 |
| GET | `…/agents/{id}/versions` | `agents.read` | История версий | 2 |
| GET | `…/agents/{id}/versions/{version_id}` | `agents.read` | Неизменяемый снимок версии | 2 |
| GET | `…/agents/{id}/tool-grants` | `agents.read` | Выданные инструменты | 2 |
| PUT | `…/agents/{id}/tool-grants` | `tools.policy.manage` | Полный набор грантов с переопределениями политики (только ужесточение) **[If-Match]** | 2 |
| GET | `…/agents/{id}/memories` | `agents.read` | Долговременная память | 3 |
| DELETE | `…/agents/{id}/memories/{memory_id}` | `agents.manage` | Удалить запись памяти | 3 |
| GET | `…/agents/{id}/metrics?from&to` | `agents.read` | Задачи, принятые и отклонённые результаты, стоимость, время | 5 |
| DELETE | `…/agents/{id}` | `agents.manage` | Архивирование (история сохраняется) | 2 |

Создать агента или его версию с моделью, которая не разрешена компании, нельзя: `422`. Лимиты версии не могут превышать политику компании. Повышать лимиты расходов агента сверх установленных владельцем может только `owner`.

### 3.5. LLM-провайдеры и модели (`delo-llm`)

| Метод | Путь | Разрешение | Описание | Этап |
|---|---|---|---|---|
| GET | `…/llm/providers` | `llm.read` | Подключённые провайдеры (без ключей) | 2 |
| POST | `…/llm/providers` | `llm.manage` | Подключить провайдер: `kind` (`anthropic`, `openai_compatible`), `base_url`, ключ (шифруется, в ответах не возвращается) | 2 |
| PATCH / DELETE | `…/llm/providers/{id}` | `llm.manage` | Изменить, отключить, заменить ключ | 2 |
| POST | `…/llm/providers/{id}/test` | `llm.manage` | Проверка подключения реальным минимальным запросом; результат и задержка | 2 |
| GET | `…/llm/models` | `llm.read` | Модели, разрешённые компании, с ценами за токен | 2 |
| PUT | `…/llm/models` | `llm.manage` | Разрешённые модели и их цены | 2 |
| GET | `…/llm/usage?from&to&group_by=agent\|model\|day` | `llm.read` | Расход токенов и стоимость | 3 |

Адаптеры российских LLM — **[D-08]**. До решения они в API не появляются.

### 3.6. Инструменты и политики (`delo-tools`)

| Метод | Путь | Разрешение | Описание | Этап |
|---|---|---|---|---|
| GET | `…/tools` | `agents.read` | Каталог инструментов: ключ, категория (`READ`/`WRITE`/`EXTERNAL`/`FINANCIAL`/`CRITICAL`), схемы, доступность | 2 |
| GET | `…/tool-policy` | `agents.read` | Политика компании: решения по категориям, потолок для агентов, пороги, белые списки | 2 |
| PUT | `…/tool-policy` | `tools.policy.manage`; изменение FINANCIAL и CRITICAL — только `owner` | Изменить политику **[If-Match]** | 2 |
| POST | `…/tool-policy/simulate` | `tools.policy.manage` | «Что решит Policy Engine» для агента, инструмента и входа. Ничего не исполняет | 3 |

Ограничения, которые проверяет сервер (не UI):
- FINANCIAL нельзя перевести в `allow`, CRITICAL — ни во что, кроме согласования владельцем;
- `tool-grants` агента не расширяют потолок компании;
- инструменты вывода и перевода денег не существуют в реестре.

### 3.7. Согласования (`delo-approvals`)

| Метод | Путь | Разрешение | Описание | Этап |
|---|---|---|---|---|
| GET | `…/approvals?status=pending&kind=…` | `approvals.read` | Очередь согласований | 3 |
| GET | `…/approvals/counts` | `approvals.read` | Счётчики по статусам (для бейджа в навигации) | 3 |
| GET | `…/approvals/{id}` | `approvals.read` | Описание, точный payload, хеш payload, обоснование агента, риск, ссылки на задачу и запуск | 3 |
| POST | `…/approvals/{id}/decision` | `approvals.decide` (для CRITICAL — только `owner`; для FINANCIAL — ещё и `finance.approve`) | Решение **[IK]** | 3 |
| POST | `…/approvals/{id}/cancel` | инициатор-человек или `approvals.decide` | Отмена до решения | 3 |

```json
// POST …/approvals/{id}/decision
{ "decision": "approve", "payload_hash": "sha256:…", "comment": "Согласовано" }
```

- Сервер сверяет `payload_hash` с сохранённым. Если payload успели изменить, ответ — `409 conflict`: одобряется ровно то, что видел человек.
- Решение принимается только для статуса `pending` и только до `expires_at`.
- Агент не может согласовать что-либо: разрешения `approvals.decide` у агентов нет. Нельзя согласовать и запрос, созданный на основании собственного действия того же пользователя, если включено правило «четырёх глаз» (настройка компании, этап 5).

### 3.8. Запуски агентов (`delo-runtime`)

| Метод | Путь | Разрешение | Описание | Этап |
|---|---|---|---|---|
| POST | `…/tasks/{task_id}/runs` | `agents.control` | Запустить агента-исполнителя по задаче вручную **[IK]**; `423`, если агенты на паузе | 3 |
| GET | `…/runs?agent_id&task_id&status` | `agents.read` | Запуски | 3 |
| GET | `…/runs/{id}` | `agents.read` | Статус, лимиты, потраченное, причина завершения, версия агента | 3 |
| GET | `…/runs/{id}/steps` | `agents.read` | Шаги: вызовы LLM (без сырых промптов по умолчанию), вызовы инструментов, решения Policy Engine | 3 |
| GET | `…/runs/{id}/steps/{step_id}/prompt` | `agents.debug` | Полный собранный контекст шага (для отладки, событие пишется в аудит) | 3 |
| POST | `…/runs/{id}/cancel` | `agents.control` | Запрос отмены (`cancel_requested`) | 3 |
| GET | `…/agent-interactions?task_id` | `agents.read` | Делегирования, вопросы коллегам, эскалации | 4 |

Запуски, как правило, создаются не этим endpoint'ом, а автоматически: при назначении задачи агенту и по событиям (делегирование, решение согласования).

### 3.9. Задачи и проекты (`delo-tasks`)

| Метод | Путь | Разрешение | Описание | Этап |
|---|---|---|---|---|
| GET | `…/tasks` | `tasks.read` | Фильтры: `status`, `assignee_type`, `assignee_id`, `project_id`, `org_unit_id`, `created_by_type`, `due_before`, `parent_id`, `q` | 3 (базово), 4 (полностью) |
| POST | `…/tasks` | `tasks.write` | Создать задачу | 3 |
| GET | `…/tasks/{id}` | `tasks.read` | Карточка: исполнитель, создатель, ожидаемый результат, критерии, режим приёмки, результат, подзадачи, зависимости | 3 |
| PATCH | `…/tasks/{id}` | `tasks.write` | Поля задачи **[If-Match]** | 3 |
| POST | `…/tasks/{id}/assign` | `tasks.assign` | `{ "assignee": { "type": "agent", "id": "…" } }`; типы `user`, `agent`, `org_unit`, `external_contractor` | 3 |
| POST | `…/tasks/{id}/transition` | `tasks.write` | Смена статуса по конечному автомату; недопустимый переход — `409 invalid_transition` | 3 |
| POST | `…/tasks/{id}/review` | `tasks.review` | Принять или вернуть результат: `{ "decision": "accept" \| "request_changes", "comment": "…" }` | 3 |
| GET | `…/tasks/{id}/events` | `tasks.read` | История изменений | 3 |
| GET / POST | `…/tasks/{id}/comments` | `tasks.read` / `tasks.write` | Комментарии | 4 |
| POST / DELETE | `…/tasks/{id}/dependencies[/{depends_on_id}]` | `tasks.write` | Зависимости; цикл → `422` | 4 |
| POST / DELETE | `…/tasks/{id}/attachments[/{file_id}]` | `tasks.write` | Вложения (файл загружается через §3.11) | 4 |
| GET | `…/projects` | `tasks.read` | Проекты | 4 |
| POST / PATCH / DELETE | `…/projects[/{id}]` | `projects.manage` | Управление проектами | 4 |

Человек, принимающий результат, не может быть исполнителем этой же задачи. Автоприёмка (`acceptance_mode = auto`) недоступна для задач, созданных людьми.

### 3.10. База знаний (`delo-knowledge`)

| Метод | Путь | Разрешение | Описание | Этап |
|---|---|---|---|---|
| GET | `…/kb/collections` | `kb.read` | Разделы | 4 |
| POST / PATCH / DELETE | `…/kb/collections[/{id}]` | `kb.manage` | Разделы и доступ к ним (роли, подразделения, агенты) | 4 |
| GET | `…/kb/documents?collection_id&q` | `kb.read` | Документы; `q` — полнотекстовый поиск | 4 |
| POST | `…/kb/documents` | `kb.write` | Создать документ (текст или из загруженного файла) | 4 |
| GET | `…/kb/documents/{id}` | `kb.read` | Документ с текущей версией | 4 |
| POST | `…/kb/documents/{id}/versions` | `kb.write` | Новая версия **[If-Match]** | 4 |
| GET | `…/kb/documents/{id}/versions` | `kb.read` | История версий | 4 |
| DELETE | `…/kb/documents/{id}` | `kb.manage` | Архивирование | 4 |
| GET | `…/kb/search?q&collection_id&limit` | `kb.read` | Поиск по фрагментам с подсветкой | 4 |

Извлечение текста: `txt` и `md` — на этапе 4; `pdf` и `docx` — отдельной задачей того же этапа, с явным статусом обработки документа (`pending`/`ready`/`failed`). Семантический поиск — **[D-09]**.

### 3.11. Файлы (`delo-storage`)

| Метод | Путь | Разрешение | Описание | Этап |
|---|---|---|---|---|
| POST | `…/files` | `files.write` | `multipart/form-data`; проверка размера, MIME по содержимому, sha256 | 4 |
| GET | `…/files/{id}` | доступ к объекту-владельцу | Метаданные | 4 |
| GET | `…/files/{id}/download` | доступ к объекту-владельцу | `302` на presigned URL (TTL 5 минут) или поток | 4 |

Файл хранится в S3-совместимом хранилище с ключом `companies/{company_id}/…`. Публичных ссылок нет.

### 3.12. Финансы (`delo-finance`)

| Метод | Путь | Разрешение | Описание | Этап |
|---|---|---|---|---|
| GET | `…/finance/accounts` | `finance.read` | План счетов | 5 |
| POST / PATCH | `…/finance/accounts[/{id}]` | `finance.manage` | Управление счетами | 5 |
| GET | `…/finance/categories` | `finance.read` | Статьи доходов и расходов | 5 |
| GET | `…/finance/transactions?from&to&account_id` | `finance.read` | Журнал операций | 5 |
| POST | `…/finance/transactions` | `finance.write` | Учётная операция с проводками (сумма проводок = 0) **[IK]** | 5 |
| POST | `…/finance/transactions/{id}/reverse` | `finance.write` | Сторно (операции не редактируются и не удаляются) **[IK]** | 5 |
| GET | `…/finance/reports/summary?from&to` | `finance.read` | Доходы, расходы, остатки, денежный поток | 5 |
| GET | `…/finance/budgets` | `finance.read` | Бюджеты: план и факт | 5 |
| POST / PUT | `…/finance/budgets[/{id}]` | `finance.manage`; утверждение — `finance.approve` | Бюджеты; черновик от агента утверждает человек **[If-Match]** | 5 |
| GET | `…/finance/payment-requests?status` | `finance.read` | Платёжные запросы | 5 |
| POST | `…/finance/payment-requests` | `finance.write` | Создать черновик **[IK]** | 5 |
| POST | `…/finance/payment-requests/{id}/submit` | `finance.write` | На согласование → создаётся `approval_request` | 5 |
| POST | `…/finance/payment-requests/{id}/mark-executed` | `finance.execute` | Отметить «оплачено вне системы» с файлом-подтверждением **[IK]**. Только из `approved` | 5 |
| POST | `…/finance/payment-requests/{id}/cancel` | `finance.write` | Отмена до исполнения | 5 |
| GET / PUT | `…/finance/spending-limits` | `finance.read` / только `owner` | Лимиты расходов агентов и компании | 5 |

Автоматического исполнения платежей нет: порт `PaymentGateway` в MVP без адаптеров. Агент может только создать черновик платёжного запроса через инструмент категории FINANCIAL; этот запрос всегда проходит согласование человеком.

### 3.13. CRM (`delo-crm`)

| Метод | Путь | Разрешение | Описание | Этап |
|---|---|---|---|---|
| GET | `…/counterparties?kind&q` | `crm.read` | Контрагенты: `client`, `supplier`, `contractor`, `partner` | 4 |
| POST / PATCH / DELETE | `…/counterparties[/{id}]` | `crm.write` | Управление контрагентами | 4 |
| — | `…/crm/contacts`, `…/crm/deals` | `crm.*` | Контакты и воронка сделок | 6 |

Справочник контрагентов появляется на этапе 4, потому что он нужен задачам (`external_contractor`) и финансам.

### 3.14. Интеграции (`delo-integrations`)

| Метод | Путь | Разрешение | Описание | Этап |
|---|---|---|---|---|
| GET | `/api/v1/integration-kinds` | — | Доступные типы коннекторов и требуемые scopes | 6 |
| GET | `…/integrations` | `integrations.read` | Подключения и их статус | 6 |
| POST | `…/integrations` | `integrations.manage` | Создать подключение (учётные данные шифруются) | 6 |
| PATCH | `…/integrations/{id}` | `integrations.manage` | Включить/выключить, изменить scopes | 6 |
| POST | `…/integrations/{id}/test` | `integrations.manage` | Реальная проверка соединения | 6 |
| DELETE | `…/integrations/{id}` | `integrations.manage` | Удалить подключение и учётные данные | 6 |
| GET | `…/integrations/{id}/requests` | `integrations.read` | Журнал запросов (без тел с персональными данными) | 6 |

**2А.** Тип `two_a` зарезервирован в схеме БД, но **не появляется** ни в `integration-kinds`, ни где-либо ещё в API, пока нет спецификации API 2А (**[D-10]**). Попытка создать подключение `two_a` возвращает `422 validation_failed`, как для любого неизвестного типа.

### 3.15. Уведомления

| Метод | Путь | Разрешение | Описание | Этап |
|---|---|---|---|---|
| GET | `/api/v1/me/notifications?unread=true` | — | Уведомления пользователя по всем компаниям | 4 |
| POST | `/api/v1/me/notifications/read` | — | Отметить прочитанными: `{ "ids": [...] }` или `{ "all": true }` | 4 |

Каналы доставки, кроме интерфейса (email, мессенджеры), — этап 6.

### 3.16. Аудит

| Метод | Путь | Разрешение | Описание | Этап |
|---|---|---|---|---|
| GET | `…/audit?actor_type&actor_id&action&object_type&object_id&from&to` | `audit.read` | Журнал аудита (только чтение) | 1 |

### 3.17. Главная (dashboard) и лента активности

| Метод | Путь | Разрешение | Описание | Этап |
|---|---|---|---|---|
| GET | `…/dashboard` | `company.read` | Сводка для главной страницы (см. ниже) | 4 |
| GET | `…/activity?actor_type&agent_id&limit&cursor` | `company.read` | Лента деятельности: что делали агенты и люди | 4 |

Ответ `/dashboard` содержит только реальные данные; если модуль ещё не реализован, его блок отсутствует (а не заполнен нулями):

```json
{
  "requires_action": {
    "approvals_pending": 3,
    "tasks_in_review": 5,
    "escalations": 1
  },
  "agents": [
    { "id": "…", "name": "Финансовый директор", "status": "active",
      "current_run": { "id": "…", "task_id": "…", "status": "running", "started_at": "…" } }
  ],
  "tasks": { "in_progress": 12, "overdue": 2, "done_last_7d": 30 },
  "llm_spend": { "period": "month", "spent": { "amount_minor": 120000, "currency": "RUB" },
                 "limit": { "amount_minor": 500000, "currency": "RUB" } },
  "agents_paused": false
}
```

Счётчики отфильтрованы по правам пользователя: например, без `finance.read` блок `llm_spend` отсутствует.

---

## 4. Разрешения (RBAC)

Перечень фиксирован в коде (`delo-iam::domain::Permission`). Роли в БД ссылаются на эти строки; неизвестная строка в `role_permissions` отклоняется при старте и миграции.

### 4.1. Перечень разрешений

| Разрешение | Смысл | Может ли быть у агента |
|---|---|---|
| `company.read` | Видеть профиль, цели, главную, ленту | да |
| `company.manage` | Менять профиль и цели компании | **никогда** |
| `members.read` | Видеть участников и роли | да |
| `members.manage` | Приглашать, исключать, менять роли участников | **никогда** |
| `roles.manage` | Создавать и менять собственные роли | **никогда** |
| `org.read` | Видеть оргструктуру | да |
| `org.manage` | Менять оргструктуру и назначения | нет (в MVP) |
| `agents.read` | Видеть агентов, версии, запуски, шаги | да (в пределах грантов) |
| `agents.manage` | Создавать и настраивать агентов | **никогда** |
| `agents.control` | Запускать, отменять запуски, «стоп-кран» | **никогда** |
| `agents.debug` | Видеть полный собранный контекст LLM | **никогда** |
| `llm.read` | Видеть провайдеров, модели, расход | нет |
| `llm.manage` | Подключать провайдеров, менять модели и цены | **никогда** |
| `tools.policy.manage` | Менять политику инструментов и гранты агентов | **никогда** |
| `approvals.read` | Видеть согласования | нет |
| `approvals.decide` | Принимать решения по согласованиям | **никогда** |
| `tasks.read` | Видеть задачи | да (в пределах `data_scopes`) |
| `tasks.write` | Создавать и менять задачи | да (через инструменты) |
| `tasks.assign` | Назначать исполнителя | да (через `delegate_task`, с ограничениями) |
| `tasks.review` | Принимать результат задачи | только в режиме `reviewer_agent` |
| `projects.manage` | Управлять проектами | нет |
| `kb.read` | Читать базу знаний | да (в пределах `data_scopes`) |
| `kb.write` | Создавать документы и версии | да (через инструменты) |
| `kb.manage` | Управлять разделами и доступом | **никогда** |
| `files.write` | Загружать файлы | нет |
| `finance.read` | Видеть финансы | да (CFO, только чтение отчётов) |
| `finance.write` | Учётные операции, черновики платёжных запросов | только черновики через FINANCIAL-инструмент |
| `finance.manage` | Счета, статьи, бюджеты | нет |
| `finance.approve` | Утверждать бюджеты и платежи (вместе с `approvals.decide`) | **никогда** |
| `finance.execute` | Отмечать платёж исполненным | **никогда** |
| `crm.read` / `crm.write` | Контрагенты, контакты, сделки | да (через инструменты) |
| `integrations.read` | Видеть интеграции и журнал | нет |
| `integrations.manage` | Подключать и отключать интеграции | **никогда** |
| `audit.read` | Читать журнал аудита | нет |

«**Никогда**» — запрет в коде: такие разрешения отбрасываются при вычислении `effective(agent)` независимо от настроек компании.

Действия только для **владельца** (не разрешение, а проверка роли `owner`): удаление компании, передача владения, изменение политики FINANCIAL и CRITICAL, повышение лимитов расходов агентов, решение по согласованиям CRITICAL, `PUT …/finance/spending-limits`.

### 4.2. Системные роли

| Разрешение | owner | admin | manager | member | accountant | viewer |
|---|:-:|:-:|:-:|:-:|:-:|:-:|
| `company.read` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `company.manage` | ✓ | ✓ | | | | |
| `members.read` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `members.manage`, `roles.manage` | ✓ | ✓ | | | | |
| `org.read` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `org.manage` | ✓ | ✓ | | | | |
| `agents.read` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `agents.manage` | ✓ | ✓ | | | | |
| `agents.control` | ✓ | ✓ | ✓ | | | |
| `agents.debug` | ✓ | ✓ | | | | |
| `llm.read` | ✓ | ✓ | ✓ | | ✓ | |
| `llm.manage` | ✓ | ✓ | | | | |
| `tools.policy.manage` | ✓ | ✓ ¹ | | | | |
| `approvals.read` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `approvals.decide` | ✓ | ✓ | ✓ ² | | ✓ ³ | |
| `tasks.read` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `tasks.write`, `tasks.assign`, `tasks.review` | ✓ | ✓ | ✓ | ✓ | | |
| `projects.manage` | ✓ | ✓ | ✓ | | | |
| `kb.read` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `kb.write` | ✓ | ✓ | ✓ | ✓ | ✓ | |
| `kb.manage` | ✓ | ✓ | ✓ | | | |
| `files.write` | ✓ | ✓ | ✓ | ✓ | ✓ | |
| `finance.read` | ✓ | ✓ | | | ✓ | |
| `finance.write`, `finance.manage` | ✓ | | | | ✓ | |
| `finance.approve` | ✓ | | | | ✓ ³ | |
| `finance.execute` | ✓ | | | | ✓ | |
| `crm.read` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `crm.write` | ✓ | ✓ | ✓ | ✓ | | |
| `integrations.read` | ✓ | ✓ | ✓ | | | |
| `integrations.manage` | ✓ | ✓ | | | | |
| `audit.read` | ✓ | ✓ | | | ✓ | |

1. `admin` меняет политику только для READ/WRITE/EXTERNAL. FINANCIAL и CRITICAL меняет только `owner`.
2. `manager` решает согласования категорий WRITE и EXTERNAL. FINANCIAL требует `finance.approve`, CRITICAL — `owner`.
3. `accountant` решает только согласования категории FINANCIAL.

Матрица — предложение для обсуждения (см. [DECISIONS.md](DECISIONS.md)). Она хранится как seed системных ролей и не зашита в код проверок: проверки работают только с разрешениями.

---

## 5. Real-time: поток событий (SSE)

```
GET /api/v1/companies/{company_id}/events/stream
Accept: text/event-stream
Last-Event-ID: <id>     (необязательно; для дочитывания после разрыва)
```

- Разрешение: `company.read`. Сервер фильтрует каждое событие по правам получателя: например, события финансов приходят только при `finance.read`.
- Формат кадра:

```
id: 01JC7Q…            (id записи domain_events, монотонный)
event: task.updated
data: {"type":"task.updated","company_id":"…","object":{"type":"task","id":"…"},"actor":{"type":"agent","id":"…","name":"…"},"occurred_at":"…"}
```

- В событии передаются **ссылки и минимальные поля**, а не полные объекты. Клиент инвалидирует соответствующий ключ TanStack Query и перечитывает данные через REST, где проверяются права. Так поток не становится обходным каналом утечки данных.
- Каждые 25 секунд сервер отправляет `: ping`. После разрыва клиент переподключается с `Last-Event-ID`; сервер досылает события из `domain_events` за последние 24 часа, а при более длинном разрыве отправляет `event: resync`, по которому клиент перечитывает всё.
- Ограничение: не более 5 одновременных потоков на пользователя.

Основные типы событий:

| Событие | Когда |
|---|---|
| `task.created`, `task.updated`, `task.status_changed`, `task.assigned`, `task.result_submitted` | Изменения задач |
| `run.started`, `run.step_completed`, `run.awaiting_approval`, `run.finished` | Ход запуска агента |
| `approval.requested`, `approval.decided`, `approval.expired` | Согласования |
| `agent.updated`, `company.agents_paused` | Состояние агентов |
| `notification.created` | Новое уведомление пользователя |
| `finance.payment_request.updated` | Статус платёжного запроса |
| `kb.document.updated` | Изменение документа базы знаний |

---

## 6. Безопасность API: сводка проверок

| Проверка | Где выполняется |
|---|---|
| Аутентификация, CSRF | Tower middleware |
| Членство в компании, установка `app.company_id` | Middleware извлечения `CompanyScope` |
| Разрешение на действие | Application-слой модуля (`ctx.require(...)`), не в handler'е и не в UI |
| Принадлежность объекта компании | Запросы с `company_id` + RLS как второй рубеж |
| Валидация входа | Типизированные DTO + доменные конструкторы |
| Идемпотентность | Middleware + таблица `idempotency_keys` |
| Лимиты частоты | Tower middleware |
| Размер тела, таймаут запроса | Tower middleware (по умолчанию 1 МБ и 30 секунд; для загрузки файлов — отдельные значения) |
| Секреты в ответах | Не сериализуются: тип `Secret<T>` не реализует `Serialize` |

Агенты не обращаются к HTTP API. Их действия проходят через инструменты, которые вызывают те же use cases с актором-агентом. Поэтому проверки прав и аудит одинаковы для людей и агентов, а Policy Engine добавляет ещё один рубеж поверх них.

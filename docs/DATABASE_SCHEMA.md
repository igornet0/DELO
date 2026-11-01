# ДЕЛО — Схема базы данных

> Статус: **Этап 0, проект схемы.** Миграции ещё не созданы; DDL ниже — целевая схема, из которой на этапах 1–5 будут нарезаны миграции.
> DDL проверен выполнением на чистой БД PostgreSQL 14.17 (см. §13). Целевая версия в production — PostgreSQL 16+.
> Обоснование ключевых решений — [ARCHITECTURE.md](ARCHITECTURE.md), открытые вопросы — [DECISIONS.md](DECISIONS.md).

## 1. Соглашения

| Правило | Описание |
|---|---|
| Первичные ключи | `uuid`, значения UUIDv7 генерирует приложение (сортируемы по времени, нет утечки счётчиков). |
| Арендаторские таблицы | содержат `company_id uuid NOT NULL`; уникальность `(company_id, id)`; связи внутри компании — **составные FK** `(company_id, x_id)`, чтобы БД запрещала ссылки между компаниями. |
| Время | `timestamptz`, UTC. `created_at`, `updated_at` — у изменяемых сущностей. |
| Деньги | `bigint` в минимальных единицах (копейки) + `currency char(3)`. LLM-цены — в микро-единицах (`*_micros`). |
| Перечисления | `text` + `CHECK` (проще эволюционировать, чем `ENUM`). В Rust — enum с `sqlx::Type`. |
| Мягкое удаление | `archived_at` для бизнес-сущностей; физическое удаление — только для технических данных по retention. |
| Акторы | пара `actor_type` + `actor_id` (`user`/`agent`/`system`/`integration`). |
| Гибкие поля | `jsonb` — только для конфигураций и снимков, не для данных, по которым нужна целостность. |
| RLS | на всех таблицах с `company_id` (§12). |
| Префиксы | таблицы сгруппированы по модулям; модуль пишет только в свои таблицы. |

Необходимые расширения: `citext`, `pg_trgm`. Позже — `vector` (pgvector), см. D-09.

## 2. IAM

```sql
CREATE EXTENSION IF NOT EXISTS citext;
CREATE EXTENSION IF NOT EXISTS pg_trgm;

CREATE TABLE users (
    id                uuid PRIMARY KEY,
    email             citext NOT NULL UNIQUE,
    email_verified_at timestamptz,
    display_name      text NOT NULL CHECK (length(display_name) BETWEEN 1 AND 200),
    locale            text NOT NULL DEFAULT 'ru',
    timezone          text NOT NULL DEFAULT 'Europe/Moscow',
    status            text NOT NULL DEFAULT 'active' CHECK (status IN ('active','locked','deleted')),
    created_at        timestamptz NOT NULL DEFAULT now(),
    updated_at        timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE user_credentials (
    user_id           uuid PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
    password_hash     text NOT NULL,                 -- argon2id PHC string
    password_changed_at timestamptz NOT NULL DEFAULT now(),
    failed_attempts   int NOT NULL DEFAULT 0,
    locked_until      timestamptz
);

CREATE TABLE sessions (
    id            uuid PRIMARY KEY,
    user_id       uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    token_hash    bytea NOT NULL UNIQUE,             -- sha256(token)
    created_at    timestamptz NOT NULL DEFAULT now(),
    last_seen_at  timestamptz NOT NULL DEFAULT now(),
    expires_at    timestamptz NOT NULL,
    revoked_at    timestamptz,
    ip            inet,
    user_agent    text,
    mfa_verified  boolean NOT NULL DEFAULT false
);
CREATE INDEX sessions_user_idx ON sessions(user_id) WHERE revoked_at IS NULL;

CREATE TABLE auth_tokens (                            -- сброс пароля, подтверждение email
    id          uuid PRIMARY KEY,
    user_id     uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    purpose     text NOT NULL CHECK (purpose IN ('password_reset','email_verification')),
    token_hash  bytea NOT NULL UNIQUE,
    expires_at  timestamptz NOT NULL,
    used_at     timestamptz,
    created_at  timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE mfa_factors (                            -- зарезервировано, реализация после MVP
    id           uuid PRIMARY KEY,
    user_id      uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    kind         text NOT NULL CHECK (kind IN ('totp','webauthn')),
    secret_enc   bytea NOT NULL,
    label        text,
    created_at   timestamptz NOT NULL DEFAULT now(),
    last_used_at timestamptz
);

CREATE TABLE api_tokens (
    id           uuid PRIMARY KEY,
    user_id      uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    company_id   uuid,                                -- NULL = токен пользователя без привязки к компании
    name         text NOT NULL,
    prefix       text NOT NULL,                       -- видимая часть для идентификации
    token_hash   bytea NOT NULL UNIQUE,
    scopes       text[] NOT NULL,
    expires_at   timestamptz,
    last_used_at timestamptz,
    revoked_at   timestamptz,
    created_at   timestamptz NOT NULL DEFAULT now()
);
```

## 3. Компании, членство, RBAC

```sql
CREATE TABLE companies (
    id              uuid PRIMARY KEY,
    name            text NOT NULL CHECK (length(name) BETWEEN 1 AND 200),
    legal_name      text,
    legal_form      text,                             -- ООО, ИП, АО ...
    inn             text,
    country         char(2) NOT NULL DEFAULT 'RU',
    base_currency   char(3) NOT NULL DEFAULT 'RUB',
    timezone        text NOT NULL DEFAULT 'Europe/Moscow',
    industry        text,
    description     text,                             -- контекст компании для агентов
    stage           text NOT NULL DEFAULT 'idea' CHECK (stage IN ('idea','launch','operating','scaling')),
    agents_paused   boolean NOT NULL DEFAULT false,   -- стоп-кран всех агентов
    settings        jsonb NOT NULL DEFAULT '{}',
    created_by      uuid NOT NULL REFERENCES users(id),
    created_at      timestamptz NOT NULL DEFAULT now(),
    updated_at      timestamptz NOT NULL DEFAULT now(),
    archived_at     timestamptz
);

CREATE TABLE roles (
    id          uuid PRIMARY KEY,
    company_id  uuid REFERENCES companies(id) ON DELETE CASCADE,   -- NULL = системная роль
    key         text NOT NULL,                                     -- owner, admin, ... или custom_*
    name        text NOT NULL,
    description text,
    is_system   boolean NOT NULL DEFAULT false,
    created_at  timestamptz NOT NULL DEFAULT now(),
    CHECK (is_system = (company_id IS NULL))
);
CREATE UNIQUE INDEX roles_system_key_uq  ON roles(key) WHERE company_id IS NULL;
CREATE UNIQUE INDEX roles_company_key_uq ON roles(company_id, key) WHERE company_id IS NOT NULL;

CREATE TABLE role_permissions (
    role_id     uuid NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    permission  text NOT NULL,                        -- проверяется по перечню в коде
    PRIMARY KEY (role_id, permission)
);

CREATE TABLE company_memberships (
    company_id  uuid NOT NULL REFERENCES companies(id) ON DELETE CASCADE,
    user_id     uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role_id     uuid NOT NULL REFERENCES roles(id),
    status      text NOT NULL DEFAULT 'active' CHECK (status IN ('active','suspended')),
    joined_at   timestamptz NOT NULL DEFAULT now(),
    PRIMARY KEY (company_id, user_id)
);
CREATE INDEX memberships_user_idx ON company_memberships(user_id);
-- Инвариант «у компании ровно один owner» обеспечивается use case'ом передачи владения
-- в транзакции с блокировкой строки компании.

CREATE TABLE invitations (
    id          uuid PRIMARY KEY,
    company_id  uuid NOT NULL REFERENCES companies(id) ON DELETE CASCADE,
    email       citext NOT NULL,
    role_id     uuid NOT NULL REFERENCES roles(id),
    token_hash  bytea NOT NULL UNIQUE,
    invited_by  uuid NOT NULL REFERENCES users(id),
    status      text NOT NULL DEFAULT 'pending' CHECK (status IN ('pending','accepted','revoked','expired')),
    expires_at  timestamptz NOT NULL,
    created_at  timestamptz NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX invitations_pending_uq ON invitations(company_id, email) WHERE status = 'pending';

CREATE TABLE company_goals (
    id          uuid PRIMARY KEY,
    company_id  uuid NOT NULL REFERENCES companies(id) ON DELETE CASCADE,
    parent_id   uuid,
    title       text NOT NULL,
    description text,
    metric      text,
    target_value numeric,
    current_value numeric,
    due_date    date,
    status      text NOT NULL DEFAULT 'active' CHECK (status IN ('active','achieved','dropped')),
    created_at  timestamptz NOT NULL DEFAULT now(),
    updated_at  timestamptz NOT NULL DEFAULT now(),
    UNIQUE (company_id, id),
    FOREIGN KEY (company_id, parent_id) REFERENCES company_goals(company_id, id)
);
```

## 4. Оргструктура

```sql
CREATE TABLE org_units (
    id          uuid PRIMARY KEY,
    company_id  uuid NOT NULL REFERENCES companies(id) ON DELETE CASCADE,
    parent_id   uuid,                                 -- дерево; обход — рекурсивный CTE
    name        text NOT NULL,
    kind        text NOT NULL DEFAULT 'department' CHECK (kind IN ('company','division','department','team')),
    description text,
    sort_order  int NOT NULL DEFAULT 0,
    created_at  timestamptz NOT NULL DEFAULT now(),
    updated_at  timestamptz NOT NULL DEFAULT now(),
    archived_at timestamptz,
    UNIQUE (company_id, id),
    FOREIGN KEY (company_id, parent_id) REFERENCES org_units(company_id, id),
    CHECK (parent_id IS NULL OR parent_id <> id)
);
CREATE UNIQUE INDEX org_units_root_uq ON org_units(company_id) WHERE parent_id IS NULL;
-- Отсутствие циклов при перемещении проверяется в use case (рекурсивный CTE под блокировкой).

CREATE TABLE positions (
    id          uuid PRIMARY KEY,
    company_id  uuid NOT NULL REFERENCES companies(id) ON DELETE CASCADE,
    org_unit_id uuid NOT NULL,
    title       text NOT NULL,
    responsibilities text,
    is_head     boolean NOT NULL DEFAULT false,       -- руководитель подразделения
    holder_kind text NOT NULL CHECK (holder_kind IN ('human','agent','any')),
    permissions text[] NOT NULL DEFAULT '{}',         -- дополнительные права должности
    created_at  timestamptz NOT NULL DEFAULT now(),
    updated_at  timestamptz NOT NULL DEFAULT now(),
    archived_at timestamptz,
    UNIQUE (company_id, id),
    FOREIGN KEY (company_id, org_unit_id) REFERENCES org_units(company_id, id)
);
CREATE UNIQUE INDEX positions_one_head_uq ON positions(org_unit_id) WHERE is_head AND archived_at IS NULL;
```

Назначение на должность хранится после таблицы агентов (§5), так как ссылается на них.

## 5. Агенты и LLM

```sql
CREATE TABLE llm_providers (                          -- подключения компании к провайдерам
    id            uuid PRIMARY KEY,
    company_id    uuid REFERENCES companies(id) ON DELETE CASCADE,  -- NULL = платформенный
    kind          text NOT NULL CHECK (kind IN ('anthropic','openai_compatible')),
    name          text NOT NULL,
    base_url      text,
    credential_id uuid,                               -- ссылка на зашифрованный секрет (§10)
    enabled       boolean NOT NULL DEFAULT true,
    created_at    timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE llm_models (
    id                       uuid PRIMARY KEY,
    provider_id              uuid NOT NULL REFERENCES llm_providers(id) ON DELETE CASCADE,
    model_name               text NOT NULL,
    display_name             text NOT NULL,
    context_window           int NOT NULL,
    max_output_tokens        int NOT NULL,
    supports_tools           boolean NOT NULL DEFAULT true,
    price_currency           char(3) NOT NULL DEFAULT 'USD',
    input_price_per_mtok_micros  bigint NOT NULL,     -- цена за 1M токенов в микро-единицах
    output_price_per_mtok_micros bigint NOT NULL,
    enabled                  boolean NOT NULL DEFAULT true,
    UNIQUE (provider_id, model_name)
);

CREATE TABLE agent_templates (
    id           uuid PRIMARY KEY,
    company_id   uuid REFERENCES companies(id) ON DELETE CASCADE,   -- NULL = системный шаблон
    key          text NOT NULL,                       -- ceo, cfo, business_analyst ...
    name         text NOT NULL,
    role_summary text NOT NULL,
    instructions text NOT NULL,
    default_tools text[] NOT NULL DEFAULT '{}',
    default_limits jsonb NOT NULL DEFAULT '{}',
    default_org_unit_hint text,
    version      int NOT NULL DEFAULT 1,
    created_at   timestamptz NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX agent_templates_system_uq ON agent_templates(key) WHERE company_id IS NULL;

CREATE TABLE agents (
    id                 uuid PRIMARY KEY,
    company_id         uuid NOT NULL REFERENCES companies(id) ON DELETE CASCADE,
    template_id        uuid REFERENCES agent_templates(id),
    name               text NOT NULL,
    role_title         text NOT NULL,
    description        text,
    status             text NOT NULL DEFAULT 'active' CHECK (status IN ('active','paused','disabled')),
    current_version_id uuid,                          -- FK добавляется после agent_versions
    max_concurrent_runs int NOT NULL DEFAULT 2 CHECK (max_concurrent_runs BETWEEN 1 AND 50),
    created_by         uuid NOT NULL REFERENCES users(id),
    created_at         timestamptz NOT NULL DEFAULT now(),
    updated_at         timestamptz NOT NULL DEFAULT now(),
    archived_at        timestamptz,
    UNIQUE (company_id, id)
);

CREATE TABLE agent_versions (                         -- неизменяемые снимки конфигурации
    id              uuid PRIMARY KEY,
    company_id      uuid NOT NULL,
    agent_id        uuid NOT NULL,
    version         int NOT NULL,
    instructions    text NOT NULL,
    model_id        uuid NOT NULL REFERENCES llm_models(id),
    temperature     real,
    limits          jsonb NOT NULL,                   -- max_steps, max_cost_micros, max_wall_time_s ...
    data_scopes     jsonb NOT NULL DEFAULT '{}',      -- разделы БЗ, подразделения, к которым есть доступ
    created_by      uuid NOT NULL REFERENCES users(id),
    created_at      timestamptz NOT NULL DEFAULT now(),
    UNIQUE (company_id, id),
    UNIQUE (agent_id, version),
    FOREIGN KEY (company_id, agent_id) REFERENCES agents(company_id, id) ON DELETE CASCADE
);
ALTER TABLE agents ADD CONSTRAINT agents_current_version_fk
    FOREIGN KEY (company_id, current_version_id) REFERENCES agent_versions(company_id, id)
    DEFERRABLE INITIALLY DEFERRED;

CREATE TABLE position_assignments (
    id           uuid PRIMARY KEY,
    company_id   uuid NOT NULL,
    position_id  uuid NOT NULL,
    user_id      uuid REFERENCES users(id),
    agent_id     uuid,
    started_at   timestamptz NOT NULL DEFAULT now(),
    ended_at     timestamptz,
    FOREIGN KEY (company_id, position_id) REFERENCES positions(company_id, id),
    FOREIGN KEY (company_id, agent_id)    REFERENCES agents(company_id, id),
    CHECK ((user_id IS NULL) <> (agent_id IS NULL))
);
CREATE UNIQUE INDEX position_assignments_active_uq ON position_assignments(position_id) WHERE ended_at IS NULL;

CREATE TABLE agent_memories (
    id          uuid PRIMARY KEY,
    company_id  uuid NOT NULL,
    agent_id    uuid NOT NULL,
    kind        text NOT NULL CHECK (kind IN ('fact','preference','lesson','summary')),
    content     text NOT NULL CHECK (length(content) <= 4000),
    source_run_id uuid,
    trust       text NOT NULL DEFAULT 'internal' CHECK (trust IN ('trusted','internal','untrusted')),
    created_at  timestamptz NOT NULL DEFAULT now(),
    expires_at  timestamptz,
    search_tsv  tsvector GENERATED ALWAYS AS (to_tsvector('russian', content)) STORED,
    FOREIGN KEY (company_id, agent_id) REFERENCES agents(company_id, id) ON DELETE CASCADE
);
CREATE INDEX agent_memories_tsv_idx ON agent_memories USING gin(search_tsv);

CREATE TABLE agent_metrics_daily (
    company_id       uuid NOT NULL,
    agent_id         uuid NOT NULL,
    day              date NOT NULL,
    runs_total       int NOT NULL DEFAULT 0,
    runs_failed      int NOT NULL DEFAULT 0,
    tasks_completed  int NOT NULL DEFAULT 0,
    results_accepted int NOT NULL DEFAULT 0,
    results_rejected int NOT NULL DEFAULT 0,
    tokens_input     bigint NOT NULL DEFAULT 0,
    tokens_output    bigint NOT NULL DEFAULT 0,
    cost_micros      bigint NOT NULL DEFAULT 0,
    PRIMARY KEY (company_id, agent_id, day),
    FOREIGN KEY (company_id, agent_id) REFERENCES agents(company_id, id) ON DELETE CASCADE
);
```

## 6. Проекты, задачи, взаимодействия

```sql
CREATE TABLE counterparties (
    id          uuid PRIMARY KEY,
    company_id  uuid NOT NULL REFERENCES companies(id) ON DELETE CASCADE,
    kind        text NOT NULL CHECK (kind IN ('client','supplier','contractor','partner')),
    name        text NOT NULL,
    inn         text,
    email       citext,
    phone       text,
    notes       text,
    created_at  timestamptz NOT NULL DEFAULT now(),
    updated_at  timestamptz NOT NULL DEFAULT now(),
    archived_at timestamptz,
    UNIQUE (company_id, id)
);

CREATE TABLE projects (
    id          uuid PRIMARY KEY,
    company_id  uuid NOT NULL REFERENCES companies(id) ON DELETE CASCADE,
    goal_id     uuid,
    name        text NOT NULL,
    description text,
    owner_user_id uuid REFERENCES users(id),
    status      text NOT NULL DEFAULT 'active' CHECK (status IN ('planned','active','on_hold','done','cancelled')),
    review_mode text NOT NULL DEFAULT 'human' CHECK (review_mode IN ('human','reviewer_agent','auto')),
    start_date  date,
    due_date    date,
    created_at  timestamptz NOT NULL DEFAULT now(),
    updated_at  timestamptz NOT NULL DEFAULT now(),
    archived_at timestamptz,
    UNIQUE (company_id, id),
    FOREIGN KEY (company_id, goal_id) REFERENCES company_goals(company_id, id)
);

CREATE TABLE tasks (
    id                uuid PRIMARY KEY,
    company_id        uuid NOT NULL REFERENCES companies(id) ON DELETE CASCADE,
    number            bigint NOT NULL,                -- человекочитаемый номер внутри компании
    project_id        uuid,
    parent_task_id    uuid,
    root_task_id      uuid,                           -- корень дерева делегирования (бюджет, лимиты)
    delegation_depth  int NOT NULL DEFAULT 0 CHECK (delegation_depth BETWEEN 0 AND 10),
    title             text NOT NULL CHECK (length(title) BETWEEN 1 AND 500),
    description       text,
    expected_result   text,
    acceptance_criteria text,
    status            text NOT NULL DEFAULT 'todo'
                      CHECK (status IN ('backlog','todo','in_progress','blocked','in_review','done','cancelled')),
    priority          smallint NOT NULL DEFAULT 2 CHECK (priority BETWEEN 0 AND 4),
    review_mode       text CHECK (review_mode IN ('human','reviewer_agent','auto')),  -- NULL = из проекта
    -- исполнитель: ровно один вид
    assignee_kind     text CHECK (assignee_kind IN ('user','agent','org_unit','counterparty')),
    assignee_user_id  uuid REFERENCES users(id),
    assignee_agent_id uuid,
    assignee_org_unit_id uuid,
    assignee_counterparty_id uuid,
    reviewer_user_id  uuid REFERENCES users(id),
    reviewer_agent_id uuid,
    -- кто создал и почему
    created_by_type   text NOT NULL CHECK (created_by_type IN ('user','agent','system','integration')),
    created_by_id     uuid,
    created_by_run_id uuid,
    reason            text,
    due_at            timestamptz,
    started_at        timestamptz,
    completed_at      timestamptz,
    result_summary    text,
    created_at        timestamptz NOT NULL DEFAULT now(),
    updated_at        timestamptz NOT NULL DEFAULT now(),
    UNIQUE (company_id, id),
    UNIQUE (company_id, number),
    FOREIGN KEY (company_id, project_id)        REFERENCES projects(company_id, id),
    FOREIGN KEY (company_id, parent_task_id)    REFERENCES tasks(company_id, id),
    FOREIGN KEY (company_id, root_task_id)      REFERENCES tasks(company_id, id),
    FOREIGN KEY (company_id, assignee_agent_id) REFERENCES agents(company_id, id),
    FOREIGN KEY (company_id, assignee_org_unit_id) REFERENCES org_units(company_id, id),
    FOREIGN KEY (company_id, assignee_counterparty_id) REFERENCES counterparties(company_id, id),
    FOREIGN KEY (company_id, reviewer_agent_id) REFERENCES agents(company_id, id),
    CHECK (
        (assignee_kind IS NULL AND num_nonnulls(assignee_user_id, assignee_agent_id, assignee_org_unit_id, assignee_counterparty_id) = 0)
     OR (assignee_kind = 'user'         AND assignee_user_id IS NOT NULL         AND num_nonnulls(assignee_agent_id, assignee_org_unit_id, assignee_counterparty_id) = 0)
     OR (assignee_kind = 'agent'        AND assignee_agent_id IS NOT NULL        AND num_nonnulls(assignee_user_id, assignee_org_unit_id, assignee_counterparty_id) = 0)
     OR (assignee_kind = 'org_unit'     AND assignee_org_unit_id IS NOT NULL     AND num_nonnulls(assignee_user_id, assignee_agent_id, assignee_counterparty_id) = 0)
     OR (assignee_kind = 'counterparty' AND assignee_counterparty_id IS NOT NULL AND num_nonnulls(assignee_user_id, assignee_agent_id, assignee_org_unit_id) = 0)
    )
);
CREATE INDEX tasks_company_status_idx ON tasks(company_id, status);
CREATE INDEX tasks_assignee_agent_idx ON tasks(company_id, assignee_agent_id) WHERE assignee_agent_id IS NOT NULL;
CREATE INDEX tasks_parent_idx         ON tasks(company_id, parent_task_id);

CREATE TABLE company_counters (                       -- последовательности номеров внутри компании
    company_id uuid NOT NULL REFERENCES companies(id) ON DELETE CASCADE,
    name       text NOT NULL,
    value      bigint NOT NULL DEFAULT 0,
    PRIMARY KEY (company_id, name)
);

CREATE TABLE task_dependencies (
    company_id     uuid NOT NULL,
    task_id        uuid NOT NULL,
    depends_on_id  uuid NOT NULL,
    PRIMARY KEY (task_id, depends_on_id),
    FOREIGN KEY (company_id, task_id)       REFERENCES tasks(company_id, id) ON DELETE CASCADE,
    FOREIGN KEY (company_id, depends_on_id) REFERENCES tasks(company_id, id) ON DELETE CASCADE,
    CHECK (task_id <> depends_on_id)
);
-- Ацикличность графа проверяется в use case рекурсивным CTE.

CREATE TABLE task_comments (
    id          uuid PRIMARY KEY,
    company_id  uuid NOT NULL,
    task_id     uuid NOT NULL,
    author_type text NOT NULL CHECK (author_type IN ('user','agent','system','integration')),
    author_id   uuid,
    run_id      uuid,
    body        text NOT NULL,
    created_at  timestamptz NOT NULL DEFAULT now(),
    edited_at   timestamptz,
    FOREIGN KEY (company_id, task_id) REFERENCES tasks(company_id, id) ON DELETE CASCADE
);

CREATE TABLE task_events (                            -- история задачи (append-only)
    id          uuid PRIMARY KEY,
    company_id  uuid NOT NULL,
    task_id     uuid NOT NULL,
    kind        text NOT NULL,                        -- created, status_changed, assigned, result_submitted ...
    actor_type  text NOT NULL,
    actor_id    uuid,
    run_id      uuid,
    data        jsonb NOT NULL DEFAULT '{}',
    created_at  timestamptz NOT NULL DEFAULT now(),
    FOREIGN KEY (company_id, task_id) REFERENCES tasks(company_id, id) ON DELETE CASCADE
);
CREATE INDEX task_events_task_idx ON task_events(task_id, created_at);

CREATE TABLE agent_interactions (
    id                uuid PRIMARY KEY,
    company_id        uuid NOT NULL,
    kind              text NOT NULL CHECK (kind IN ('delegation','question','escalation','report')),
    initiator_agent_id uuid,
    initiator_user_id uuid REFERENCES users(id),
    initiator_run_id  uuid,
    recipient_kind    text NOT NULL CHECK (recipient_kind IN ('agent','user','org_unit')),
    recipient_id      uuid NOT NULL,
    task_id           uuid NOT NULL,                  -- задача-носитель взаимодействия
    reason            text NOT NULL,
    context           text,
    expected_result   text,
    status            text NOT NULL DEFAULT 'open' CHECK (status IN ('open','answered','rejected','expired','cancelled')),
    response          text,
    created_at        timestamptz NOT NULL DEFAULT now(),
    closed_at         timestamptz,
    FOREIGN KEY (company_id, initiator_agent_id) REFERENCES agents(company_id, id),
    FOREIGN KEY (company_id, task_id) REFERENCES tasks(company_id, id),
    CHECK ((initiator_agent_id IS NULL) <> (initiator_user_id IS NULL))
);

CREATE TABLE notifications (
    id          uuid PRIMARY KEY,
    company_id  uuid REFERENCES companies(id) ON DELETE CASCADE,
    user_id     uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    kind        text NOT NULL,
    title       text NOT NULL,
    body        text,
    link        text,
    data        jsonb NOT NULL DEFAULT '{}',
    read_at     timestamptz,
    created_at  timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX notifications_unread_idx ON notifications(user_id, created_at DESC) WHERE read_at IS NULL;
```

## 7. Исполнение агентов, инструменты, согласования

```sql
CREATE TABLE agent_runs (
    id               uuid PRIMARY KEY,
    company_id       uuid NOT NULL,
    agent_id         uuid NOT NULL,
    agent_version_id uuid NOT NULL,
    task_id          uuid,
    root_task_id     uuid,
    interaction_id   uuid,
    trigger          text NOT NULL CHECK (trigger IN ('task_assigned','question','review','manual','schedule','approval_resumed')),
    status           text NOT NULL DEFAULT 'queued'
                     CHECK (status IN ('queued','running','awaiting_approval','succeeded','failed','cancelled','limit_exceeded')),
    limits           jsonb NOT NULL,                  -- снимок лимитов на момент старта
    tainted          boolean NOT NULL DEFAULT false,  -- в контексте есть недоверенные данные
    cancel_requested boolean NOT NULL DEFAULT false,
    steps_count      int NOT NULL DEFAULT 0,
    tool_calls_count int NOT NULL DEFAULT 0,
    tokens_input     bigint NOT NULL DEFAULT 0,
    tokens_output    bigint NOT NULL DEFAULT 0,
    cost_micros      bigint NOT NULL DEFAULT 0,
    error_code       text,
    error_message    text,
    final_output     text,
    initiated_by_type text NOT NULL,
    initiated_by_id  uuid,
    created_at       timestamptz NOT NULL DEFAULT now(),
    started_at       timestamptz,
    finished_at      timestamptz,
    UNIQUE (company_id, id),
    FOREIGN KEY (company_id, agent_id)         REFERENCES agents(company_id, id),
    FOREIGN KEY (company_id, agent_version_id) REFERENCES agent_versions(company_id, id),
    FOREIGN KEY (company_id, task_id)          REFERENCES tasks(company_id, id)
);
CREATE INDEX agent_runs_active_idx ON agent_runs(company_id, agent_id)
    WHERE status IN ('queued','running','awaiting_approval');
CREATE INDEX agent_runs_task_idx ON agent_runs(company_id, task_id);

CREATE TABLE agent_run_steps (                        -- журнал шагов (append-only)
    id          uuid PRIMARY KEY,
    company_id  uuid NOT NULL,
    run_id      uuid NOT NULL,
    seq         int NOT NULL,
    kind        text NOT NULL CHECK (kind IN ('llm_call','tool_call','tool_result','approval_wait','note','error')),
    content     jsonb NOT NULL,                       -- сообщения/вызовы в нормализованном виде
    model_id    uuid REFERENCES llm_models(id),
    tokens_input  int,
    tokens_output int,
    cost_micros bigint,
    duration_ms int,
    created_at  timestamptz NOT NULL DEFAULT now(),
    UNIQUE (run_id, seq),
    FOREIGN KEY (company_id, run_id) REFERENCES agent_runs(company_id, id) ON DELETE CASCADE
);

CREATE TABLE llm_usage (                              -- учёт расхода для бюджетов и отчётов
    id          uuid PRIMARY KEY,
    company_id  uuid NOT NULL REFERENCES companies(id) ON DELETE CASCADE,
    agent_id    uuid,
    run_id      uuid,
    model_id    uuid NOT NULL REFERENCES llm_models(id),
    tokens_input  int NOT NULL,
    tokens_output int NOT NULL,
    tokens_cached int NOT NULL DEFAULT 0,
    cost_micros bigint NOT NULL,
    cost_currency char(3) NOT NULL,
    provider_request_id text,
    created_at  timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX llm_usage_company_time_idx ON llm_usage(company_id, created_at);

CREATE TABLE tool_definitions (                       -- каталог, синхронизируется из кода при старте
    key            text PRIMARY KEY,                  -- 'tasks.create', 'knowledge.search' ...
    version        int NOT NULL,
    name           text NOT NULL,
    description    text NOT NULL,
    category       text NOT NULL CHECK (category IN ('read','write','external','financial','critical')),
    input_schema   jsonb NOT NULL,
    output_schema  jsonb NOT NULL,
    required_permissions text[] NOT NULL DEFAULT '{}',
    default_limits jsonb NOT NULL DEFAULT '{}',
    integration_kind text,                            -- для инструментов интеграций
    available      boolean NOT NULL DEFAULT true,
    updated_at     timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE company_tool_policies (
    company_id   uuid NOT NULL REFERENCES companies(id) ON DELETE CASCADE,
    tool_key     text NOT NULL REFERENCES tool_definitions(key),
    mode         text NOT NULL CHECK (mode IN ('allow','require_approval','deny')),
    params       jsonb NOT NULL DEFAULT '{}',         -- пороги, белые списки, частота
    updated_by   uuid NOT NULL REFERENCES users(id),
    updated_at   timestamptz NOT NULL DEFAULT now(),
    PRIMARY KEY (company_id, tool_key)
);
-- Запрет ослабления политики для FINANCIAL/CRITICAL — проверка в application-слое
-- (Policy Engine игнорирует 'allow' для этих категорий).

CREATE TABLE agent_tool_grants (
    company_id  uuid NOT NULL,
    agent_id    uuid NOT NULL,
    tool_key    text NOT NULL REFERENCES tool_definitions(key),
    mode_override text CHECK (mode_override IN ('require_approval','deny')),  -- только ужесточение
    params      jsonb NOT NULL DEFAULT '{}',
    granted_by  uuid NOT NULL REFERENCES users(id),
    granted_at  timestamptz NOT NULL DEFAULT now(),
    PRIMARY KEY (agent_id, tool_key),
    FOREIGN KEY (company_id, agent_id) REFERENCES agents(company_id, id) ON DELETE CASCADE
);

CREATE TABLE approval_requests (
    id              uuid PRIMARY KEY,
    company_id      uuid NOT NULL,
    kind            text NOT NULL CHECK (kind IN ('tool_call','payment','budget_change','policy_change','other')),
    title           text NOT NULL,
    description     text NOT NULL,
    payload         jsonb NOT NULL,                   -- точные параметры действия
    payload_hash    bytea NOT NULL,                   -- одобряется ровно этот payload
    risk_level      text NOT NULL CHECK (risk_level IN ('low','medium','high','critical')),
    required_permission text NOT NULL,                -- approvals.decide / finance.approve / owner
    requested_by_type text NOT NULL,
    requested_by_id uuid,
    run_id          uuid,
    task_id         uuid,
    justification   text,
    status          text NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending','approved','rejected','expired','cancelled','executed','failed')),
    decided_by      uuid REFERENCES users(id),
    decided_at      timestamptz,
    decision_comment text,
    expires_at      timestamptz NOT NULL,
    executed_at     timestamptz,
    execution_result jsonb,
    created_at      timestamptz NOT NULL DEFAULT now(),
    UNIQUE (company_id, id),
    FOREIGN KEY (company_id, run_id)  REFERENCES agent_runs(company_id, id),
    FOREIGN KEY (company_id, task_id) REFERENCES tasks(company_id, id),
    CHECK ((status IN ('approved','rejected','executed','failed')) = (decided_by IS NOT NULL))
);
CREATE INDEX approval_requests_pending_idx ON approval_requests(company_id, created_at) WHERE status = 'pending';

CREATE TABLE tool_invocations (
    id              uuid PRIMARY KEY,
    company_id      uuid NOT NULL,
    run_id          uuid NOT NULL,
    step_seq        int NOT NULL,
    call_index      int NOT NULL,
    tool_key        text NOT NULL REFERENCES tool_definitions(key),
    tool_version    int NOT NULL,
    input           jsonb NOT NULL,
    decision        text NOT NULL CHECK (decision IN ('allow','require_approval','deny')),
    decision_reason text NOT NULL,
    approval_id     uuid,
    status          text NOT NULL CHECK (status IN ('pending','running','succeeded','failed','denied','awaiting_approval','cancelled')),
    output          jsonb,
    output_trust    text CHECK (output_trust IN ('trusted','internal','untrusted')),
    error           text,
    duration_ms     int,
    created_at      timestamptz NOT NULL DEFAULT now(),
    finished_at     timestamptz,
    UNIQUE (run_id, step_seq, call_index),            -- идемпотентность исполнения
    FOREIGN KEY (company_id, run_id)      REFERENCES agent_runs(company_id, id),
    FOREIGN KEY (company_id, approval_id) REFERENCES approval_requests(company_id, id)
);
```

## 8. База знаний и файлы

```sql
CREATE TABLE files (
    id           uuid PRIMARY KEY,
    company_id   uuid NOT NULL REFERENCES companies(id) ON DELETE CASCADE,
    storage_key  text NOT NULL UNIQUE,                -- ключ в S3: {company_id}/{yyyy}/{id}
    file_name    text NOT NULL,
    content_type text NOT NULL,
    size_bytes   bigint NOT NULL CHECK (size_bytes >= 0),
    sha256       bytea NOT NULL,
    uploaded_by_type text NOT NULL,
    uploaded_by_id uuid,
    trust        text NOT NULL DEFAULT 'internal' CHECK (trust IN ('trusted','internal','untrusted')),
    created_at   timestamptz NOT NULL DEFAULT now(),
    UNIQUE (company_id, id)
);

CREATE TABLE task_attachments (
    company_id uuid NOT NULL,
    task_id    uuid NOT NULL,
    file_id    uuid NOT NULL,
    added_at   timestamptz NOT NULL DEFAULT now(),
    PRIMARY KEY (task_id, file_id),
    FOREIGN KEY (company_id, task_id) REFERENCES tasks(company_id, id) ON DELETE CASCADE,
    FOREIGN KEY (company_id, file_id) REFERENCES files(company_id, id)
);

CREATE TABLE kb_collections (                         -- разделы БЗ, единица доступа
    id          uuid PRIMARY KEY,
    company_id  uuid NOT NULL REFERENCES companies(id) ON DELETE CASCADE,
    name        text NOT NULL,
    description text,
    visibility  text NOT NULL DEFAULT 'company' CHECK (visibility IN ('company','restricted')),
    created_at  timestamptz NOT NULL DEFAULT now(),
    UNIQUE (company_id, id)
);

CREATE TABLE kb_collection_access (                   -- для restricted-разделов
    company_id    uuid NOT NULL,
    collection_id uuid NOT NULL,
    principal_kind text NOT NULL CHECK (principal_kind IN ('role','user','agent','org_unit')),
    principal_id  uuid NOT NULL,
    access        text NOT NULL CHECK (access IN ('read','write')),
    PRIMARY KEY (collection_id, principal_kind, principal_id),
    FOREIGN KEY (company_id, collection_id) REFERENCES kb_collections(company_id, id) ON DELETE CASCADE
);

CREATE TABLE kb_documents (
    id                 uuid PRIMARY KEY,
    company_id         uuid NOT NULL,
    collection_id      uuid NOT NULL,
    title              text NOT NULL,
    doc_type           text NOT NULL DEFAULT 'note' CHECK (doc_type IN ('note','policy','report','reference','file')),
    current_version_id uuid,
    tags               text[] NOT NULL DEFAULT '{}',
    metadata           jsonb NOT NULL DEFAULT '{}',
    created_by_type    text NOT NULL,
    created_by_id      uuid,
    created_at         timestamptz NOT NULL DEFAULT now(),
    updated_at         timestamptz NOT NULL DEFAULT now(),
    archived_at        timestamptz,
    UNIQUE (company_id, id),
    FOREIGN KEY (company_id, collection_id) REFERENCES kb_collections(company_id, id)
);
CREATE INDEX kb_documents_title_trgm ON kb_documents USING gin(title gin_trgm_ops);

CREATE TABLE kb_document_versions (
    id           uuid PRIMARY KEY,
    company_id   uuid NOT NULL,
    document_id  uuid NOT NULL,
    version      int NOT NULL,
    body         text NOT NULL,                       -- markdown / извлечённый текст
    source_file_id uuid,
    trust        text NOT NULL DEFAULT 'internal' CHECK (trust IN ('trusted','internal','untrusted')),
    change_note  text,
    author_type  text NOT NULL,
    author_id    uuid,
    run_id       uuid,
    created_at   timestamptz NOT NULL DEFAULT now(),
    UNIQUE (company_id, id),
    UNIQUE (document_id, version),
    FOREIGN KEY (company_id, document_id)    REFERENCES kb_documents(company_id, id) ON DELETE CASCADE,
    FOREIGN KEY (company_id, source_file_id) REFERENCES files(company_id, id)
);
ALTER TABLE kb_documents ADD CONSTRAINT kb_documents_current_version_fk
    FOREIGN KEY (company_id, current_version_id) REFERENCES kb_document_versions(company_id, id)
    DEFERRABLE INITIALLY DEFERRED;

CREATE TABLE kb_chunks (
    id           uuid PRIMARY KEY,
    company_id   uuid NOT NULL,
    document_id  uuid NOT NULL,
    version_id   uuid NOT NULL,
    collection_id uuid NOT NULL,                      -- денормализовано для фильтра доступа
    ordinal      int NOT NULL,
    content      text NOT NULL,
    tsv          tsvector NOT NULL,                   -- russian + simple, заполняет индексатор
    -- embedding vector(N)                            -- после включения pgvector (D-09)
    UNIQUE (version_id, ordinal),
    FOREIGN KEY (company_id, version_id) REFERENCES kb_document_versions(company_id, id) ON DELETE CASCADE
);
CREATE INDEX kb_chunks_tsv_idx ON kb_chunks USING gin(tsv);
CREATE INDEX kb_chunks_scope_idx ON kb_chunks(company_id, collection_id);
```

## 9. Финансы

```sql
CREATE TABLE fin_accounts (                           -- план счетов компании
    id          uuid PRIMARY KEY,
    company_id  uuid NOT NULL REFERENCES companies(id) ON DELETE CASCADE,
    code        text NOT NULL,                        -- '51', '62.01' или собственный
    name        text NOT NULL,
    kind        text NOT NULL CHECK (kind IN ('asset','liability','equity','income','expense')),
    is_money    boolean NOT NULL DEFAULT false,       -- денежный счёт (банк, касса) — для cash flow
    currency    char(3) NOT NULL,
    parent_id   uuid,
    archived_at timestamptz,
    UNIQUE (company_id, id),
    UNIQUE (company_id, code),
    FOREIGN KEY (company_id, parent_id) REFERENCES fin_accounts(company_id, id)
);

CREATE TABLE fin_categories (                         -- статьи доходов/расходов для бюджетов и отчётов
    id          uuid PRIMARY KEY,
    company_id  uuid NOT NULL REFERENCES companies(id) ON DELETE CASCADE,
    name        text NOT NULL,
    kind        text NOT NULL CHECK (kind IN ('income','expense')),
    parent_id   uuid,
    UNIQUE (company_id, id),
    FOREIGN KEY (company_id, parent_id) REFERENCES fin_categories(company_id, id)
);

CREATE TABLE fin_transactions (
    id              uuid PRIMARY KEY,
    company_id      uuid NOT NULL REFERENCES companies(id) ON DELETE CASCADE,
    occurred_on     date NOT NULL,
    description     text NOT NULL,
    category_id     uuid,
    counterparty_id uuid,
    project_id      uuid,
    payment_request_id uuid,
    source          text NOT NULL CHECK (source IN ('manual','payment_request','import','integration','reversal')),
    reverses_id     uuid,                             -- сторно вместо изменения/удаления
    idempotency_key text NOT NULL,
    created_by_type text NOT NULL,
    created_by_id   uuid,
    created_at      timestamptz NOT NULL DEFAULT now(),
    UNIQUE (company_id, id),
    UNIQUE (company_id, idempotency_key),
    FOREIGN KEY (company_id, category_id)     REFERENCES fin_categories(company_id, id),
    FOREIGN KEY (company_id, counterparty_id) REFERENCES counterparties(company_id, id),
    FOREIGN KEY (company_id, project_id)      REFERENCES projects(company_id, id),
    FOREIGN KEY (company_id, reverses_id)     REFERENCES fin_transactions(company_id, id)
);

CREATE TABLE fin_entries (                            -- проводки: сумма по транзакции и валюте = 0
    id             uuid PRIMARY KEY,
    company_id     uuid NOT NULL,
    transaction_id uuid NOT NULL,
    account_id     uuid NOT NULL,
    amount_minor   bigint NOT NULL CHECK (amount_minor <> 0),   -- >0 дебет, <0 кредит
    currency       char(3) NOT NULL,
    FOREIGN KEY (company_id, transaction_id) REFERENCES fin_transactions(company_id, id),
    FOREIGN KEY (company_id, account_id)     REFERENCES fin_accounts(company_id, id)
);
CREATE INDEX fin_entries_account_idx ON fin_entries(company_id, account_id);
CREATE INDEX fin_entries_tx_idx      ON fin_entries(transaction_id);

CREATE FUNCTION fin_check_balanced() RETURNS trigger LANGUAGE plpgsql AS $$
DECLARE
    unbalanced int;
BEGIN
    SELECT count(*) INTO unbalanced FROM (
        SELECT currency FROM fin_entries
        WHERE transaction_id = NEW.transaction_id
        GROUP BY currency HAVING sum(amount_minor) <> 0
    ) s;
    IF unbalanced > 0 THEN
        RAISE EXCEPTION 'fin_transaction % is unbalanced', NEW.transaction_id
            USING ERRCODE = 'check_violation';
    END IF;
    RETURN NULL;
END $$;

CREATE CONSTRAINT TRIGGER fin_entries_balanced
    AFTER INSERT ON fin_entries
    DEFERRABLE INITIALLY DEFERRED
    FOR EACH ROW EXECUTE FUNCTION fin_check_balanced();

CREATE FUNCTION forbid_mutation() RETURNS trigger LANGUAGE plpgsql AS $$
BEGIN
    RAISE EXCEPTION '% is append-only', TG_TABLE_NAME USING ERRCODE = 'insufficient_privilege';
END $$;

-- Проводки и транзакции неизменяемы: исправление — только сторно.
CREATE TRIGGER fin_entries_immutable BEFORE UPDATE OR DELETE ON fin_entries
    FOR EACH ROW EXECUTE FUNCTION forbid_mutation();
CREATE TRIGGER fin_transactions_immutable BEFORE UPDATE OR DELETE ON fin_transactions
    FOR EACH ROW EXECUTE FUNCTION forbid_mutation();

CREATE TABLE budgets (
    id          uuid PRIMARY KEY,
    company_id  uuid NOT NULL REFERENCES companies(id) ON DELETE CASCADE,
    name        text NOT NULL,
    period_start date NOT NULL,
    period_end  date NOT NULL CHECK (period_end >= period_start),
    currency    char(3) NOT NULL,
    status      text NOT NULL DEFAULT 'draft' CHECK (status IN ('draft','pending_approval','active','closed')),
    created_by_type text NOT NULL,
    created_by_id uuid,
    approved_by uuid REFERENCES users(id),
    approved_at timestamptz,
    created_at  timestamptz NOT NULL DEFAULT now(),
    UNIQUE (company_id, id)
);

CREATE TABLE budget_lines (
    id           uuid PRIMARY KEY,
    company_id   uuid NOT NULL,
    budget_id    uuid NOT NULL,
    category_id  uuid NOT NULL,
    project_id   uuid,
    planned_minor bigint NOT NULL CHECK (planned_minor >= 0),
    FOREIGN KEY (company_id, budget_id)   REFERENCES budgets(company_id, id) ON DELETE CASCADE,
    FOREIGN KEY (company_id, category_id) REFERENCES fin_categories(company_id, id),
    FOREIGN KEY (company_id, project_id)  REFERENCES projects(company_id, id)
);

CREATE TABLE payment_requests (
    id               uuid PRIMARY KEY,
    company_id       uuid NOT NULL REFERENCES companies(id) ON DELETE CASCADE,
    counterparty_id  uuid NOT NULL,
    amount_minor     bigint NOT NULL CHECK (amount_minor > 0),
    currency         char(3) NOT NULL,
    purpose          text NOT NULL,
    category_id      uuid,
    project_id       uuid,
    task_id          uuid,
    due_on           date,
    status           text NOT NULL DEFAULT 'draft'
                     CHECK (status IN ('draft','pending_approval','approved','rejected','executing','executed','failed','cancelled')),
    execution_method text CHECK (execution_method IN ('manual','gateway')),
    approval_id      uuid,
    transaction_id   uuid,
    external_ref     text,                            -- номер платёжного поручения и т.п.
    idempotency_key  text NOT NULL,
    requested_by_type text NOT NULL,
    requested_by_id  uuid,
    run_id           uuid,
    created_at       timestamptz NOT NULL DEFAULT now(),
    updated_at       timestamptz NOT NULL DEFAULT now(),
    UNIQUE (company_id, id),
    UNIQUE (company_id, idempotency_key),
    FOREIGN KEY (company_id, counterparty_id) REFERENCES counterparties(company_id, id),
    FOREIGN KEY (company_id, category_id)     REFERENCES fin_categories(company_id, id),
    FOREIGN KEY (company_id, project_id)      REFERENCES projects(company_id, id),
    FOREIGN KEY (company_id, task_id)         REFERENCES tasks(company_id, id),
    FOREIGN KEY (company_id, approval_id)     REFERENCES approval_requests(company_id, id),
    FOREIGN KEY (company_id, transaction_id)  REFERENCES fin_transactions(company_id, id),
    CHECK (status NOT IN ('approved','executing','executed') OR approval_id IS NOT NULL)
);
ALTER TABLE fin_transactions ADD CONSTRAINT fin_transactions_payment_request_fk
    FOREIGN KEY (company_id, payment_request_id) REFERENCES payment_requests(company_id, id);

CREATE TABLE spending_limits (                        -- лимиты для агентов и LLM
    id           uuid PRIMARY KEY,
    company_id   uuid NOT NULL REFERENCES companies(id) ON DELETE CASCADE,
    subject_kind text NOT NULL CHECK (subject_kind IN ('company','agent')),
    subject_id   uuid,                                -- NULL для company
    resource     text NOT NULL CHECK (resource IN ('llm_cost','payment_request_amount')),
    period       text NOT NULL CHECK (period IN ('per_run','day','month','per_operation')),
    limit_minor  bigint NOT NULL CHECK (limit_minor >= 0),
    currency     char(3) NOT NULL,
    updated_by   uuid NOT NULL REFERENCES users(id),
    updated_at   timestamptz NOT NULL DEFAULT now(),
    CHECK ((subject_kind = 'company') = (subject_id IS NULL))
);
CREATE UNIQUE INDEX spending_limits_uq ON spending_limits(company_id, subject_kind, coalesce(subject_id, '00000000-0000-0000-0000-000000000000'::uuid), resource, period);
```

## 10. Интеграции и секреты

```sql
CREATE TABLE encrypted_secrets (
    id           uuid PRIMARY KEY,
    company_id   uuid REFERENCES companies(id) ON DELETE CASCADE,   -- NULL = платформенный
    purpose      text NOT NULL,                       -- llm_provider, integration ...
    key_version  int NOT NULL,                        -- версия мастер-ключа
    wrapped_dek  bytea NOT NULL,                      -- ключ данных, зашифрованный мастер-ключом
    nonce        bytea NOT NULL,
    ciphertext   bytea NOT NULL,                      -- AES-256-GCM
    created_at   timestamptz NOT NULL DEFAULT now(),
    rotated_at   timestamptz
);
ALTER TABLE llm_providers ADD CONSTRAINT llm_providers_credential_fk
    FOREIGN KEY (credential_id) REFERENCES encrypted_secrets(id);

CREATE TABLE integration_connections (
    id            uuid PRIMARY KEY,
    company_id    uuid NOT NULL REFERENCES companies(id) ON DELETE CASCADE,
    kind          text NOT NULL,                      -- 'two_a' зарезервирован, адаптер не реализован (D-10)
    name          text NOT NULL,
    status        text NOT NULL DEFAULT 'disconnected'
                  CHECK (status IN ('disconnected','connected','error','revoked')),
    enabled       boolean NOT NULL DEFAULT false,
    scopes        text[] NOT NULL DEFAULT '{}',       -- что разрешено передавать/получать
    config        jsonb NOT NULL DEFAULT '{}',        -- несекретные настройки
    credential_id uuid REFERENCES encrypted_secrets(id),
    last_checked_at timestamptz,
    last_error    text,
    created_by    uuid NOT NULL REFERENCES users(id),
    created_at    timestamptz NOT NULL DEFAULT now(),
    updated_at    timestamptz NOT NULL DEFAULT now(),
    UNIQUE (company_id, id)
);

CREATE TABLE integration_requests (                   -- журнал обмена с внешними системами
    id            uuid PRIMARY KEY,
    company_id    uuid NOT NULL,
    connection_id uuid NOT NULL,
    direction     text NOT NULL CHECK (direction IN ('outbound','inbound')),
    operation     text NOT NULL,
    status_code   int,
    outcome       text NOT NULL CHECK (outcome IN ('success','error','denied','timeout')),
    bytes_sent    int,
    bytes_received int,
    duration_ms   int,
    actor_type    text NOT NULL,
    actor_id      uuid,
    run_id        uuid,
    data_categories text[] NOT NULL DEFAULT '{}',     -- какие категории данных переданы (контроль ПДн)
    error         text,
    created_at    timestamptz NOT NULL DEFAULT now(),
    FOREIGN KEY (company_id, connection_id) REFERENCES integration_connections(company_id, id)
);
CREATE INDEX integration_requests_conn_idx ON integration_requests(connection_id, created_at);
```

## 11. Аудит, события, очередь, идемпотентность

```sql
CREATE TABLE audit_log (
    id            uuid PRIMARY KEY,
    company_id    uuid,                               -- NULL для событий уровня пользователя/платформы
    occurred_at   timestamptz NOT NULL DEFAULT now(),
    actor_type    text NOT NULL CHECK (actor_type IN ('user','agent','system','integration')),
    actor_id      uuid,
    action        text NOT NULL,                      -- 'task.created', 'approval.approved' ...
    object_type   text NOT NULL,
    object_id     uuid,
    changes       jsonb,                              -- diff без секретов
    request_id    text,
    run_id        uuid,
    task_id       uuid,
    approval_id   uuid,
    reason        text,
    ip            inet,
    user_agent    text
);
CREATE INDEX audit_log_company_time_idx ON audit_log(company_id, occurred_at DESC);
CREATE INDEX audit_log_object_idx       ON audit_log(object_type, object_id);
CREATE TRIGGER audit_log_immutable BEFORE UPDATE OR DELETE ON audit_log
    FOR EACH ROW EXECUTE FUNCTION forbid_mutation();

CREATE TABLE domain_events (                          -- transactional outbox
    id           bigserial PRIMARY KEY,
    event_id     uuid NOT NULL UNIQUE,
    company_id   uuid,
    event_type   text NOT NULL,
    aggregate_type text NOT NULL,
    aggregate_id uuid NOT NULL,
    payload      jsonb NOT NULL,
    actor_type   text NOT NULL,
    actor_id     uuid,
    occurred_at  timestamptz NOT NULL DEFAULT now(),
    dispatched_at timestamptz
);
CREATE INDEX domain_events_undispatched_idx ON domain_events(id) WHERE dispatched_at IS NULL;

CREATE TABLE jobs (
    id           uuid PRIMARY KEY,
    company_id   uuid,
    queue        text NOT NULL,                       -- agent_run, notifications, indexing, integrations
    kind         text NOT NULL,
    payload      jsonb NOT NULL,
    priority     smallint NOT NULL DEFAULT 0,
    status       text NOT NULL DEFAULT 'pending'
                 CHECK (status IN ('pending','running','succeeded','failed','dead','cancelled')),
    attempts     int NOT NULL DEFAULT 0,
    max_attempts int NOT NULL DEFAULT 5,
    run_after    timestamptz NOT NULL DEFAULT now(),
    locked_by    text,
    locked_until timestamptz,
    dedup_key    text,
    last_error   text,
    created_at   timestamptz NOT NULL DEFAULT now(),
    finished_at  timestamptz
);
CREATE INDEX jobs_fetch_idx ON jobs(queue, priority DESC, run_after) WHERE status = 'pending';
CREATE INDEX jobs_lease_idx ON jobs(locked_until) WHERE status = 'running';
CREATE UNIQUE INDEX jobs_dedup_uq ON jobs(queue, dedup_key) WHERE dedup_key IS NOT NULL AND status IN ('pending','running');

CREATE TABLE idempotency_keys (
    user_id      uuid NOT NULL,
    key          text NOT NULL,
    company_id   uuid,
    method       text NOT NULL,
    path         text NOT NULL,
    request_hash bytea NOT NULL,                      -- тот же ключ с другим телом → 422
    response_status int,
    response_body jsonb,
    created_at   timestamptz NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, key)
);
```

Выборка job'а воркером:

```sql
-- в транзакции
SELECT id, kind, payload FROM jobs
WHERE queue = $1 AND status = 'pending' AND run_after <= now()
ORDER BY priority DESC, run_after
FOR UPDATE SKIP LOCKED
LIMIT $2;
-- затем UPDATE ... SET status='running', locked_by=$worker, locked_until=now()+lease, attempts=attempts+1
```

Просроченная аренда (`status='running' AND locked_until < now()`) возвращается в `pending` диспетчером.

## 12. Row-Level Security

Приложение работает под ролью `delo_app` (без `BYPASSRLS`, без прав владельца таблиц). Миграции — под ролью `delo_owner`.

```sql
-- Шаблон для каждой таблицы с обязательным company_id:
ALTER TABLE tasks ENABLE ROW LEVEL SECURITY;
ALTER TABLE tasks FORCE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON tasks
    USING (company_id = current_setting('app.company_id', true)::uuid)
    WITH CHECK (company_id = current_setting('app.company_id', true)::uuid);
```

- В начале каждой транзакции в контексте компании: `SELECT set_config('app.company_id', $1, true)` (локально для транзакции, безопасно для пула соединений).
- Без установленного `app.company_id` `current_setting(..., true)` возвращает NULL → ни одна строка не видна.
- Таблицы вне компании (`users`, `sessions`, `company_memberships` для списка компаний пользователя) — без RLS по компании; доступ к ним ограничен use case'ами IAM.
- Воркеры: устанавливают `app.company_id` из job'а. Диспетчер outbox и выборка jobs работают под отдельной ролью `delo_worker_system` с доступом только к `jobs`/`domain_events`.
- Решение о включении RLS — **D-03**.

## 13. Проверка DDL

**Выполнено 2026-11-01** на PostgreSQL 14.17 (локально, временная БД `delo_ddl_check`, затем удалена). Все SQL-блоки §2–§12, кроме примера выборки job'а с плейсхолдерами `$1/$2`, выполнены одним скриптом с `ON_ERROR_STOP=1`.

| Проверка | Ожидание | Факт |
|---|---|---|
| Создание всех объектов | без ошибок | ✅ 61 таблица, 4 пользовательских триггера, RLS на `tasks` |
| Сбалансированная транзакция (+10000 / −10000) | фиксируется | ✅ зафиксирована, сумма проводок = 0 |
| Несбалансированная транзакция (+10000 / −9000) | ошибка при `COMMIT` | ✅ `fin_transaction … is unbalanced` (отложенный триггер) |
| `UPDATE`/`DELETE` в `fin_entries` | ошибка | ✅ `fin_entries is append-only` |
| `UPDATE`/`DELETE` в `audit_log` | ошибка | ✅ `audit_log is append-only` |
| Задача компании A с исполнителем-агентом компании B | ошибка FK | ✅ `tasks_company_id_assignee_agent_id_fkey` |
| Задача с двумя исполнителями | ошибка CHECK | ✅ `tasks_check` |
| Задача с одним агентом своей компании | успех | ✅ |
| RLS, роль без `BYPASSRLS`, без `app.company_id` | 0 строк | ✅ 0 из 3 |
| RLS, роль без `BYPASSRLS`, `app.company_id = A` | только строки A | ✅ 2 из 3 (обе задачи компании A) |

Замечание: суперпользователь обходит RLS даже при `FORCE ROW LEVEL SECURITY`, поэтому приложение **не должно** подключаться суперпользователем или владельцем с `BYPASSRLS` (см. §12). Скрипт проверки будет перенесён в `scripts/` и в интеграционные тесты на этапе 1; пока это разовая ручная проверка.

## 14. Связи основных сущностей

```
users ─┬─< company_memberships >─ companies ─┬─< org_units ─< positions ─< position_assignments >─ agents/users
       │            │                        ├─< agents ─< agent_versions >─ llm_models
       │          roles ─< role_permissions  │     └─< agent_tool_grants >─ tool_definitions
       └─< sessions                          ├─< projects ─< tasks ─┬─< task_dependencies
                                             │                      ├─< task_comments / task_events / task_attachments
                                             │                      ├─< agent_runs ─< agent_run_steps
                                             │                      │        └─< tool_invocations >─ approval_requests
                                             │                      └─< agent_interactions
                                             ├─< kb_collections ─< kb_documents ─< kb_document_versions ─< kb_chunks
                                             ├─< fin_accounts / fin_categories / budgets ─< budget_lines
                                             ├─< fin_transactions ─< fin_entries
                                             ├─< payment_requests >─ approval_requests, counterparties
                                             ├─< integration_connections ─< integration_requests
                                             └─< audit_log, domain_events, jobs, llm_usage
```

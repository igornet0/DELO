<<<<<<< HEAD
# ДЕЛО (DELO)

Платформа для создания и управления реальными компаниями с виртуальной командой ИИ-агентов под контролем человека.

> **Статус: Этап 0 (архитектура).** Кода приложения пока нет. Есть только проектная документация. Решения в [docs/DECISIONS.md](docs/DECISIONS.md) ждут подтверждения.

## Документация

| Документ | Содержание |
|---|---|
| [ARCHITECTURE.md](docs/ARCHITECTURE.md) | Архитектура: модули, слои, безопасность, агентный runtime, LLM-слой, развёртывание |
| [DATABASE_SCHEMA.md](docs/DATABASE_SCHEMA.md) | Схема PostgreSQL: сущности, DDL, индексы, RLS, инварианты |
| [API_SPECIFICATION.md](docs/API_SPECIFICATION.md) | REST API: соглашения, endpoint'ы, RBAC, SSE |
| [DEVELOPMENT_ROADMAP.md](docs/DEVELOPMENT_ROADMAP.md) | Этапы 0–7, критерии готовности, отчёт по этапу 0 |
| [DECISIONS.md](docs/DECISIONS.md) | Журнал ADR, решения на подтверждение, реестр рисков |
| [BOARDDO_INTEGRATION.md](docs/BOARDDO_INTEGRATION.md) | Анализ BoardDo как основы ДЕЛО: соответствие модулей, разрывы, варианты (D-13) |

## Стек (план)

- **Backend:** Rust (Tokio, Axum, SQLx, Tower, Tracing), модульный монолит.
- **Frontend:** React, TypeScript, Vite, TanStack Query/Router, Zustand, Tailwind.
- **Инфраструктура:** PostgreSQL, S3-совместимое хранилище, Docker. Redis только при горизонтальном масштабировании.
=======
# DELO
>>>>>>> origin/main

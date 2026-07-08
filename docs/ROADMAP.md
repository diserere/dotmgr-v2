# Roadmap

План развития проекта dotmgr, от pre-development до MVP.

## Принцип

Документ → Architecture → Issue → Branch → Code → PR → Review → Merge.

Каждый этап — отдельный PR. Документ — первый артефакт, код — последний.

## Фазы

### Фаза 0: Pre-development ✅

- [x] PRD — product requirements, user stories, scope
- [x] AGENT_WORKFLOW — регламент работы с агентами
- [x] CONTEXT_AS_CODE — управление контекстом
- [x] CONTRIBUTING — git workflow, коммиты, языковая политика
- [x] CLAUDE.md — инструкции для агента при старте
- [x] ROADMAP — план развития (текущий документ)

### Фаза 1: Architecture & Design

Документ `docs/ARCHITECTURE.md`. Описать:

- Component view: install.sh → python CLI → config repo
- Data model: `.dotmgr.yaml` (манифест), Stow-структура
- CLI spec: флаги, аргументы, exit codes для `init`, `fetch`, `apply`, `list`
- Module breakdown: `sync.py`, `manifest.py`, `backup.py`, `cli.py`
- Error handling: отсутствие git, нет remote, конфликт файлов, отказ прав
- Testing strategy: изоляция ФС-операций для тестов

### Фаза 2: Implementation (MVP)

| № | Задача | Зависимости |
|---|---|---|
| 1 | `install.sh` — bootstrap скрипт | Архитектура |
| 2 | `dotmgr init` — create/clone config repo | Архитектура |
| 3 | Manifest parser — чтение `.dotmgr.yaml` | Архитектура |
| 4 | `dotmgr list` — вывод доступных групп | #3 |
| 5 | `dotmgr apply` — symlink + backup | #3 |
| 6 | `dotmgr fetch` — git pull | #1 |
| 7 | fastfetch package — сквозной happy path | #5, #6 |

### Фаза 3: Verification

- Установка на чистую Ubuntu VM (end-to-end bootstrap)
- Edge cases: конфликты, offline, нет прав
- Автотесты и CI (post-MVP)

### Post-MVP

1. Push (сохранение изменений с хоста в репо)
2. Дополнительные типы групп (скрипты, GUI-конфиги)
3. Секреты (API-ключи, SSH)
4. Per-host конфигурация / шаблонизация
5. Self-update
6. Uninstall

# Implementation Plan: install.sh + CLI stub (задача 1 MVP)

## Что делаем

Bootstrap-скрипт `install.sh` — однострочная установка dotmgr на чистую систему.
CLI-заглушка — простейший Python-скрипт с одной зависимостью.

## DoD

1. `install.sh` клонирует публичный репо в `~/.dotmgr/tool/`
2. Создаёт venv в `~/.dotmgr/tool/venv/`
3. Устанавливает зависимость `PyYAML`
4. Создаёт symlink `~/.local/bin/dotmgr → ~/.dotmgr/tool/venv/bin/dotmgr`
5. `dotmgr` выводит "Hello from dotmgr v0.1" и список установленных зависимостей
6. Python-код использует `yaml` (PyYAML) для вывода — чтобы подтвердить, что зависимость работает

## Промпт для субагента-Developer

```
Ты — Developer в проекте dotmgr. Твоя задача: реализовать первую задачу MVP — install.sh + CLI-заглушку.

## Контекст (обязательно прочитай)

- CLAUDE.md — правила проекта
- docs/PRD.md — product requirements
- docs/ROADMAP.md — план, задача 1
- docs/ARCHITECTURE.md — архитектура, особенно разделы 1.2 (install.sh), 3.1 (init), 4 (Module Breakdown)

## Ключевые архитектурные решения (зафиксированы в обсуждениях, но могут быть неочевидны)

1. Venv — внутри `tool/`, а не рядом: `~/.dotmgr/tool/venv/`
2. Единственная зависимость для MVP — `pyyaml` (PyYAML)
3. CLI-скрипт — простой Python-файл с аргументами через `sys.argv` или minimal `argparse`, без typer/click. Entrypoint: скрипт, установленный через `pip install -e .` (pyproject.toml)
4. `install.sh` принимает env vars: `DOTMGR_REPO`, `DOTMGR_BRANCH`, `DOTMGR_PREFIX`
5. Проверка ОС — soft warning с подтверждением (Ubuntu 24.04+), а не hard block
6. Установка системных пакетов: `git python3 python3-venv python3-pip`
7. Формат трейлера `Co-authored-by: Developer <developer.name@local.ai>` — перед блоком пустая строка, после email ничего

## Что нужно сделать

1. Создать `install.sh` — bash-скрипт с функциями `check_os`, `install_deps`, `clone_tool`, `setup_venv`, `install_cli`, `prompt_init`
2. Создать структуру Python-пакета по архитектуре (раздел 4.1):
   - `src/dotmgr/` с `__init__.py`, `__main__.py`, `cli.py`
   - `pyproject.toml` с `[project.scripts]` entrypoint
3. `cli.py` — функция `main()`, которая выводит "Hello from dotmgr v0.1" + использует `yaml.dump({"dependency": "pyyaml"})` чтобы показать, что зависимость работает
4. Exit codes: 0 — успешно, 1 — аргументы

## Не делать

- Не реализовывать `dotmgr init`, `fetch`, `apply`, `list` — это другие задачи
- Не усложнять CLI-парсинг (аргументы — только `--version` и `--help`)
- Не добавлять тесты (это post-MVP)

## Результат

Верни:
1. Полный текст `install.sh`
2. Структуру `src/dotmgr/` с файлами
3. `pyproject.toml`
```

## Формат работы

1. Оркестратор (OpenCode) запускает субагента-Developer с промптом выше
2. Developer работает в feature-ветке `feat/install-sh`
3. Результат — PR с кодом
4. Человек ревьюит → правки → merge

## Связанные документы

- docs/ARCHITECTURE.md
- docs/PRD.md
- docs/ROADMAP.md
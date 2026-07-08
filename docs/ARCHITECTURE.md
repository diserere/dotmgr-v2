# ARCHITECTURE: dotmgr v1 (MVP)

> **Статус:** Черновик (Draft)
> **Фаза:** Pre-development → Architecture & Design
> **Версия:** 1.0

---

## 1. Component View

### 1.1 Общая архитектура

```
Публичный репо (github.com/<user>/dotmgr)
├── install.sh          # Bash bootstrap
└── src/dotmgr/         # Python-ядро
    ├── cli.py          # Entrypoint, dispatch
    ├── manifest.py     # Парсинг .dotmgr.yaml
    ├── sync.py         # Symlink + backup + rollback
    └── config.py       # Локальное состояние

Приватный репо (github.com/<user>/dotmgr-config)
└── .dotmgr.yaml        # Манифест групп
└── <group>/            # Stow-подобные директории
```

### 1.2 Компоненты системы

#### `install.sh`

**Назначение:** One-liner bootstrap. Устанавливает всё необходимое для работы dotmgr.

**Что делает (последовательно):**
1. **Detect OS** — проверяет `lsb_release -si` / `cat /etc/os-release`. Если не Ubuntu 24.04+ — exit с сообщением. `TODO: уточнить — soft warning или hard block?`
2. **Install system packages** — `apt-get install -y git python3 python3-venv`
3. **Clone public repo** — `git clone https://github.com/<user>/dotmgr.git ~/.dotmgr/tool/`
4. **Create venv** — `python3 -m venv ~/.dotmgr/venv/ && ~/.dotmgr/venv/bin/pip install pyyaml` (и другие зависимости)
5. **Install CLI** — создаёт symlink `~/.local/bin/dotmgr → ~/.dotmgr/venv/bin/dotmgr` (или добавляет PATH в `.bashrc`)
6. **Post-install prompt** — предлагает запустить `dotmgr init`

**Параметры (env vars):**
- `DOTMGR_REPO` — URL публичного репо (по умолчанию `https://github.com/<user>/dotmgr`)
- `DOTMGR_BRANCH` — ветка для установки (по умолчанию `main`)
- `DOTMGR_PREFIX` — путь установки (по умолчанию `~/.dotmgr`)

**Exit codes:**
| Код | Значение |
|---|---|
| 0 | Успешно |
| 1 | Неподдерживаемая ОС |
| 2 | Ошибка установки пакетов (apt fail) |
| 3 | Ошибка клонирования репо |
| 4 | Ошибка создания venv |

#### `~/.dotmgr/` — локальное состояние

```
~/.dotmgr/
├── tool/               # Клонированный публичный репо (исходный код)
├── venv/               # Python virtual environment
├── config/             # Клонированный приватный config-репо (dotmgr-config)
├── backups/            # Бекапы заменённых файлов (apply-backup-<timestamp>/)
└── state.json          # Сериализованное состояние apply
```

#### Python-ядро

Интерпретатор: `~/.dotmgr/venv/bin/python3`. Entrypoint: `~/.dotmgr/venv/bin/dotmgr` (консольный скрипт, установленный через `pip install -e .` или direct symlink).

### 1.3 Поток bootstrap

```
User terminal
    │
    ├── wget -O- <url>/install.sh | bash
    │       │
    │       ├── [1] Проверка ОС (Ubuntu 24.04+)
    │       ├── [2] apt install git python3 python3-venv
    │       ├── [3] git clone <public-repo> ~/.dotmgr/tool/
    │       ├── [4] python3 -m venv ~/.dotmgr/venv/ + pip install
    │       ├── [5] symlink dotmgr → PATH
    │       └── [6] prompt: "Run 'dotmgr init' now? [Y/n]"
    │
    └── dotmgr init
            │
            ├── [1] Проверка: существует ли ~/.dotmgr/config/
            ├── [2] Если да → git pull (fetch + merge)
            ├── [3] Если нет → prompt: создать пустой репо или клонировать?
            │       ├── Создать: git init + scaffold дефолтной структуры
            │       └── Клонировать: git clone <private-repo-url>
            ├── [4] Проверка наличия .dotmgr.yaml в корне config
            └── [5] Сохранение состояния → state.json
```

### 1.4 Поток apply

```
dotmgr fetch
    │
    ├── cd ~/.dotmgr/config/ && git pull
    ├── Если pull неудачен → exit c кодом 1 (см. Error Handling)
    └── Обновление HEAD в state.json

dotmgr apply [group...]
    │
    ├── [1] Freshness check: git status -b (относительно origin)
    │       ├── Если рассинхрон → WARNING, но apply продолжается
    │       └── (user может прервать Ctrl+C)
    ├── [2] Парсинг .dotmgr.yaml → список групп
    ├── [3] Фильтрация по group... (если переданы)
    ├── [4] Для каждой группы:
    │       ├── type: config → stow-like symlink (source → dest)
    │       │       ├── Если dest существует и не symlink → backup
    │       │       ├── Если dest уже symlink на тот же source → skip
    │       │       └── Иначе → backup + symlink
    │       ├── type: package → apt install
    │       └── (будущие типы: script, secret и т.д.)
    ├── [5] Запись state.json (что применено, timestamp)
    └── [6] Если ошибка на любом шаге → rollback
```

---

## 2. Data Model

### 2.1 Манифест `.dotmgr.yaml`

Расположение: корень приватного config-репо (`~/.dotmgr/config/.dotmgr.yaml`).

**Схема:**

```yaml
# .dotmgr.yaml
version: 1                    # Версия формата манифеста

groups:
  git:
    description: "Git configuration"
    type: config              # Тип группы: config | package
    source: git/              # Путь относительно корня config-репо
    dest: "~/"                # Целевая директория в ФС
    # Содержимое git/ маппится на ~/:
    #   git/.gitconfig → ~/.gitconfig
    #   git/.gitignore_global → ~/.gitignore_global

  bash:
    description: "Shell config (bashrc, aliases)"
    type: config
    source: bash/
    dest: "~/"

  fastfetch:
    description: "Fastfetch config + package"
    type: config
    source: fastfetch/
    dest: "~/.config/fastfetch/"

  packages:
    description: "System packages"
    type: package
    packages:                 # Список apt-пакетов
      - fastfetch
      - htop
      - tmux
```

**Правила:**
- `type: config` — обязателен `source` и `dest`
- `type: package` — обязателен `packages`; `source`/`dest` игнорируются
- Одна группа может содержать только один тип (смешанные группы — post-MVP)
- `dest` поддерживает `~` и env-переменные (`$HOME`, `$XDG_CONFIG_HOME`) — резолвятся на стороне Python

### 2.2 Stow-like структура

В config-репо директории зеркалят целевую ФС. Маппинг:

```
dotmgr-config/
├── .dotmgr.yaml
└── <group source>/
    └── <relative path inside group>
        └── <file>

ФС: <group dest>/<relative path inside group>
```

**Пример (группа `git`, source=`git/`, dest=`~/`):**

```
config-repo/git/
    .gitconfig          → ~/.gitconfig
    .gitignore_global   → ~/.gitignore_global

config-repo/bash/
    .bashrc             → ~/.bashrc
    .bash_aliases       → ~/.bash_aliases

config-repo/fastfetch/
    config.jsonc        → ~/.config/fastfetch/config.jsonc
```

Важно: `source` указывает на директорию внутри config-репо, `dest` — на целевую директорию в ФС. Относительные пути внутри source маппятся один-в-один под dest.

### 2.3 Состояние `state.json`

Расположение: `~/.dotmgr/state.json`

**Схема:**

```json
{
  "version": 1,
  "tool_commit": "abc123def",
  "config_repo": {
    "url": "git@github.com:user/dotmgr-config.git",
    "branch": "main",
    "head": "def456abc",
    "last_fetch": "2026-07-08T12:00:00Z"
  },
  "applied_groups": {
    "git": {
      "applied_at": "2026-07-08T12:05:00Z",
      "files": [
        {
          "source": "git/.gitconfig",
          "dest": "/home/user/.gitconfig",
          "type": "symlink"
        }
      ]
    },
    "packages": {
      "applied_at": "2026-07-08T12:06:00Z",
      "packages": ["fastfetch", "htop"]
    }
  },
  "backups": [
    {
      "path": "/home/user/.gitconfig",
      "backup_path": "/home/user/.dotmgr/backups/apply-20260708T120500/.gitconfig",
      "timestamp": "2026-07-08T12:05:00Z"
    }
  ]
}
```

**Назначение:**
- Определять, какие группы уже применены
- Определять freshness (сравнение head с last_fetch)
- Определять, какие файлы были заменены и где их бекапы

---

## 3. CLI Spec

### 3.1 `dotmgr init`

```
dotmgr init [OPTIONS]
```

| Флаг | Тип | По умолчанию | Описание |
|---|---|---|---|
| `--repo-url` | `str` | `git@github.com:<user>/dotmgr-config.git` | URL приватного config-репо |
| `--name` | `str` | `dotmgr-config` | Имя директории в `~/.dotmgr/` |
| `--no-scaffold` | `bool` | `False` | Не создавать дефолтную структуру |
| `--force` | `bool` | `False` | Пересоздать config, если существует |

**Поведение:**
1. Если `~/.dotmgr/config/` не существует:
   - Если `--repo-url` указан → `git clone --recurse-submodules`
   - Если нет → `git init` + scaffold (.dotmgr.yaml с пустыми группами)
2. Если `~/.dotmgr/config/` уже существует:
   - `git pull` (как fetch)
   - Без `--force` → exit 0, сообщение "already initialized"
   - С `--force` → удалить и пересоздать (осторожно!)
3. Валидация: после init проверить наличие `.dotmgr.yaml`; если нет → WARNING

**Exit codes:**
| Код | Значение |
|---|---|
| 0 | Успешно |
| 1 | Ошибка клонирования (нет доступа, неверный URL) |
| 2 | Ошибка создания репо (нет git, нет прав на запись) |
| 3 | Ошибка scaffold (не удалось создать .dotmgr.yaml) |

### 3.2 `dotmgr fetch`

```
dotmgr fetch
```

**Поведение:**
1. `cd ~/.dotmgr/config/ && git pull`
2. Если `config/` не существует → exit 1
3. Если `git pull` падает (network, auth, merge conflict) → exit 1
4. Успех → обновить `state.json.config_repo.head` и `last_fetch`

**Exit codes:**
| Код | Значение |
|---|---|
| 0 | Успешно (есть обновления или уже актуально) |
| 1 | Ошибка (config не инициализирован, нет git, нет remote, network) |

### 3.3 `dotmgr apply [group...]`

```
dotmgr apply [OPTIONS] [GROUP...]
```

| Флаг | Тип | По умолчанию | Описание |
|---|---|---|---|
| `--skip-fetch` | `bool` | `False` | Не проверять freshness перед apply |
| `--backup` | `str` | `always` | `always` / `on-conflict` / `never` |
| `--dry-run` | `bool` | `False` | Показать что будет сделано без изменений |

**Поведение:**
1. Freshness check (если не `--skip-fetch`):
   - `git status -b` → проверить `ahead`/`behind`
   - Если `behind` → WARNING: "Local config is behind remote. Run 'dotmgr fetch' first."
   - Если `ahead` (возможно при ручных правках) → WARNING: "Local config has uncommitted changes."
   - Apply всё равно выполняется (user может прервать Ctrl+C)
2. Парсинг `.dotmgr.yaml`
3. Если `GROUP...` не пуст → фильтрация групп по именам
4. Если `--dry-run` → вывести план и exit
5. Для каждой группы:
   - **type: config:**
     - Рекурсивный обход `source` директории
     - Для каждого файла: вычислить dest-путь (`<dest>/<relative-path>`)
     - Если dest файл существует:
       - Если это symlink, указывающий на правильный source → skip
       - Если это symlink, указывающий не туда → WARNING + перезапись
       - Если это обычный файл → backup + перезапись
     - Backup: копия файла в `~/.dotmgr/backups/apply-<timestamp>/`
     - Создать symlink: `ln -sf <source-full-path> <dest-path>`
   - **type: package:**
     - `sudo apt-get install -y <pkg1> <pkg2> ...`
     - Если `SUDO_ASKPASS` не задан и нет tty → `TODO: как быть без sudo?`
6. Запись в `state.json`

**Exit codes:**
| Код | Значение |
|---|---|
| 0 | Все группы применены успешно |
| 1 | Ошибка парсинга манифеста |
| 2 | Нет config-репо (init не выполнен) |
| 3 | Частичный успех (одна из групп упала) |
| 4 | Ошибка прав (не удалось создать symlink) |

### 3.4 `dotmgr list`

```
dotmgr list [OPTIONS]
```

| Флаг | Тип | По умолчанию | Описание |
|---|---|---|---|
| `--json` | `bool` | `False` | Вывод в JSON |
| `--applied` | `bool` | `False` | Только применённые группы |

**Вывод (plain):**

```
Доступные группы:
  git          Git configuration                      [config]   ✅ applied
  bash         Shell config (bashrc, aliases)         [config]   ❌ not applied
  fastfetch    Fastfetch config + package             [config]   ✅ applied
  packages     System packages                        [package]  ✅ applied
```

**Exit codes:**
| Код | Значение |
|---|---|
| 0 | Успешно |
| 1 | Манифест не найден / невалиден |

---

## 4. Module Breakdown

### 4.1 Структура Python-пакета

```
src/dotmgr/
├── __init__.py
├── __main__.py       # python -m dotmgr support
├── cli.py            # Entrypoint, dispatch
├── manifest.py       # Парсинг и валидация .dotmgr.yaml
├── sync.py           # Apply-логика: symlink, backup, rollback
├── config.py         # state.json read/write
├── errors.py         # Кастомные исключения
└── utils.py          # Вспомогательное (path resolution, git helpers)

install.sh            # Отдельный файл в корне репо (не Python)
```

### 4.2 Модуль `cli.py`

**Ответственность:** Точка входа. Парсинг аргументов, диспетчеризация к соответствующим модулям.

```python
# Ключевые функции (прототипы):
def main() -> None:
    """Entrypoint: typer.ArgumentParser или ручной argparse dispatch."""

def cmd_init(repo_url: str | None, name: str, no_scaffold: bool, force: bool) -> int:
    """Инициализация config-репо."""

def cmd_fetch() -> int:
    """git pull в config-репо."""

def cmd_apply(groups: list[str], skip_fetch: bool, backup: str, dry_run: bool) -> int:
    """Применение групп."""

def cmd_list(json_output: bool, applied_only: bool) -> int:
    """Вывод доступных групп."""
```

**Зависимости:** `manifest`, `sync`, `config`

### 4.3 Модуль `manifest.py`

**Ответственность:** Парсинг, валидация и представление `.dotmgr.yaml`.

```python
@dataclass
class ConfigGroup:
    name: str
    description: str
    group_type: str                # "config" | "package"
    source: str | None = None      # Путь в config-репо (для config-групп)
    dest: str | None = None        # Целевая ФС-директория (для config-групп)
    packages: list[str] | None = None  # Список apt-пакетов (для package-групп)

@dataclass
class Manifest:
    version: int
    groups: dict[str, ConfigGroup]

def load_manifest(path: Path) -> Manifest:
    """Читает и валидирует .dotmgr.yaml. Возвращает Manifest или кидает ManifestError."""

def validate(manifest: Manifest) -> list[str]:
    """Проверяет обязательные поля и типы. Возвращает список ошибок валидации."""
```

**Валидации:**
- `version` должен быть `1`
- Для `type: config`: обязаны присутствовать `source` и `dest`, `packages` отсутствует
- Для `type: package`: обязателен `packages`, `source` и `dest` отсутствуют
- `source` директория должна существовать в config-репо
- Имена групп уникальны

**Ошибки:**
- `ManifestNotFoundError` — файл не найден
- `ManifestParseError` — YAML синтаксическая ошибка
- `ManifestValidationError` — невалидная схема (подробности в сообщении)

### 4.4 Модуль `sync.py`

**Ответственность:** Apply-логика: создание symlink, backup, rollback.

```python
@dataclass
class SyncAction:
    source: Path      # Полный путь в config-репо
    dest: Path        # Целевой путь в ФС
    action: str       # "symlink" | "skip" | "backup+replace"
    backup_path: Path | None = None

@dataclass
class BackupEntry:
    dest: Path
    backup_path: Path
    timestamp: str

class ApplyError(Exception):
    ...

def plan(config_group: ConfigGroup, config_repo_root: Path) -> list[SyncAction]:
    """Вычисляет список действий без их выполнения."""

def execute(actions: list[SyncAction], backup_mode: str) -> list[BackupEntry]:
    """Выполняет действия. При ошибке вызывает rollback."""

def rollback(backup_entries: list[BackupEntry]) -> None:
    """Откатывает backup: возвращает файлы на место, удаляет symlink."""
```

**Ключевые решения:**
- Backup-директория: `~/.dotmgr/backups/apply-<YYYYMMDDTHHMMSS>/`
- При `--dry-run` вызывается только `plan()`, выводится таблица действий
- При ошибке во время `execute()` — немедленный rollback всех уже выполненных действий в рамках текущего apply
- Rollback не затрагивает предыдущие бекапы

### 4.5 Модуль `config.py`

**Ответственность:** Чтение и запись `~/.dotmgr/state.json`.

```python
@dataclass
class ConfigRepoState:
    url: str
    branch: str
    head: str
    last_fetch: str | None

@dataclass
class FileApplyState:
    source: str
    dest: str
    type: str  # "symlink"

@dataclass
class GroupApplyState:
    applied_at: str
    files: list[FileApplyState] | None = None
    packages: list[str] | None = None

@dataclass
class DotmgrState:
    version: int
    tool_commit: str
    config_repo: ConfigRepoState
    applied_groups: dict[str, GroupApplyState]
    backups: list[BackupEntry]

def load(path: Path = DEFAULT_STATE_PATH) -> DotmgrState:
    """Читает state.json. Если файла нет — возвращает дефолтное пустое состояние."""

def save(state: DotmgrState, path: Path = DEFAULT_STATE_PATH) -> None:
    """Сериализует и записывает state.json."""
```

### 4.6 Модуль `utils.py`

```python
def resolve_path(path_str: str) -> Path:
    """Резолвит ~, $HOME, $XDG_CONFIG_HOME и т.д."""

def git_head(repo_path: Path) -> str:
    """Возвращает HEAD commit hash репозитория."""

def git_remote_url(repo_path: Path) -> str:
    """Возвращает remote origin URL."""

def git_status_behind(repo_path: Path) -> bool:
    """Проверяет, отстаёт ли локальный репо от origin."""
```

### 4.7 `install.sh` (bash)

Хранится в корне публичного репо. Не часть Python-пакета.

```bash
#!/usr/bin/env bash
set -euo pipefail

DOTMGR_REPO="${DOTMGR_REPO:-https://github.com/<user>/dotmgr}"
DOTMGR_BRANCH="${DOTMGR_BRANCH:-main}"
DOTMGR_PREFIX="${DOTMGR_PREFIX:-$HOME/.dotmgr}"

main() {
    check_os
    install_deps
    clone_tool
    setup_venv
    install_cli
    prompt_init
}
```

**Поведение:** См. раздел 1.2.

---

## 5. Error Handling

### 5.1 Матрица ошибок

| Ситуация | Где возникает | Действие | User-сообщение |
|---|---|---|---|
| Нет `git` на системе | `install.sh`, шаг 2 | `apt-get install git` (install.sh сам ставит). Если apt fail → exit 2 | `"Failed to install git. Try: sudo apt-get install git"` |
| Нет `python3` / `python3-venv` | `install.sh`, шаг 2 | Аналогично git | `"Failed to install python3/python3-venv"` |
| Нет `git` в runtime | Любая команда (init, fetch) | Проверить `shutil.which('git')` в cli.py → exit | `"git is required but not found. Install: sudo apt-get install git"` |
| Нет интернета / нет доступа к remote | `init --repo-url`, `fetch` | `git clone` / `git pull` падает → exit 1 | `"Failed to reach remote. Check your internet connection and repo URL."` |
| Нет remote URL (config-repo без origin) | `fetch` | Проверить `git remote -v` → exit 1 | `"No remote configured for config repo. Use 'dotmgr init --repo-url <url>'"` |
| Config-repo не инициализирован | `fetch`, `apply`, `list` | Проверить существование `~/.dotmgr/config/.git` → exit | `"Config repo not initialized. Run 'dotmgr init' first."` |
| Dest файл уже существует и не symlink | `apply`, type:config | `--backup=always`: backup + replace. `--backup=on-conflict`: backup + replace. `--backup=never`: WARNING + skip | `"~/.gitconfig already exists. Backed up to ~/.dotmgr/backups/.../.gitconfig"` |
| Dest файл — symlink не туда | `apply`, type:config | WARNING + перезапись | `"~/.gitconfig is a symlink to /other/path. Replacing with dotmgr link."` |
| Нет прав на запись в dest | `apply`, symlink | Ошибка создания symlink → rollback + exit | `"Permission denied: cannot create symlink at /etc/someconfig. Try with sudo."` (apply с sudo — post-MVP) |
| Нет прав на `sudo apt-get install` | `apply`, type:package | Проверить `os.geteuid() == 0` или `sudo -n true 2>/dev/null`. Если нет — exit | `"Package installation requires sudo. Run with sudo or install packages manually."` |
| Манифест повреждён / невалиден | `apply`, `list` | `ManifestParseError` / `ManifestValidationError` → exit | `"Invalid manifest at ~/.dotmgr/config/.dotmgr.yaml: <details>"` |
| Манифест не найден | `apply`, `list` | Проверить существование файла → exit | `"Manifest not found at ~/.dotmgr/config/.dotmgr.yaml. Check your config repo."` |
| Pull с merge conflict | `fetch` | `git pull` падает с конфликтом → exit 1 | `"Merge conflict during fetch. Resolve manually in ~/.dotmgr/config/ then run fetch again."` |
| Группа не найдена | `apply <group>` | Проверить в manifest → WARNING + продолжить с остальными | `"Group 'xyz' not found in manifest. Available groups: git, bash, ..."` |
| `~/.dotmgr` не существует | Любая команда | Проверить существование `~/.dotmgr/` → `init` сам создаст; остальные — exit | `"dotmgr not installed. Run install.sh first."` |

### 5.2 Принципы обработки

1. **Fail fast** — при ошибке валидации или отсутствии обязательных компонентов — немедленный exit с понятным сообщением
2. **Rollback при частичном apply** — если apply прервался на середине, все уже созданные symlink откатываются, бекапы восстанавливаются
3. **Никаких силосообразных ошибок** — каждое сообщение об ошибке содержит: что пошло не так, что сделал dotmgr, что делать пользователю
4. **Non-zero exit codes** — скрипты и CI могут полагаться на exit code

---

## 6. Testing Strategy

### 6.1 Уровни тестирования

| Уровень | Что тестируем | Инструмент | Изоляция |
|---|---|---|---|
| Unit | `manifest.py`, `config.py`, `utils.py` | `pytest` | Чистые функции, без ФС / сети |
| Integration | `sync.py`, `cli.py` (частично) | `pytest` + `tmp_path` fixture | temp-директории, mock git |
| E2E | `install.sh` + полный цикл | `pytest` + Docker / VM | Контейнер с Ubuntu 24.04 |

### 6.2 Unit-тесты

**`test_manifest.py`:**
- Парсинг валидного `.dotmgr.yaml` → корректный `Manifest`
- Парсинг с отсутствующими полями → `ManifestValidationError`
- Парсинг с неправильным `type` → `ManifestValidationError`
- Парсинг пустого / не-YAML файла → `ManifestParseError`
- `validate()` для config-группы без `source` → ошибка
- `validate()` для package-группы без `packages` → ошибка

```python
# Пример fixture
@pytest.fixture
def valid_manifest_yaml(tmp_path):
    content = """
version: 1
groups:
  test:
    description: "Test group"
    type: config
    source: test/
    dest: "~/"
"""
    path = tmp_path / ".dotmgr.yaml"
    path.write_text(content)
    return path
```

**`test_config.py`:**
- Загрузка несуществующего `state.json` → дефолтное состояние
- Загрузка валидного `state.json` → корректный `DotmgrState`
- Save → load → identity (сериализация-десериализация)
- Загрузка повреждённого JSON → исключение

**`test_utils.py`:**
- `resolve_path("~/.config")` → `/home/user/.config`
- `resolve_path("$XDG_CONFIG_HOME/fastfetch")` → корректный путь
- `git_head()` с mock репозиторием

### 6.3 Integration-тесты

**`test_sync.py` (изолированная ФС):**

```python
@pytest.fixture
def config_repo(tmp_path):
    """Создаёт структуру, имитирующую config-репо."""
    repo = tmp_path / "config"
    repo.mkdir()
    (repo / "git").mkdir()
    (repo / "git" / ".gitconfig").write_text("[user]\n\tname = Test\n")
    return repo

@pytest.fixture
def home(tmp_path):
    """Имитация ~/."""
    return tmp_path / "home"

def test_symlink_created(config_repo, home):
    """Группа git: .gitconfig должен быть symlink в ~/."""
    group = ConfigGroup(name="git", type="config", source="git/", dest=str(home))
    actions = plan(group, config_repo)
    execute(actions)
    assert home.joinpath(".gitconfig").is_symlink()
    assert home.joinpath(".gitconfig").read_text() == "[user]\n\tname = Test\n"

def test_backup_on_conflict(config_repo, home):
    """Если ~/.gitconfig уже существует — должен быть backup."""
    existing = home / ".gitconfig"
    existing.write_text("existing config")
    group = ConfigGroup(name="git", type="config", source="git/", dest=str(home))
    actions = plan(group, config_repo)
    execute(actions, backup_mode="always")
    assert existing.is_symlink()  # заменён на symlink
    backups = list((backup_dir).glob("*"))
    assert len(backups) == 1
    assert backups[0].read_text() == "existing config"

def test_rollback_on_failure(config_repo, home):
    """Если apply падает — изменения откатываются."""
    # TODO: смоделировать ошибку создания symlink (например, несуществующий dest)
    pass
```

**`test_cli.py` (интеграция через subprocess или CliRunner):**
- `dotmgr list --json` → корректный JSON
- `dotmgr apply --dry-run` → ничего не меняет
- `dotmgr apply unknown-group` → exit 3 + сообщение

### 6.4 Моки

```python
# conftest.py
@pytest.fixture(autouse=True)
def mock_git_operations(monkeypatch):
    """Заглушка git-операций — не ходим в реальный git."""
    def mock_run(cmd, *args, **kwargs):
        # Имитируем git clone, git pull, git status
        class Result:
            returncode = 0
            stdout = ""
            stderr = ""
        return Result()
    monkeypatch.setattr("subprocess.run", mock_run)
```

**Отдельно mock для `git status -b`:**
```python
def mock_git_status_ahead(monkeypatch):
    """Имитация: локальный репо впереди remote."""
    ...

def mock_git_status_behind(monkeypatch):
    """Имитация: локальный репо позади remote."""
    ...

def mock_git_status_clean(monkeypatch):
    """Имитация: up-to-date."""
    ...
```

### 6.5 E2E-тесты (будущее)

- Docker-образ `ubuntu:24.04`
- `docker run` → `wget ... | bash` → `dotmgr init --repo-url <test-repo>` → `dotmgr apply` → проверка symlink
- CI: GitHub Actions с Ubuntu 24.04 runner (post-MVP)

### 6.6 Что не тестируем (MVP)

- Поддержка других ОС / дистрибутивов
- Push-сценарий (будет в V2)
- Производительность на 1000+ файлах
- Атомарность при kill -9 (graceful degradation, не гарантия)

---

## 7. Non-Goals (явно)

- **Push** (commit + push изменений с машины) — V2
- **Шаблонизация** (per-host Jinja-подстановки) — V2
- **Секреты / encrypted vault** — V2
- **Uninstall** — V2
- **Self-update** — V2
- **Windows / MacOS** — только при появлении реального спроса
- **GUI / TUI** — не планируется

---

## 8. Open Questions / TODOs

- [ ] `install.sh`: должен ли `install.sh` хардкодить URL публичного репо, или принимать параметром? Если хардкод — как менять при форке?
- [ ] `sudo` для `type: package`: как быть, если пользователь не в sudoers и нет пароля в tty? Использовать `pkexec`? Отказаться от apply всей группы?
- [ ] `apply --backup=on-conflict`: критерий конфликта — только существование файла? Или ещё сравнение содержимого?
- [ ] `state.json`: хранить ли историю бекапов (всегда append) или только последний? Если хранить все — как чистить?
- [ ] `init --scaffold`: что именно создавать? Только `.dotmgr.yaml` + `.gitignore`? Или сразу несколько групп-примеров?
- [ ] `verify` (post-MVP): проверка, что symlink валидны и не битые

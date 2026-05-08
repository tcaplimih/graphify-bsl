# Установка и подключение Graphify BSL

Эта инструкция для текущего fork-а:

```text
C:\Projects\RAG\graphify-bsl
```

В нём добавлена поддержка `.bsl`. Если поставить обычный `graphifyy` из PyPI, BSL-поддержки там может не быть.

## Главное

Graphify состоит из двух частей:

| Что | Как часто делать | Где хранится |
|---|---:|---|
| CLI `graphify` с BSL-поддержкой | один раз на машину | `C:\Projects\RAG\graphify-bsl\.venv` |
| Глобальный skill для Codex | один раз на пользователя | профиль Codex |
| Граф проекта `graphify-out/` | отдельно в каждом проекте | корень конкретного проекта |
| Правила проекта `AGENTS.md`, `.codex/hooks.json` | отдельно в каждом проекте | корень конкретного проекта |

Да: каждый проект нужно подключать отдельно, потому что у каждого проекта свой набор файлов, свой `graphify-out/` и свои правила для ассистента.

## 1. Установить Graphify BSL из текущего fork-а

Открой PowerShell без прав администратора:

```powershell
cd C:\Projects\RAG\graphify-bsl
py -3.14 -m venv .venv
.\.venv\Scripts\python.exe -m pip install --upgrade pip
.\.venv\Scripts\python.exe -m pip install -e .
```

Проверь, что BSL parser установлен:

```powershell
.\.venv\Scripts\python.exe -c "import tree_sitter_bsl as tsbsl; from tree_sitter import Language, Parser; Parser(Language(tsbsl.language())); print('ok')"
```

Ожидаемый ответ:

```text
ok
```

## 2. Сделать команду `graphify` доступной из любого проекта

Чтобы не писать полный путь к exe, добавь alias в PowerShell profile.

Открой profile:

```powershell
notepad $PROFILE
```

Добавь строку:

```powershell
function graphify { & "C:\Projects\RAG\graphify-bsl\.venv\Scripts\graphify.exe" @args }
```

Перезапусти PowerShell и проверь:

```powershell
graphify --help
```

Если alias не нужен, можно везде использовать полный путь:

```powershell
C:\Projects\RAG\graphify-bsl\.venv\Scripts\graphify.exe --help
```

Дальше в инструкции используется короткая команда `graphify`.

## 3. Один раз подключить Graphify skill к Codex

Это глобальная установка skill-а для Codex:

```powershell
graphify install --platform codex
```

Дополнительно проверь `~\.codex\config.toml`. Для Codex рекомендуется:

```toml
[features]
multi_agent = true
```

Этот шаг не создаёт граф проекта. Он только учит Codex понимать команду `$graphify` и правила Graphify.

## 4. Подключить конкретный проект

Ниже команды для любого нового проекта. Эти шаги выполняются из корня проекта.

Пример пути:

```powershell
cd C:\Projects\SomeProject
```

### 4.1. Один раз добавить правила Graphify в проект для Codex

```powershell
graphify codex install
```

Этот шаг выполняется один раз на проект, а не после каждого обновления графа.

Команда обновит или создаст:

```text
AGENTS.md
.codex/hooks.json
```

После этого Codex в этом проекте должен читать `graphify-out/GRAPH_REPORT.md` перед архитектурными вопросами и напоминать об обновлении графа после изменений кода.

Команду можно запускать и до, и после создания `graphify-out/`. Практичнее запускать её до первого `graphify update .`, чтобы проектные правила уже были зафиксированы сразу.

### 4.2. Создать первичный code-only граф

```powershell
graphify update .
```

Это работает без API-ключей. Для BSL-проектов этого уже достаточно, чтобы извлечь:

- `.bsl` файлы;
- процедуры и функции;
- `Перем`;
- `Экспорт`;
- `Асинх`;
- директивы `&НаКлиенте`, `&НаСервере...`;
- локальные и межфайловые `calls`.

После команды появится:

```text
graphify-out/
├── graph.json
├── graph.html
└── GRAPH_REPORT.md
```

В дальнейшем после изменений кода или BSL повторяется только:

```powershell
graphify update .
```

`graphify codex install` повторять не нужно, если `AGENTS.md` и `.codex/hooks.json` уже на месте.

### 4.3. При необходимости поставить git hooks

```powershell
graphify hook install
```

Это проектная настройка. Её надо ставить отдельно в каждом git-репозитории, где нужен auto-update/merge-driver для graphify.

## 5. Semantic-граф без своего API-ключа

Есть три разных режима. Их важно не смешивать.

| Режим | Команда | Документы/PDF/картинки | Нужен свой API-ключ |
|---|---|---:|---:|
| Code-only CLI | `graphify update .` | нет | нет |
| Headless CLI semantic | `graphify extract . --backend gemini` | да | да |
| Через Codex/ассистента | `$graphify .` в чате | да | нет отдельного ключа от тебя |

Почему так:

- `graphify update .` — обычный локальный Python-процесс. Он не имеет доступа к текущему LLM-чату, поэтому делает только AST по коду.
- `graphify extract . --backend ...` — локальный процесс сам вызывает внешний LLM backend, поэтому ему нужен API-ключ.
- `$graphify .` в Codex — это workflow ассистента. В этом режиме semantic extraction делает текущий Codex-ассистент, поэтому отдельный `GEMINI_API_KEY`/`OPENAI_API_KEY` не нужен, если платформа Codex уже доступна.

### Как попросить Codex сделать полный semantic-граф

В чате Codex, находясь в нужном проекте, напиши:

```text
$graphify .
```

или обычным текстом:

```text
Построй полный Graphify-граф по этому проекту: код, docs, PDF и картинки.
```

Для обновления после изменений:

```text
$graphify --update
```

или:

```text
Обнови Graphify-граф, включая semantic-часть по документации/PDF/картинкам.
```

Это не PowerShell-команда. Это команда/запрос в чате ассистента.

## 6. Когда что запускать

### Только изменился код или BSL

В PowerShell:

```powershell
graphify update .
```

### Изменилась документация, PDF или картинки, и нет своего API-ключа

В чате Codex:

```text
$graphify --update
```

### Нужен автоматический запуск без Codex-чата

Тогда нужен backend/API-ключ или локальный LLM backend:

```powershell
$env:GEMINI_API_KEY = "<ключ>"
graphify extract . --backend gemini
```

Или, если настроен Ollama:

```powershell
graphify extract . --backend ollama
```

## 7. Проверка подключения проекта

В корне проекта:

```powershell
Test-Path .\graphify-out\GRAPH_REPORT.md
Test-Path .\graphify-out\graph.json
Test-Path .\AGENTS.md
```

Ожидаемо:

```text
True
True
True
```

Открыть интерактивный граф:

```powershell
start .\graphify-out\graph.html
```

Задать ручной CLI-запрос к текущему проекту:

```powershell
graphify query "где используется РассчитатьНалог?"
```

Задать запрос к графу другого проекта:

```powershell
graphify query "где используется РассчитатьНалог?" --graph C:\Projects\OtherProject\graphify-out\graph.json
```

## 8. Что коммитить

Обычно коммитят:

```text
AGENTS.md
.codex/hooks.json
graphify-out/GRAPH_REPORT.md
graphify-out/graph.json
graphify-out/graph.html
```

Обычно не обязательно коммитить:

```text
graphify-out/cache/
graphify-out/manifest.json
graphify-out/cost.json
```

## 9. Если надо индексировать только часть проекта

Создай `.graphifyignore` в корне нужного проекта.

Пример: индексировать только корневые папки `docs`, `embedded-processors`, `main-processor`, `pluggable-module`:

```gitignore
*
!/docs/**
!/embedded-processors/**
!/main-processor/**
!/pluggable-module/**
```

После изменения ignore-файла обнови граф:

```powershell
graphify update .
```

Если нужна semantic-часть по документации без своего API-ключа, попроси Codex:

```text
$graphify --update
```

## 10. Минимальный чеклист для нового BSL-проекта

Выполни в PowerShell:

```powershell
cd C:\Projects\SomeProject
graphify update .
graphify codex install
```

Если нужна semantic-часть по docs/PDF/картинкам без API-ключа, затем в чате Codex:

```text
$graphify --update
```

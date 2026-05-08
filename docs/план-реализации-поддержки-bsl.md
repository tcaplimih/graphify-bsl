# План реализации поддержки BSL в Graphify

> Для агентной реализации: выполнять план по чекбоксам сверху вниз. Перед изменением кода заново прочитать `AGENTS.md`. После изменения code-файлов выполнить `.\.venv\Scripts\python.exe -m graphify update .`, потому что в этом рабочем дереве CLI `graphify` сейчас не находится в `PATH`.

**Цель:** добавить в fork Graphify корректную AST-поддержку файлов 1С:Предприятие 8.3 (`.bsl`) с русскими идентификаторами, директивами исполнения, экспортными процедурами/функциями и базовым call graph.

**Архитектура:** BSL добавляется как отдельный tree-sitter extractor в существующий pipeline `detect -> extract -> build -> cluster -> analyze -> report -> export`. Сначала исправляется Unicode-безопасность идентификаторов, затем подключается `tree-sitter-bsl`, после этого добавляется `extract_bsl()` и тесты. Generic extractor для BSL не используется, потому что AST BSL не имеет отдельного `body` field у процедур/функций и содержит специфичные для 1С директивы `&НаКлиенте`, `&НаСервере`, `Экспорт`, `Асинх`.

**Стек:** Python 3.10+, текущий пакет `graphifyy`, `tree-sitter>=0.23.0`, `tree-sitter-bsl==0.1.6`, `pytest`, `networkx`.

---

## Исходные факты

- Рабочая директория: `C:\Projects\RAG\graphify-bsl`.
- Upstream: `https://github.com/safishamsi/graphify.git`.
- Текущая ветка: `v7`.
- Пакет проекта называется `graphifyy`, но CLI-команда называется `graphify`.
- В `.venv` на момент анализа не были установлены `graphifyy` и `tree-sitter-bsl`; установка из лога ушла в user-site Python 3.14.
- В рабочем дереве уже есть пользовательские изменения: `tests/fixtures/sample.F90` изменен, документы Gemini не отслеживаются. Их не трогать без отдельного указания.
- `graphify-out/GRAPH_REPORT.md` в корне проекта отсутствует, поэтому обязательный graphify-контекст сейчас недоступен.
- `tree-sitter-bsl` опубликован на PyPI, версия `0.1.6`, релиз `2026-03-10`, Python `>=3.10`.
- Узлы грамматики BSL, важные для MVP: `procedure_definition`, `function_definition`, `var_definition`, `method_call`, `call_expression`, `call_statement`, `preprocessor`, `annotation`, `new_expression`.

## Главный риск

Текущий `_make_id()` в `graphify/extract.py` удаляет все символы, кроме ASCII `a-zA-Z0-9`. Для BSL это ломает граф: `РассчитатьИтоги()`, `Печать()` и другие русские имена могут превратиться в одинаковые или пустые ID. Поэтому нельзя начинать с простого добавления `extract_bsl()`: сначала нужна Unicode-безопасная стратегия ID и нормализации.

## Целевые файлы

- Изменить: `pyproject.toml` — добавить зависимость `tree-sitter-bsl`.
- Изменить: `graphify/extract.py` — Unicode-safe ID, `extract_bsl()`, регистрация `.bsl`.
- Изменить: `graphify/build.py` — синхронизировать нормализацию ID и label dedup с Unicode.
- Изменить: `graphify/dedup.py` — не терять кириллицу при dedup.
- Изменить: `graphify/detect.py` — добавить `.bsl` в `CODE_EXTENSIONS`.
- Изменить: `tests/test_languages.py` — тесты экстрактора BSL.
- Изменить: `tests/test_multilang.py` — dispatch-тест для `.bsl`.
- Изменить: `tests/test_detect.py` — классификация `.bsl` как code.
- Изменить: `tests/test_watch.py` — `.bsl` попадает в watched code extensions.
- Создать: `tests/fixtures/sample.bsl` — основной fixture на русском синтаксисе.
- Создать: `tests/fixtures/sample_en.bsl` — fixture на английских BSL keywords, если первый MVP пройдет без нестабильности.
- Изменить: `README.md` — добавить BSL в список code-файлов.
- Изменить: `ARCHITECTURE.md` — уточнить, что BSL использует отдельный extractor.

---

## Этап 1. Подготовить окружение

### Задача 1.1. Убедиться, что используется локальная `.venv`

**Файлы:** не менять.

- [ ] Выполнить:

```powershell
.\.venv\Scripts\python.exe --version
.\.venv\Scripts\python.exe -m pip show graphifyy
.\.venv\Scripts\python.exe -m pip show tree-sitter-bsl
```

**Ожидаемо до правок:** `graphifyy` и `tree-sitter-bsl` могут отсутствовать в `.venv`.

- [ ] Установить проект в editable-режиме именно в `.venv`:

```powershell
.\.venv\Scripts\python.exe -m pip install -e .
```

**Ожидаемо:** `Successfully installed graphifyy-0.7.10`.

- [ ] Проверить импорт:

```powershell
.\.venv\Scripts\python.exe -c "import graphify; print(graphify.__version__)"
```

**Ожидаемо:** печатается версия пакета без traceback.

### Задача 1.2. Зафиксировать стартовые тесты

**Файлы:** не менять.

- [ ] Выполнить быстрый baseline:

```powershell
.\.venv\Scripts\python.exe -m pytest tests/test_languages.py tests/test_multilang.py tests/test_detect.py tests/test_watch.py -q
```

**Ожидаемо:** существующие тесты проходят. Если падает тест, не исправлять вслепую: сначала понять, связан ли он с текущим рабочим деревом или с окружением.

---

## Этап 2. Сделать ID и нормализацию Unicode-безопасными

### Задача 2.1. Написать падающие тесты на Unicode ID

**Файлы:**

- Изменить: `tests/test_extract.py`
- Изменить: `tests/test_build.py`
- Изменить: `tests/test_dedup.py`

- [ ] Добавить в `tests/test_extract.py` тест на `_make_id()`:

```python
from graphify.extract import _make_id


def test_make_id_preserves_cyrillic_identifiers():
    assert _make_id("Модуль", "РассчитатьИтоги") == "модуль_рассчитатьитоги"
    assert _make_id("Модуль", "Печать") == "модуль_печать"
    assert _make_id("Модуль", "РассчитатьИтоги") != _make_id("Модуль", "Печать")
```

- [ ] Добавить в `tests/test_build.py` тест на `_normalize_id()`:

```python
from graphify.build import _normalize_id


def test_normalize_id_preserves_cyrillic_identifiers():
    assert _normalize_id("Модуль:РассчитатьИтоги()") == "модуль_рассчитатьитоги"
    assert _normalize_id("Модуль:Печать()") == "модуль_печать"
```

- [ ] Добавить в `tests/test_dedup.py` тест на `_norm()`:

```python
from graphify.dedup import _norm


def test_dedup_norm_preserves_cyrillic_labels():
    assert _norm("РассчитатьИтоги()") == "рассчитатьитоги"
    assert _norm("Печать Документа") == "печать документа"
```

- [ ] Запустить тесты и убедиться, что они красные:

```powershell
.\.venv\Scripts\python.exe -m pytest tests/test_extract.py::test_make_id_preserves_cyrillic_identifiers tests/test_build.py::test_normalize_id_preserves_cyrillic_identifiers tests/test_dedup.py::test_dedup_norm_preserves_cyrillic_labels -q
```

**Ожидаемо:** тесты падают из-за текущей ASCII-only нормализации.

### Задача 2.2. Исправить `_make_id()`

**Файлы:**

- Изменить: `graphify/extract.py`

- [ ] Заменить реализацию `_make_id()` на Unicode-aware вариант:

```python
def _make_id(*parts: str) -> str:
    """Build a stable node ID from one or more name parts, preserving Unicode letters."""
    combined = "_".join(p.strip("_.") for p in parts if p)
    cleaned = re.sub(r"[^\w]+", "_", combined, flags=re.UNICODE)
    return cleaned.strip("_").lower()
```

- [ ] Запустить целевой тест:

```powershell
.\.venv\Scripts\python.exe -m pytest tests/test_extract.py::test_make_id_preserves_cyrillic_identifiers -q
```

**Ожидаемо:** тест проходит.

### Задача 2.3. Исправить `_normalize_id()` и label normalization

**Файлы:**

- Изменить: `graphify/build.py`
- Изменить: `graphify/dedup.py`

- [ ] В `graphify/build.py` заменить `_normalize_id()` на Unicode-aware вариант:

```python
def _normalize_id(s: str) -> str:
    """Normalize an ID string the same way extract._make_id does."""
    cleaned = re.sub(r"[^\w]+", "_", s, flags=re.UNICODE)
    return cleaned.strip("_").lower()
```

- [ ] В `graphify/build.py` заменить `_norm_label()`:

```python
def _norm_label(label: str) -> str:
    """Canonical dedup key — lowercase, Unicode alphanumeric words only."""
    return re.sub(r"[^\w ]", "", label.lower(), flags=re.UNICODE).strip()
```

- [ ] В `graphify/dedup.py` заменить `_norm()`:

```python
def _norm(label: str) -> str:
    """Lowercase + collapse non-alphanumeric Unicode runs to space."""
    return re.sub(r"[^\w]+", " ", label.lower(), flags=re.UNICODE).strip()
```

- [ ] Запустить целевые тесты:

```powershell
.\.venv\Scripts\python.exe -m pytest tests/test_build.py::test_normalize_id_preserves_cyrillic_identifiers tests/test_dedup.py::test_dedup_norm_preserves_cyrillic_labels -q
```

**Ожидаемо:** тесты проходят.

- [ ] Запустить тесты модулей, которые могут зависеть от ID:

```powershell
.\.venv\Scripts\python.exe -m pytest tests/test_extract.py tests/test_build.py tests/test_dedup.py tests/test_export.py tests/test_serve.py -q
```

**Ожидаемо:** тесты проходят. Если появятся snapshot-like ожидания старых ASCII ID, обновлять только те ожидания, где Unicode-safe ID не меняет смысл существующих ASCII-проектов.

---

## Этап 3. Подключить `tree-sitter-bsl` и обнаружение файлов

### Задача 3.1. Добавить зависимость

**Файлы:**

- Изменить: `pyproject.toml`

- [ ] Добавить зависимость рядом с остальными tree-sitter пакетами:

```toml
    "tree-sitter-bsl",
```

- [ ] Установить зависимости в `.venv`:

```powershell
.\.venv\Scripts\python.exe -m pip install -e .
```

**Ожидаемо:** `tree-sitter-bsl` устанавливается без сборки из исходников на Windows x86-64.

- [ ] Проверить импорт грамматики:

```powershell
.\.venv\Scripts\python.exe -c "import tree_sitter_bsl as tsbsl; from tree_sitter import Language, Parser; parser = Parser(Language(tsbsl.language())); print('ok')"
```

**Ожидаемо:** `ok`.

### Задача 3.2. Добавить `.bsl` в detection

**Файлы:**

- Изменить: `graphify/detect.py`
- Изменить: `tests/test_detect.py`
- Изменить: `tests/test_watch.py`

- [ ] Добавить тест в `tests/test_detect.py`:

```python
from pathlib import Path

from graphify.detect import FileType, classify_file


def test_bsl_is_code_file():
    assert classify_file(Path("ОбщийМодуль.bsl")) == FileType.CODE
```

- [ ] Добавить тест в `tests/test_watch.py`:

```python
def test_watch_includes_bsl_code_extension():
    assert ".bsl" in _WATCHED_EXTENSIONS
```

- [ ] Запустить тесты и убедиться, что они красные:

```powershell
.\.venv\Scripts\python.exe -m pytest tests/test_detect.py::test_bsl_is_code_file tests/test_watch.py::test_watch_includes_bsl_code_extension -q
```

**Ожидаемо:** `.bsl` пока не распознается.

- [ ] Добавить `.bsl` в `CODE_EXTENSIONS` в `graphify/detect.py`.

- [ ] Запустить целевые тесты:

```powershell
.\.venv\Scripts\python.exe -m pytest tests/test_detect.py::test_bsl_is_code_file tests/test_watch.py::test_watch_includes_bsl_code_extension -q
```

**Ожидаемо:** тесты проходят, потому что `_WATCHED_EXTENSIONS` строится из `CODE_EXTENSIONS`.

---

## Этап 4. Реализовать BSL extractor

### Задача 4.1. Создать основной fixture

**Файлы:**

- Создать: `tests/fixtures/sample.bsl`

- [ ] Создать fixture:

```bsl
&НаКлиенте
Процедура РассчитатьИПоказать(Знач Сумма) Экспорт
    Результат = РассчитатьНалог(Сумма);
    Сообщить(Результат);
КонецПроцедуры

&НаСервереБезКонтекста
Функция РассчитатьНалог(Сумма) Экспорт
    Возврат Сумма * 0.2;
КонецФункции

Перем ОбщийКэш Экспорт;

Procedure EnglishProcedure() Export
    EnglishHelper();
EndProcedure

Function EnglishHelper()
    Return 1;
EndFunction
```

### Задача 4.2. Написать падающие тесты для `extract_bsl()`

**Файлы:**

- Изменить: `tests/test_languages.py`

- [ ] Добавить импорт:

```python
from graphify.extract import extract_bsl
```

- [ ] Добавить тесты:

```python
def test_bsl_no_error():
    r = extract_bsl(FIXTURES / "sample.bsl")
    assert "error" not in r


def test_bsl_finds_russian_procedure_and_function():
    r = extract_bsl(FIXTURES / "sample.bsl")
    labels = _labels(r)
    assert "РассчитатьИПоказать()" in labels
    assert "РассчитатьНалог()" in labels


def test_bsl_finds_english_keywords():
    r = extract_bsl(FIXTURES / "sample.bsl")
    labels = _labels(r)
    assert "EnglishProcedure()" in labels
    assert "EnglishHelper()" in labels


def test_bsl_preserves_export_and_execution_context_metadata():
    r = extract_bsl(FIXTURES / "sample.bsl")
    nodes = {n["label"]: n for n in r["nodes"]}
    assert nodes["РассчитатьИПоказать()"]["bsl_export"] is True
    assert nodes["РассчитатьИПоказать()"]["bsl_execution_context"] == "НаКлиенте"
    assert nodes["РассчитатьНалог()"]["bsl_export"] is True
    assert nodes["РассчитатьНалог()"]["bsl_execution_context"] == "НаСервереБезКонтекста"


def test_bsl_emits_local_call_edges():
    r = extract_bsl(FIXTURES / "sample.bsl")
    calls = _calls(r)
    assert ("РассчитатьИПоказать()", "РассчитатьНалог()") in calls
    assert ("EnglishProcedure()", "EnglishHelper()") in calls


def test_bsl_no_dangling_edges():
    r = extract_bsl(FIXTURES / "sample.bsl")
    node_ids = {n["id"] for n in r["nodes"]}
    for e in r["edges"]:
        assert e["source"] in node_ids
        assert e["target"] in node_ids
```

- [ ] Запустить тесты и убедиться, что они красные:

```powershell
.\.venv\Scripts\python.exe -m pytest tests/test_languages.py -q -k bsl
```

**Ожидаемо:** импорт `extract_bsl` отсутствует.

### Задача 4.3. Реализовать helper-функции BSL в `graphify/extract.py`

**Файлы:**

- Изменить: `graphify/extract.py`

- [ ] Добавить рядом с публичными extractors функции для BSL:

```python
_BSL_FUNCTION_TYPES = frozenset({"procedure_definition", "function_definition"})
_BSL_CALL_TYPES = frozenset({"call_expression", "method_call"})


def _bsl_directives_before(node, source: bytes) -> list[str]:
    directives: list[str] = []
    prev = node.prev_named_sibling
    while prev is not None and prev.type == "preprocessor":
        text = _read_text(prev, source).strip()
        if text.startswith("&"):
            directives.append(text.lstrip("&").split("(", 1)[0].strip())
        prev = prev.prev_named_sibling
    directives.reverse()
    return directives


def _bsl_execution_context(directives: list[str]) -> str | None:
    contexts = {
        "НаКлиенте",
        "AtClient",
        "НаСервере",
        "AtServer",
        "НаСервереБезКонтекста",
        "AtServerNoContext",
        "НаКлиентеНаСервереБезКонтекста",
        "AtClientAtServerNoContext",
    }
    for directive in directives:
        if directive in contexts:
            return directive
    return None


def _bsl_node_has_child_type(node, child_type: str) -> bool:
    return any(child.type == child_type for child in node.children)


def _bsl_call_name(node, source: bytes) -> tuple[str | None, bool]:
    if node.type == "method_call":
        name_node = node.child_by_field_name("name")
        return (_read_text(name_node, source), False) if name_node else (None, False)
    if node.type == "call_expression":
        method_call = None
        for child in node.children:
            if child.type == "method_call":
                method_call = child
        if method_call is not None:
            name_node = method_call.child_by_field_name("name")
            if name_node is not None:
                return _read_text(name_node, source), True
    return None, False
```

### Задача 4.4. Реализовать `extract_bsl()`

**Файлы:**

- Изменить: `graphify/extract.py`

- [ ] Добавить функцию:

```python
def extract_bsl(path: Path) -> dict:
    """Extract BSL procedures, functions, exports, execution directives, and calls."""
    try:
        import tree_sitter_bsl as tsbsl
        from tree_sitter import Language, Parser
        language = Language(tsbsl.language())
    except ImportError:
        return {"nodes": [], "edges": [], "error": "tree_sitter_bsl not installed"}
    except Exception as e:
        return {"nodes": [], "edges": [], "error": str(e)}

    try:
        parser = Parser(language)
        source = path.read_bytes()
        tree = parser.parse(source)
    except Exception as e:
        return {"nodes": [], "edges": [], "error": str(e)}

    stem = _file_stem(path)
    str_path = str(path)
    file_nid = _make_id(str_path)
    nodes: list[dict] = []
    edges: list[dict] = []
    raw_calls: list[dict] = []
    seen_ids: set[str] = set()
    label_to_nid: dict[str, str] = {}
    function_bodies: list[tuple[str, object]] = []

    def add_node(nid: str, label: str, line: int, **attrs) -> None:
        if nid in seen_ids:
            return
        seen_ids.add(nid)
        node = {
            "id": nid,
            "label": label,
            "file_type": "code",
            "source_file": str_path,
            "source_location": f"L{line}",
        }
        node.update(attrs)
        nodes.append(node)

    def add_edge(src: str, tgt: str, relation: str, line: int,
                 confidence: str = "EXTRACTED", context: str | None = None) -> None:
        edge = {
            "source": src,
            "target": tgt,
            "relation": relation,
            "confidence": confidence,
            "source_file": str_path,
            "source_location": f"L{line}",
            "weight": 1.0,
        }
        if context:
            edge["context"] = context
        edges.append(edge)

    add_node(file_nid, path.name, 1, symbol_kind="file")

    def walk_definitions(node) -> None:
        if node.type in _BSL_FUNCTION_TYPES:
            name_node = node.child_by_field_name("name")
            if name_node is not None:
                name = _read_text(name_node, source)
                line = node.start_point[0] + 1
                nid = _make_id(stem, name)
                directives = _bsl_directives_before(node, source)
                kind = "procedure" if node.type == "procedure_definition" else "function"
                add_node(
                    nid,
                    f"{name}()",
                    line,
                    symbol_kind=kind,
                    bsl_export=node.child_by_field_name("export") is not None,
                    bsl_async=_bsl_node_has_child_type(node, "ASYNC_KEYWORD"),
                    bsl_annotations=directives,
                    bsl_execution_context=_bsl_execution_context(directives),
                )
                label_to_nid[name.lower()] = nid
                add_edge(file_nid, nid, "contains", line)
                function_bodies.append((nid, node))
            return

        if node.type == "var_definition":
            for child in node.children:
                if child.type == "identifier":
                    name = _read_text(child, source)
                    line = child.start_point[0] + 1
                    nid = _make_id(stem, name)
                    add_node(
                        nid,
                        name,
                        line,
                        symbol_kind="variable",
                        bsl_export=node.child_by_field_name("export") is not None,
                    )
                    add_edge(file_nid, nid, "contains", line)

        for child in node.children:
            walk_definitions(child)

    walk_definitions(tree.root_node)

    seen_call_pairs: set[tuple[str, str]] = set()

    def walk_calls(node, caller_nid: str) -> None:
        if node.type in _BSL_FUNCTION_TYPES and node is not next((body for nid, body in function_bodies if nid == caller_nid), None):
            return

        if node.type in _BSL_CALL_TYPES:
            callee_name, is_member_call = _bsl_call_name(node, source)
            if callee_name:
                tgt_nid = label_to_nid.get(callee_name.lower())
                if tgt_nid and tgt_nid != caller_nid:
                    pair = (caller_nid, tgt_nid)
                    if pair not in seen_call_pairs:
                        seen_call_pairs.add(pair)
                        add_edge(caller_nid, tgt_nid, "calls", node.start_point[0] + 1, context="call")
                elif not tgt_nid:
                    raw_calls.append({
                        "caller_nid": caller_nid,
                        "callee": callee_name,
                        "is_member_call": is_member_call,
                        "source_file": str_path,
                        "source_location": f"L{node.start_point[0] + 1}",
                    })

        for child in node.children:
            walk_calls(child, caller_nid)

    for caller_nid, body_node in function_bodies:
        walk_calls(body_node, caller_nid)

    valid_ids = seen_ids
    clean_edges = [
        edge for edge in edges
        if edge["source"] in valid_ids and edge["target"] in valid_ids
    ]
    return {"nodes": nodes, "edges": clean_edges, "raw_calls": raw_calls}
```

**Замечание для реализации:** условие остановки в `walk_calls()` нужно проверить на реальном AST. Если оно окажется избыточным или нестабильным, заменить его на передачу текущего root-узла тела и пропуск вложенных `procedure_definition`/`function_definition` только когда `node is not root_body`.

- [ ] Запустить BSL-тесты:

```powershell
.\.venv\Scripts\python.exe -m pytest tests/test_languages.py -q -k bsl
```

**Ожидаемо:** тесты проходят или показывают точное расхождение AST, которое исправляется в helper-функциях BSL без изменения общей архитектуры.

---

## Этап 5. Зарегистрировать `.bsl` в dispatcher и общей экстракции

### Задача 5.1. Добавить `extract_bsl` в `_DISPATCH`

**Файлы:**

- Изменить: `graphify/extract.py`
- Изменить: `tests/test_multilang.py`

- [ ] В `_DISPATCH` добавить:

```python
    ".bsl": extract_bsl,
```

- [ ] В `tests/test_multilang.py::test_extract_dispatches_all_languages` добавить `FIXTURES / "sample.bsl"` и проверку `sample.bsl`:

```python
        FIXTURES / "sample.bsl",
```

```python
    assert any("sample.bsl" in f for f in source_files)
```

- [ ] Запустить:

```powershell
.\.venv\Scripts\python.exe -m pytest tests/test_multilang.py::test_extract_dispatches_all_languages -q
```

**Ожидаемо:** dispatch включает BSL fixture.

### Задача 5.2. Проверить cross-file calls для BSL

**Файлы:**

- Создать: `tests/fixtures/bsl_module_a.bsl`
- Создать: `tests/fixtures/bsl_module_b.bsl`
- Изменить: `tests/test_multilang.py`

- [ ] Создать `tests/fixtures/bsl_module_a.bsl`:

```bsl
Процедура ВызватьОбщуюПроцедуру() Экспорт
    ОбщаяПроцедура();
КонецПроцедуры
```

- [ ] Создать `tests/fixtures/bsl_module_b.bsl`:

```bsl
Процедура ОбщаяПроцедура() Экспорт
КонецПроцедуры
```

- [ ] Добавить тест:

```python
def test_bsl_cross_file_call_resolution():
    files = [
        FIXTURES / "bsl_module_a.bsl",
        FIXTURES / "bsl_module_b.bsl",
    ]
    r = extract(files)
    node_by_id = {n["id"]: n["label"] for n in r["nodes"]}
    calls = {
        (node_by_id.get(e["source"], e["source"]), node_by_id.get(e["target"], e["target"]), e["confidence"])
        for e in r["edges"]
        if e["relation"] == "calls"
    }
    assert ("ВызватьОбщуюПроцедуру()", "ОбщаяПроцедура()", "INFERRED") in calls
```

- [ ] Запустить:

```powershell
.\.venv\Scripts\python.exe -m pytest tests/test_multilang.py::test_bsl_cross_file_call_resolution -q
```

**Ожидаемо:** cross-file call резолвится через существующий `raw_calls` pass как `INFERRED`.

---

## Этап 6. Документация

### Задача 6.1. Обновить README

**Файлы:**

- Изменить: `README.md`

- [ ] В таблице поддерживаемых code-файлов добавить `.bsl`.
- [ ] В тексте про tree-sitter languages добавить `BSL`.
- [ ] Не обновлять переводы README на первом проходе, если это не требуется релизной задачей.

- [ ] Проверить, что README не содержит утверждения о поддержке `.os`, если `.os` не добавлен:

```powershell
rg -n "\.os|OneScript|BSL|bsl" README.md docs
```

**Ожидаемо:** `.bsl` упомянут, `.os` не заявлен как поддерживаемый, пока нет отдельного теста.

### Задача 6.2. Обновить ARCHITECTURE

**Файлы:**

- Изменить: `ARCHITECTURE.md`

- [ ] В секции добавления языков указать, что BSL реализован отдельным extractor, потому что:
  - требуется сохранить кириллические identifiers в ID;
  - процедуры/функции BSL содержат executable statements непосредственно в definition node;
  - важны директивы исполнения и `Экспорт`.

---

## Этап 7. Полная проверка

### Задача 7.1. Запустить фокусные тесты

**Файлы:** не менять.

- [ ] Выполнить:

```powershell
.\.venv\Scripts\python.exe -m pytest tests/test_languages.py tests/test_multilang.py tests/test_detect.py tests/test_watch.py tests/test_extract.py tests/test_build.py tests/test_dedup.py -q
```

**Ожидаемо:** все фокусные тесты проходят.

### Задача 7.2. Запустить полный тестовый набор

**Файлы:** не менять.

- [ ] Выполнить:

```powershell
.\.venv\Scripts\python.exe -m pytest tests/ -q
```

**Ожидаемо:** весь test suite проходит. Если есть падения в тестах, не связанных с BSL, сначала проверить dirty worktree и окружение.

### Задача 7.3. Проверить реальный graphify update

**Файлы:** обновятся артефакты `graphify-out`, если команда успешно построит граф.

- [ ] Выполнить проектное правило после изменения code-файлов:

```powershell
.\.venv\Scripts\python.exe -m graphify update .
```

**Ожидаемо:** обновляются `graphify-out/graph.json` и `graphify-out/GRAPH_REPORT.md`. Если команда откажется перезаписать граф из-за shrink safety, повторить только после осознанной проверки:

```powershell
.\.venv\Scripts\python.exe -m graphify update . --force
```

---

## Этап 8. Отложенные расширения после MVP

### `.os` и OneScript

Добавлять `.os` только отдельным изменением:

- fixture `tests/fixtures/sample.os`;
- тест `classify_file(Path("script.os")) == FileType.CODE`;
- dispatch-тест;
- проверка, что `tree-sitter-bsl` реально парсит типичный OneScript без массового `ERROR`.

### Метаданные конфигурации 1С

После стабильного `.bsl` добавить анализ контекста файлов конфигурации:

- `Ext/Module.bsl`;
- `Forms/*/Module.bsl`;
- `Commands/*/Module.bsl`;
- `ObjectModule.bsl`;
- `ManagerModule.bsl`;
- имя объекта из соседних XML-файлов конфигурации.

Этот шаг должен добавлять не только процедуры, но и узлы уровня `Catalog.Номенклатура`, `Document.РеализацияТоваровУслуг`, `CommonModule.ОбщегоНазначения`.

### Более точное разрешение вызовов

После MVP добавить отдельный resolver:

- `ОбщийМодуль.Метод()` -> метод экспортного общего модуля;
- `Справочники.Номенклатура.НайтиПоКоду()` -> platform API, не связывать с пользовательской процедурой;
- `ЭтотОбъект.Провести()` и `Форма.Элементы.*` не резолвить как глобальные вызовы без доказательства.

---

## Критерии готовности MVP

- `.bsl` определяется как code file.
- `watch` реагирует на `.bsl` как на code-only изменение.
- `extract_bsl()` извлекает русские и английские процедуры/функции.
- Русские labels сохраняются в графе.
- Русские node IDs не схлопываются.
- `Экспорт`, `Асинх` и директивы `&НаКлиенте`/`&НаСервере...` сохраняются в node metadata.
- Локальные вызовы внутри файла дают `calls` с `confidence="EXTRACTED"`.
- Cross-file вызовы без import evidence остаются `confidence="INFERRED"`.
- Нет dangling edges в BSL extractor.
- Фокусные и полные тесты проходят.
- После изменения code-файлов выполнен `.\.venv\Scripts\python.exe -m graphify update .`.


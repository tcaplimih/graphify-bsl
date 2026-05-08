# **Архитектурный анализ и руководство по интеграции языка 1С:Предприятие 8.3 в экосистему графов знаний graphify** (Исследование составлено Gemini 3.1 Thinking)

Развитие интеллектуальных агентов программирования в последние годы столкнулось с фундаментальным ограничением, связанным с объемом контекстного окна трансформерных моделей. Несмотря на экспоненциальный рост допустимого количества токенов, стоимость и задержки, связанные с обработкой полных репозиториев, остаются критическим барьером для эффективной разработки. Репозиторий graphify представляет собой инновационное решение этой проблемы, предлагая переход от простого поиска по сходству векторов к структурированному анализу кодовой базы через графы знаний.1 Данный отчет посвящен глубокому исследованию архитектуры graphify, разработке детальной стратегии внедрения поддержки языка 1С:Предприятие 8.3 (BSL) и верификации механизмов обработки кириллических данных, что критически важно для русскоязычного сегмента разработки.

## **Архитектурная парадигма и модульная декомпозиция системы**

Проект graphify спроектирован как локальный инструмент, ориентированный на конфиденциальность и минимизацию затрат на API. В основе его работы лежит функциональный конвейер, который преобразует исходный код, документацию и мультимедийные файлы в унифицированное графовое представление.3 Согласно документации ARCHITECTURE.md, система избегает разделяемого состояния, изолируя все побочные эффекты внутри директории graphify-out/.3

### **Структура конвейера обработки данных**

Архитектура graphify разделена на семь функциональных этапов, каждый из которых инкапсулирован в отдельном модуле. Такая модульность позволяет расширять возможности системы, например, добавлять новые языки программирования, не затрагивая алгоритмы кластеризации или механизмы генерации отчетов.3

| Модуль | Ключевая функция | Входные данные | Выходные данные |
| :---- | :---- | :---- | :---- |
| detect.py | collect\_files(root) | Путь к директории | Список отфильтрованных путей Path |
| extract.py | extract(path) | Путь к файлу | Словарь узлов и ребер |
| build.py | build\_graph(extractions) | Список словарей | Граф NetworkX (nx.Graph) |
| cluster.py | cluster(G) | nx.Graph | Граф с атрибутами сообществ |
| analyze.py | analyze(G) | nx.Graph | Словарь анализа (центральные узлы, связи) |
| report.py | render\_report(G, analysis) | Граф и результаты анализа | Markdown-отчет (GRAPH\_REPORT.md) |
| export.py | export(G, out\_dir) | nx.Graph | JSON, HTML, SVG, MCP, Obsidian |

Этот конвейер обеспечивает однонаправленный поток данных, где каждый этап повышает уровень абстракции — от сырых байтов в файловой системе до высокоуровневых архитектурных инсайтов.3

### **Механизмы извлечения: от AST к семантическим связям**

Одной из наиболее сильных сторон graphify является его трехстадийная модель извлечения данных, которая балансирует между детерминированной точностью и семантической гибкостью.2

Первая стадия — детерминированное извлечение через абстрактные синтаксические деревья (AST). Для этого используется библиотека tree-sitter, которая обеспечивает высокую скорость парсинга и надежность.2 Этот процесс является полностью локальным, что гарантирует сохранность интеллектуальной собственности, так как исходный код никогда не покидает машину пользователя.2 Отношения, полученные на этом этапе (импорты, вызовы функций, наследование), помечаются меткой уверенности EXTRACTED.3

Вторая стадия включает локальную транскрипцию аудио и видео файлов с использованием модели faster-whisper. Это позволяет интегрировать записи митингов и обучающие видео непосредственно в контекст разработки.2

Третья стадия — семантическое извлечение из неструктурированных документов (PDF, Markdown) и изображений. Здесь graphify обращается к настроенному ИИ-бэкенду (Anthropic, OpenAI или Gemini) для выявления концептуальных связей, которые невозможно обнаружить с помощью формальных грамматик.2 Такие связи получают метки INFERRED или AMBIGUOUS, что позволяет ИИ-агентам отличать синтаксические факты от предположений.3

## **Пошаговое руководство по добавлению поддержки языка 1С 8.3 (BSL)**

Интеграция языка 1С:Предприятие 8.3 требует расширения нескольких ключевых компонентов системы. Основная сложность заключается в двуязычности синтаксиса BSL (поддержка как русских, так и английских ключевых слов) и специфических директивах компиляции.9 Согласно инструкциям из ARCHITECTURE.md, процесс добавления нового языка включает следующие шаги.3

### **Шаг 1: Подключение грамматики и зависимостей**

Первым шагом является выбор или разработка парсера tree-sitter для BSL. Существуют зрелые реализации, такие как tree-sitter-bsl, которые поддерживают современные конструкции языка, включая асинхронные вызовы и препроцессорные директивы.9 Необходимо добавить соответствующий пакет в зависимости проекта в файле pyproject.toml.3

### **Шаг 2: Настройка обнаружения файлов**

Система graphify должна научиться распознавать файлы 1С. Для этого необходимо внести изменения в два модуля:

1. В graphify/detect.py нужно добавить расширение .bsl (и, возможно, .os для скриптов OneScript) в набор CODE\_EXTENSIONS. Это позволит функции collect\_files() включать данные файлы в процесс сканирования.10  
2. В graphify/watch.py следует обновить список \_WATCHED\_EXTENSIONS, чтобы изменения в файлах 1С инициировали автоматическую пересборку графа в режиме реального времени.3

| Файл | Переменная / Секция | Действие |
| :---- | :---- | :---- |
| detect.py | CODE\_EXTENSIONS | Добавить '.bsl', '.os' |
| watch.py | \_WATCHED\_EXTENSIONS | Добавить '.bsl', '.os' |
| pyproject.toml | dependencies | Добавить tree-sitter-bsl |

### **Шаг 3: Реализация экстрактора BSL**

Центральным элементом интеграции является создание функции extract\_bsl(path: Path) \-\> dict в модуле graphify/extract.py. Эта функция должна следовать установленному шаблону: инициализация парсера, обход дерева AST и формирование словаря узлов и ребер.3

При обходе AST для 1С 8.3 критически важно корректно обрабатывать следующие узлы:

* **Определения процедур и функций**: Необходимо различать procedure\_definition и function\_definition.9  
* **Экспортные сущности**: Ключевое слово Экспорт (или Export) должно фиксироваться как атрибут узла для последующего анализа области видимости.9  
* **Директивы компиляции**: Такие аннотации, как &НаКлиенте, &НаСервере, &НаСервереБезКонтекста, предоставляют важную информацию о месте выполнения кода и должны извлекаться как метаданные функций.9  
* **Вызовы методов**: Извлечение ребер типа calls между вызывающей и вызываемой процедурами.3

### **Шаг 4: Регистрация в диспетчере и постобработка**

После написания функции extract\_bsl, её необходимо зарегистрировать в основном диспетчере extract() внутри extract.py. Это свяжет расширение файла с конкретной логикой парсинга.3

Кроме того, для 1С характерно использование глобальных модулей и обращение к методам без явного импорта. Чтобы graphify мог разрешить такие связи, экстрактор должен возвращать неразрешенные вызовы как raw\_calls. Встроенный в систему механизм постобработки автоматически сопоставит эти вызовы с глобальной картой меток всех файлов проекта на этапе сборки графа.11

### **Шаг 5: Тестирование и верификация**

Согласно стандартам разработки проекта, необходимо создать файл-фишку (fixture) в директории tests/fixtures/, содержащий репрезентативный код на языке 1С. Затем в tests/test\_languages.py добавляются тесты, проверяющие, что все процедуры, функции и связи между ними корректно извлекаются и соответствуют схеме выходных данных graphify.3

## **Исследование поддержки кириллицы и русского языка**

Для разработчиков на платформе 1С:Предприятие поддержка русского языка в именах переменных, процедур и метаданных является не просто желательной, а обязательной характеристикой. Анализ исходного кода и истории изменений graphify подтверждает глубокую совместимость системы с Unicode и кириллическими символами.

### **Безопасная обработка и декодирование**

Проблема кодировок в многоязычных проектах часто приводит к сбоям при парсинге. В graphify реализован защитный механизм декодирования: все срезы байтов, полученные от tree-sitter, декодируются в UTF-8 с параметром errors="replace".6 Это предотвращает аварийное завершение работы при встрече с некорректными символами, обеспечивая при этом корректное чтение кириллических идентификаторов в подавляющем большинстве случаев.

### **Эволюция функции sanitize\_label**

Критически важным компонентом для поддержки русского языка является функция sanitize\_label в модуле security.py. В ранних версиях (до 0.3.12) эта функция применяла агрессивное HTML-экранирование, что приводило к двойному кодированию и некорректному отображению символов в интерактивном графе.11

Однако в версии 0.3.12 логика была изменена. Теперь sanitize\_label фокусируется исключительно на удалении управляющих символов и ограничении длины метки (до 256 символов), сохраняя при этом исходные Unicode-символы нетронутыми.6 Это означает, что узлы с названиями вроде Справочник\_Номенклатура или процедуры РассчитатьИтоги будут отображаться в графическом интерфейсе корректно, без искажений.13

### **Нормализация для поиска и индексации**

Для эффективного поиска по графу система использует поле norm\_label, которое генерируется на этапе сборки в build.py. Этот механизм использует нормализацию Unicode NFKD, которая декомпозирует составные символы.11 Это обеспечивает стабильную работу поиска в визуальном интерфейсе graph.html и через команды query, даже если в коде или в запросе используются разные формы представления кириллических символов (например, буквы "й" или "ё").11

| Аспект поддержки | Механизм | Статус |
| :---- | :---- | :---- |
| Парсинг идентификаторов | Регулярные выражения с поддержкой \\p{Cyrillic} в грамматике | Полная поддержка 9 |
| Декодирование файлов | UTF-8 с заменой ошибочных байтов | Высокая отказоустойчивость 6 |
| Визуализация меток | Удаление только управляющих символов в sanitize\_label | Корректное отображение кириллицы 4 |
| Поиск по объектам | Unicode NFKD нормализация | Диакритически-инвариантный поиск 11 |

## **Анализ эффективности и топологические особенности**

Применение графов знаний в разработке на 1С дает значительные преимущества за счет специфики архитектуры этой платформы. Конфигурации 1С часто представляют собой монолитные структуры с тысячами взаимосвязанных объектов. Традиционный поиск по коду (grep) или векторный RAG часто теряют контекст из\-за высокой плотности вызовов.

### **Сокращение потребления токенов**

Использование graphify позволяет ИИ-агентам (например, Claude Code) оперировать структурой графа вместо чтения сырого текста файлов. Экспериментальные данные показывают снижение потребления токенов в 71.5 раз при работе с крупными репозиториями.1 Математически этот эффект описывается как отношение объема полного текстового контекста к объему сжатого графового отчета:

![][image1]  
Для типичного проекта 1С, где объем кода может исчисляться сотнями мегабайт, такая экономия делает возможным полноценное использование современных LLM для архитектурного анализа, которые иначе быстро исчерпали бы квоты контекста.2

### **Алгоритм Leiden и архитектурные сообщества**

В модуле cluster.py используется алгоритм Leiden для обнаружения сообществ в графе.1 В отличие от методов, основанных на семантических эмбеддингах, Leiden работает исключительно на основе плотности ребер в топологии графа. В контексте 1С это позволяет автоматически выделять подсистемы (например, "Бухгалтерия", "Склад", "Зарплата"), основываясь на интенсивности вызовов между процедурами внутри этих модулей.1

Интеграция семантических связей из документации (Pass 3\) в этот же граф до начала кластеризации позволяет ИИ обнаруживать скрытые зависимости между бизнес-требованиями и технической реализацией, помечая их как INFERRED связи.1

### **Узлы-концентраторы (God Nodes)**

Модуль analyze.py вычисляет центральность узлов, выявляя так называемые "божественные узлы" (god nodes) — объекты, через которые проходит наибольшее количество связей.1 В проектах на 1С такими узлами часто становятся общие модули с универсальными функциями или центральные справочники. Идентификация этих узлов позволяет разработчику и ИИ-агенту мгновенно понять "центр тяжести" системы и оценить риски при внесении изменений в эти критические компоненты.1

## **Безопасность и локальное использование в корпоративной среде**

Для предприятий, использующих 1С, вопрос безопасности исходного кода является приоритетным. graphify позиционирует себя как инструмент, ориентированный на локальную работу, что подтверждается отсутствием сетевой активности во время анализа графа.2

Основные защитные механизмы включают:

* **Отсутствие выполнения кода**: Система только парсит AST, не используя функции типа eval или exec, что исключает выполнение вредоносного кода из анализируемых файлов.6  
* **Изоляция сетевых запросов**: Все запросы к внешним ресурсам (например, при использовании команды ingest для загрузки документации) проходят через функции safe\_fetch() с жесткими ограничениями по таймаутам и объему данных (до 50 МБ).3  
* **Защита от SSRF**: Встроена проверка URL, блокирующая доступ к локальным интерфейсам и облачным метаданным при попытке загрузки внешних ресурсов.4

Эти меры позволяют безопасно использовать graphify в закрытых контурах разработки, где прямое облачное сканирование всего кода недопустимо по политике безопасности.

## **Заключительные выводы и рекомендации**

Репозиторий graphify представляет собой мощный инструмент для трансформации подхода к анализу кодовой базы ИИ-агентами. Проведенное исследование подтверждает, что архитектура системы, основанная на модульном конвейере и парсинге через tree-sitter, полностью готова к интеграции поддержки языка 1С:Предприятие 8.3.3

Ключевым преимуществом для русскоязычных разработчиков является подтвержденная совместимость с кириллицей, достигнутая за счет перехода к Unicode-безопасной очистке меток и нормализации поисковых индексов.11 Разработчикам, планирующим внедрение 1С в graphify, рекомендуется сфокусироваться на детальном извлечении директив компиляции и экспортных статусов, что позволит графу знаний отражать специфическую для 1С логику распределения контекстов выполнения.9

Внедрение данного инструмента в цикл разработки на платформе 1С не только сократит расходы на API в десятки раз, но и обеспечит новый уровень понимания сложных взаимосвязей внутри конфигураций, минимизируя галлюцинации ИИ-агентов за счет предоставления им достоверной, детерминированно извлеченной архитектурной карты.1

#### **Источники**

1. safishamsi/graphify at aiposthub.com \- GitHub, дата последнего обращения: мая 8, 2026, [https://github.com/safishamsi/graphify?ref=aiposthub.com](https://github.com/safishamsi/graphify?ref=aiposthub.com)  
2. Graphify: Build a Knowledge Graph From Your Entire Codebase — Without Sending Your Code to Anyone | by Mustafa Genc \- GoPenAI, дата последнего обращения: мая 8, 2026, [https://blog.gopenai.com/graphify-build-a-knowledge-graph-from-your-entire-codebase-without-sending-your-code-to-anyone-1b6924474b50](https://blog.gopenai.com/graphify-build-a-knowledge-graph-from-your-entire-codebase-without-sending-your-code-to-anyone-1b6924474b50)  
3. graphify/ARCHITECTURE.md at v4 \- GitHub, дата последнего обращения: мая 8, 2026, [https://github.com/safishamsi/graphify/blob/v4/ARCHITECTURE.md](https://github.com/safishamsi/graphify/blob/v4/ARCHITECTURE.md)  
4. Repository analysis: features, architecture, workflow, and security review · Issue \#63 · safishamsi/graphify \- GitHub, дата последнего обращения: мая 8, 2026, [https://github.com/safishamsi/graphify/issues/63](https://github.com/safishamsi/graphify/issues/63)  
5. Tree-sitter: Introduction, дата последнего обращения: мая 8, 2026, [https://tree-sitter.github.io/](https://tree-sitter.github.io/)  
6. Security \- Overview · safishamsi/graphify · GitHub, дата последнего обращения: мая 8, 2026, [https://github.com/safishamsi/graphify/security](https://github.com/safishamsi/graphify/security)  
7. salvatorecastellitti/graphify-qwen: AI coding assistant skill (Claude Code, Codex, OpenCode, Cursor, Gemini CLI, GitHub Copilot CLI, OpenClaw, Factory Droid, Trae, Google Antigravity). Turn any folder of code, docs, papers, images, or videos into a queryable knowledge graph, дата последнего обращения: мая 8, 2026, [https://github.com/salvatorecastellitti/graphify-qwen](https://github.com/salvatorecastellitti/graphify-qwen)  
8. graphify/README.md at v1 \- GitHub, дата последнего обращения: мая 8, 2026, [https://github.com/safishamsi/graphify/blob/v1/README.md](https://github.com/safishamsi/graphify/blob/v1/README.md)  
9. defendend/Claude-ast-index-search: Cli allows you to index files and greatly speed up Claude/Cursor searches (suitable for any AI agent that can use Bash) \- GitHub, дата последнего обращения: мая 8, 2026, [https://github.com/defendend/Claude-ast-index-search](https://github.com/defendend/Claude-ast-index-search)  
10. post-commit hook silently skips rebuild on .tsx/.jsx and other valid code files (stale CODE\_EXTS gate) · Issue \#222 · safishamsi/graphify \- GitHub, дата последнего обращения: мая 8, 2026, [https://github.com/safishamsi/graphify/issues/222](https://github.com/safishamsi/graphify/issues/222)  
11. CHANGELOG.md \- safishamsi/graphify \- GitHub, дата последнего обращения: мая 8, 2026, [https://github.com/safishamsi/graphify/blob/v4/CHANGELOG.md](https://github.com/safishamsi/graphify/blob/v4/CHANGELOG.md)  
12. Security \- Overview · Rootly-AI-Labs/rootly-graphify-importer \- GitHub, дата последнего обращения: мая 8, 2026, [https://github.com/Rootly-AI-Labs/rootly-graphify-importer/security](https://github.com/Rootly-AI-Labs/rootly-graphify-importer/security)  
13. 'sanitize\_label' double-encodes HTML entities in 'to\_html' JSON output · Issue \#66 · safishamsi/graphify \- GitHub, дата последнего обращения: мая 8, 2026, [https://github.com/safishamsi/graphify/issues/66](https://github.com/safishamsi/graphify/issues/66)  
14. I built a /graphify skill for Claude Code that maps your entire codebase into a knowledge graph, 71x fewer tokens, way less hallucination (32k stars, 250k downloads) : r/ClaudeAI \- Reddit, дата последнего обращения: мая 8, 2026, [https://www.reddit.com/r/ClaudeAI/comments/1ss28rj/i\_built\_a\_graphify\_skill\_for\_claude\_code\_that/](https://www.reddit.com/r/ClaudeAI/comments/1ss28rj/i_built_a_graphify_skill_for_claude_code_that/)  
15. Graphify vs. code-review-graph: Which is better for context-mapping across two very different large codebases? : r/ClaudeCode \- Reddit, дата последнего обращения: мая 8, 2026, [https://www.reddit.com/r/ClaudeCode/comments/1sme1zw/graphify\_vs\_codereviewgraph\_which\_is\_better\_for/](https://www.reddit.com/r/ClaudeCode/comments/1sme1zw/graphify_vs_codereviewgraph_which_is_better_for/)

[image1]: <data:image/png;base64,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>
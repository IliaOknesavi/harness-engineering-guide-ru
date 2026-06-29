---
author: Nexu
---

# Память и контекст

> **Ключевая мысль:** модель знает только то, что находится в её окне контекста. Память — это способ перекинуть мост между тем, что модели *нужно знать*, и тем, что она *может увидеть* в рамках одного вызова API. Сделать это правильно — самая высокорычажная задача в инженерии харнессов (harness).

## Три самостоятельных понятия

Эти термины часто смешивают, но они решают разные задачи:

| Понятие | Область действия | Сохранность | Пример |
|---------|------------------|-------------|--------|
| **Контекст** | Один вызов API | Никакой — пересобирается каждый ход | Системный промпт + инструменты + недавние сообщения + релевантные файлы |
| **Сессия** | Один разговор или одна задача | В памяти процесса, теряется при перезапуске | История сообщений, результаты вызовов инструментов, рабочее состояние |
| **Память** | Между сессиями, бессрочно | Записывается на диск | MEMORY.md, дневные логи, выученные предпочтения |

**Контекст** — это «оперативная память» модели: всё, что собрано в один промпт. **Сессия** — это состояние идущего взаимодействия. **Память** — это то, что переживает завершение сессии.

## Сборка контекста

Каждый ход агентного цикла начинается со сборки контекста. Это задача упаковки с приоритетами: у вас есть фиксированный бюджет токенов и нужно решить, что в него войдёт:

```
Context Window (e.g., 128K tokens)
┌─────────────────────────────────┐
│  System Prompt        (~500)    │  ← Always included, highest priority
│  Tool Schemas         (~2000)   │  ← Active tools only
│  Memory Summary       (~1000)   │  ← Compressed long-term memory
│  Relevant Files       (~5000)   │  ← Task-specific context
│  Conversation History (~varies) │  ← Grows over time, needs pruning
│  [Remaining Budget]             │  ← Available for new content
└─────────────────────────────────┘
```

Когда места становится мало, что именно включать, решает система приоритетов:

```python
class ContextAssembler:
    def __init__(self, max_tokens: int = 128_000):
        self.max_tokens = max_tokens
        self.sections = []  # (priority, name, content)

    def add(self, priority: int, name: str, content: str):
        self.sections.append((priority, name, content))

    def build(self) -> list[dict]:
        # Sort by priority (lower = higher priority)
        self.sections.sort(key=lambda x: x[0])
        messages = []
        used_tokens = 0
        for priority, name, content in self.sections:
            token_count = estimate_tokens(content)
            if used_tokens + token_count > self.max_tokens:
                break  # Budget exceeded — skip lower-priority content
            messages.append({"role": "system", "content": f"[{name}]\n{content}"})
            used_tokens += token_count
        return messages
```

## Управление сессией

Сессия — это граница одного запуска агента. Она хранит:

- **Историю сообщений** — полный разговор, включая вызовы инструментов и их результаты
- **Рабочее состояние** — какие файлы открыты, какие навыки (skill) загружены, текущий прогресс по задаче
- **Черновое пространство (scratch space)** — временные данные, которые агент сгенерировал, но ещё не зафиксировал

Главное проектное решение в работе с сессией — **когда её очищать**. Несколько вариантов:

| Стратегия | Поведение | Сценарий применения |
|-----------|-----------|---------------------|
| **На задачу** | Новая сессия на каждый запрос пользователя | Ассистент без состояния |
| **На разговор** | Сессия живёт через все ходы одного чата | Интерактивное программирование |
| **Постоянная** | Сессия переживает перезапуск процесса | Долгоживущий фоновый агент |

Постоянные сессии требуют сериализации — записи состояния сессии на диск, чтобы его можно было восстановить. Именно здесь сессия и память пересекаются: всё, что стоит сохранять между перезапусками, нужно записывать в файл памяти, а не держать в состоянии сессии.

## Архитектура памяти

Проверенная архитектура памяти состоит из двух уровней:

### Уровень 1: дневные логи

Сырые, хронологические записи того, что произошло. Пишутся прямо во время сессии, без отбора и редактуры:

```markdown
<!-- memory/2026-04-15.md -->
# 2026-04-15

## 14:30 — Refactored auth module
- Moved JWT validation from middleware to dedicated service
- Tests passing (23/23)
- User prefers explicit error messages over error codes

## 16:00 — Deploy to staging
- Used blue-green deployment
- Rollback plan: revert commit abc123
```

### Уровень 2: долговременная память

Отобранное, дистиллированное знание. Обновляется периодически (не каждую сессию):

```markdown
<!-- MEMORY.md -->
# Long-term Memory

## User Preferences
- Prefers explicit error messages over error codes
- Uses pytest, not unittest
- Deploy strategy: blue-green with rollback plan

## Project Knowledge
- Auth module: JWT validation in /src/services/auth.py
- Database: PostgreSQL 15, migrations in /db/migrations/
- CI: GitHub Actions, ~3min build time

## Lessons Learned
- Always run tests before committing (broke build on 4/10)
- User dislikes verbose output — keep summaries under 5 lines
```

Ключевая мысль: дневные логи дёшевы в записи (достаточно дописать в конец). Долговременная память требует суждения (что вообще стоит сохранять?). Промышленные харнессы пишут дневные логи автоматически, а MEMORY.md курируют периодически — либо по расписанию, либо когда агент обнаруживает значимые выводы.

## Цикл чтения/записи памяти

```python
def session_startup(memory_dir: str) -> str:
    """Read memory at session start."""
    sections = []
    # Always read long-term memory
    memory_path = os.path.join(memory_dir, "MEMORY.md")
    if os.path.exists(memory_path):
        sections.append(open(memory_path).read())
    # Read recent daily logs (today + yesterday)
    for days_ago in [0, 1]:
        date = (datetime.now() - timedelta(days=days_ago)).strftime("%Y-%m-%d")
        daily_path = os.path.join(memory_dir, f"memory/{date}.md")
        if os.path.exists(daily_path):
            sections.append(open(daily_path).read())
    return "\n---\n".join(sections)
```

## Паттерн AGENTS.md

Близкий, но отдельный файл — AGENTS.md: текстовый файл, который определяет, как агент должен *вести себя* (а не то, что он *помнит*). Положите его в любой каталог, и совместимый харнесс прочитает его автоматически:

```markdown
<!-- AGENTS.md -->
# Behavior

- You are a Python backend engineer
- Use pytest for all tests
- Follow Google style docstrings
- Never modify files in /config/ without asking

# Tools

- Prefer `ruff` over `pylint` for linting
- Use `uv` for package management
```

AGENTS.md **декларативен** (что делать), а MEMORY.md **опытен/эмпиричен** (что произошло). Оба внедряются в контекст при старте сессии, но служат разным целям.

## Типичные ошибки

- **Считать контекст безграничным** — даже 128K токенов заполняются быстро схемами инструментов, содержимым файлов и историей разговора. Планируйте бюджет токенов явно.
- **Никогда не подрезать историю сессии** — разговор на 50 ходов накапливает избыточный материал. Сжимайте или резюмируйте более старые ходы, чтобы вернуть себе место.
- **Писать в память слишком рьяно** — не каждый ход порождает знание, которое стоит сохранять. Избыточная запись создаёт шум, разбавляющий полезную информацию.
- **Забыть прочитать память на старте** — агент без чтения памяти фактически страдает амнезией. Это самая частая ошибка конфигурации.

## Дополнительное чтение

- [Letta: MemGPT and the Future of Agent Memory](https://www.letta.com/blog/memgpt) — управление памятью агентов по образу операционной системы
- [Anthropic: Building Effective Agents](https://www.anthropic.com/research/building-effective-agents) — паттерны памяти в продакшене

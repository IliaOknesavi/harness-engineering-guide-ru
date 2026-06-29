---
author: Nexu
---

# Инженерия контекста

> **Ключевая мысль:** Модель не знает того, что вы ей не сообщили. Инженерия контекста (context engineering) — это дисциплина принятия решений о том, что попадёт в окно контекста, в каком порядке и что будет вырезано, когда место закончится. Это работа с наибольшим рычагом во всей инженерии харнессов (harness engineering) — она влияет на результат сильнее, чем выбор модели, тюнинг промптов или дизайн инструментов.

## Проблема

Окно контекста на 128K токенов звучит огромным — пока вы не начнёте его заполнять. Один крупный файл может съесть 10K токенов. Двадцать схем инструментов — ещё 3K. История диалога растёт линейно с каждым ходом. Уже в пределах десятка ходов сложной задачи по программированию вам приходится принимать непростые решения о том, что оставить, а что выбросить.

Инженерия контекста — это искусство таких решений. У неё три опоры: **сборка** (assembly — что попадает внутрь), **сжатие** (compression — что ужимается) и **бюджетирование** (budgeting — как вы распределяете ёмкость).

## Система приоритетов при сборке контекста

Не всякий контекст одинаково ценен. Система приоритетов гарантирует, что самая критичная информация уцелеет, когда места в обрез:

| Приоритет | Категория | Типичный объём токенов | Примечания |
|----------|----------|---------------|-------|
| 0 (наивысший) | Системный промпт | 300–800 | Идентичность, правила поведения, ограничения безопасности |
| 1 | Схемы активных инструментов | 1,000–3,000 | Только загруженные навыки (skills), а не все инструменты |
| 2 | Инструкция к задаче | 200–1,000 | Текущий запрос пользователя + любые закреплённые цели |
| 3 | Сводка памяти | 500–2,000 | Сжатый MEMORY.md + дневной лог за сегодня |
| 4 | Внедрённые файлы | 2,000–20,000 | AGENTS.md, SKILL.md, релевантные исходные файлы |
| 5 | Недавний диалог | 5,000–50,000 | Последние N ходов сообщений + результаты инструментов |
| 6 (низший) | Более старый диалог | остаток | Ранние ходы, первые кандидаты на сжатие или выброс |

Сборщик идёт по этому списку сверху вниз, упаковывая содержимое, пока не исчерпается бюджет. Содержимое с более низким приоритетом усекается или исключается целиком.

```python
import tiktoken

encoder = tiktoken.encoding_for_model("gpt-4o")

def estimate_tokens(text: str) -> int:
    """Fast token estimation using tiktoken."""
    return len(encoder.encode(text))

class ContextAssembler:
    """Assemble context with priority-based token budgeting."""

    def __init__(self, max_tokens: int = 128_000, reserve: int = 4_096):
        self.max_tokens = max_tokens
        self.reserve = reserve  # Leave room for the model's response
        self.budget = max_tokens - reserve
        self.sections: list[tuple[int, str, str]] = []

    def add(self, priority: int, name: str, content: str):
        """Add a section. Lower priority number = higher importance."""
        self.sections.append((priority, name, content))

    def build(self) -> list[dict]:
        """Pack sections into messages within the token budget."""
        self.sections.sort(key=lambda s: s[0])
        messages = []
        used = 0

        for priority, name, content in self.sections:
            tokens = estimate_tokens(content)
            if used + tokens <= self.budget:
                messages.append({
                    "role": "system",
                    "content": f"[{name}]\n{content}",
                })
                used += tokens
            elif priority <= 2:
                # Critical sections get truncated rather than dropped
                remaining = self.budget - used
                truncated = self._truncate_to_tokens(content, remaining)
                if truncated:
                    messages.append({
                        "role": "system",
                        "content": f"[{name} (truncated)]\n{truncated}",
                    })
                    used += estimate_tokens(truncated)
            # Priority > 2: silently dropped when over budget

        return messages

    def _truncate_to_tokens(self, text: str, max_tokens: int) -> str:
        """Truncate text to fit within a token limit."""
        tokens = encoder.encode(text)
        if len(tokens) <= max_tokens:
            return text
        return encoder.decode(tokens[:max_tokens]) + "\n[...truncated]"
```

Параметр `reserve` легко упустить из виду, но он критичен — нужно оставить запас под ответ модели. Если упаковать контекст на 100%, модели некуда будет ответить.

## Сжатие контекста: три линии обороны

По мере развития сессии необработанная история диалога растёт безгранично. Три приёма не дают ей поглотить всё окно контекста:

### Линия 1: автоматическое затухание

Старые сообщения естественным образом теряют релевантность. Простая стратегия затухания (decay) отбрасывает сообщения за пределами фиксированного окна, оставляя лишь N последних ходов:

```python
def apply_decay(messages: list[dict], max_turns: int = 20) -> list[dict]:
    """Keep the system prompt and the last max_turns exchanges."""
    system = [m for m in messages if m["role"] == "system"]
    conversation = [m for m in messages if m["role"] != "system"]
    # Each "turn" is roughly a user + assistant + tool cycle
    if len(conversation) > max_turns * 3:
        conversation = conversation[-(max_turns * 3):]
    return system + conversation
```

### Линия 2: сжатие по порогу

Когда общее число токенов переходит порог (например, 70% бюджета), старые ходы диалога сжимаются в сводку, а недавние ходы сохраняются дословно:

```python
def threshold_compress(
    messages: list[dict],
    budget: int,
    threshold: float = 0.7,
    keep_recent: int = 10,
) -> list[dict]:
    """Compress older messages when token usage exceeds threshold."""
    total = sum(estimate_tokens(m["content"]) for m in messages)
    if total < budget * threshold:
        return messages  # Under threshold, no compression needed

    system = [m for m in messages if m["role"] == "system"]
    conversation = [m for m in messages if m["role"] != "system"]

    old = conversation[:-keep_recent]
    recent = conversation[-keep_recent:]

    summary = summarize_with_llm(old)  # Use a fast, cheap model
    compressed = system + [{
        "role": "system",
        "content": f"[Conversation summary]\n{summary}",
    }] + recent

    return compressed
```

### Линия 3: активное реферирование

Для крайне долгих задач полезно периодически извлекать ключевые факты и решения в живой документ-сводку. Это происходит не автоматически — харнесс явно просит модель сформировать контрольную точку (checkpoint):

```python
SUMMARIZE_PROMPT = """Summarize the key decisions, findings, and current state 
from this conversation. Include: files modified, tests run, errors encountered, 
and the current plan. Be concise — under 500 words."""

def active_summarize(messages: list[dict]) -> str:
    """Ask the model to produce a checkpoint summary."""
    response = llm.chat(
        messages=messages + [{"role": "user", "content": SUMMARIZE_PROMPT}],
        max_tokens=1024,
    )
    return response.choices[0].message.content
```

## Бюджетирование токенов на практике

Реальная арифметика токенов для окна контекста на 128K:

```
Total capacity:              128,000 tokens
Response reserve:             -4,096
System prompt:                  -500
Tool schemas (12 tools):      -2,400
MEMORY.md:                    -1,200
AGENTS.md:                      -800
─────────────────────────────────────
Available for conversation:  119,004 tokens

At ~3 tokens/word, that's ~39,600 words of conversation.
A 50-turn coding session with tool results: ~60,000 tokens.
→ You'll hit the budget around turn 35 without compression.
```

Вывод: для любой нетривиальной сессии сжатие — не опция, а необходимость.

## Паттерны внедрения контекста

Контекст приходит не только из истории диалога. Пять распространённых паттернов внедрения (injection):

| Паттерн | Когда | Пример |
|---------|------|---------|
| **Внедрение файлов** | При старте сессии | Загрузка AGENTS.md, MEMORY.md, релевантных исходных файлов |
| **Внедрение памяти** | При старте сессии | Сжатая долгосрочная память + недавние дневные логи |
| **Внедрение результатов инструментов** | В ходе цикла | Дописывание выводов инструментов как сообщений с ролью tool |
| **Внедрение навыков** | По требованию | Загрузка SKILL.md при активации навыка (skill) |
| **Внедрение результатов поиска** | На каждый запрос | Результаты RAG из векторного хранилища |

У каждой точки внедрения есть своя цена. Исходный файл на 200 строк — это ~800 токенов. Внедрить десяток файлов «на всякий случай» обойдётся в 8K токенов ещё до начала диалога. Действуйте осознанно: внедряйте то, что нужно, а не то, что *может* понадобиться.

## Реализация скользящего окна

Скользящее окно (sliding window) сохраняет последние ходы в неприкосновенности и сжимает всё, что находится до границы окна. Это самая практичная стратегия для продакшен-харнессов:

```python
class SlidingWindowContext:
    """Maintain a sliding window over conversation history."""

    def __init__(self, window_size: int = 15, max_tokens: int = 128_000):
        self.window_size = window_size
        self.max_tokens = max_tokens
        self.summary = ""
        self.messages: list[dict] = []

    def add(self, message: dict):
        self.messages.append(message)
        conversation = [m for m in self.messages if m["role"] != "system"]
        if len(conversation) > self.window_size * 3:
            self._compress()

    def _compress(self):
        """Move older messages into a rolling summary."""
        conversation = [m for m in self.messages if m["role"] != "system"]
        system = [m for m in self.messages if m["role"] == "system"]

        old = conversation[:-(self.window_size * 3)]
        recent = conversation[-(self.window_size * 3):]

        new_summary = summarize_with_llm(
            [{"role": "system", "content": self.summary}] + old
        )
        self.summary = new_summary
        self.messages = system + recent

    def get_messages(self) -> list[dict]:
        """Return context-ready message list."""
        result = [m for m in self.messages if m["role"] == "system"]
        if self.summary:
            result.append({
                "role": "system",
                "content": f"[Conversation history summary]\n{self.summary}",
            })
        result.extend(m for m in self.messages if m["role"] != "system")
        return result
```

## Типичные ошибки

- **Считать весь контекст равноприоритетным** — Системный промпт и инструкции к задаче должны уцелеть; старый диалог можно сжать. Без приоритетов вы либо тратите место на устаревшие сообщения, либо выбрасываете критичные инструкции.
- **Сжимать слишком агрессивно** — Реферирование теряет информацию. Если сжать результат инструмента, в котором был путь к файлу, нужный модели позже, она этот путь нагаллюцинирует. Недавние ходы держите дословно.
- **Игнорировать подсчёт токенов** — Прикидка «на глаз вроде коротко» быстро ломается. Для бюджетирования используйте настоящий подсчёт токенов (tiktoken, токенизаторы под конкретную модель).
- **Одноразовая сборка контекста** — Если собрать контекст один раз в начале сессии и больше его не обновлять, после первого же вызова инструмента модель работает с устаревшей информацией. Пересобирайте контекст на каждом ходу.

## Что почитать дальше

- [Karpathy: "Context Engineering is the New Prompt Engineering"](https://x.com/karpathy/status/1937902263428948034) — Почему сборка контекста важнее трюков с промптами
- [Letta: MemGPT and the Future of Agent Memory](https://www.letta.com/blog/memgpt) — Управление памятью в духе ОС с виртуальным контекстом
- [OpenAI: Managing Tokens](https://platform.openai.com/docs/guides/text-generation#managing-tokens) — Основы подсчёта токенов

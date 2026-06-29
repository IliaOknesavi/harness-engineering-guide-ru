---
author: Nexu
---

# Система инструментов

> **Ключевая мысль:** инструменты — это руки агента. Модель рассуждает, инструменты действуют. Но именно устройство системы инструментов — как инструменты регистрируются, описываются, диспетчеризуются и управляются — влияет на качество агента сильнее, чем сама модель.

## Что такое инструмент?

Инструмент — это функция, которую модель может вызвать по имени со структурированными аргументами. Модель видит **схему** (имя, описание, типы параметров), а харнесс (harness) отвечает за **исполнение** (собственно вызов функции и возврат результата).

```python
# What the model sees (tool schema)
{
    "name": "read_file",
    "description": "Read the contents of a file at the given path",
    "parameters": {
        "type": "object",
        "properties": {
            "path": {"type": "string", "description": "File path to read"}
        },
        "required": ["path"]
    }
}

# What the harness executes (tool implementation)
def read_file(path: str) -> str:
    with open(path, 'r') as f:
        return f.read()
```

Модель никогда не видит и не исполняет реализацию. Ей известна только схема. Это разделение фундаментально: оно означает, что вы можете менять работу инструмента, не меняя поведение модели, и можете ограничивать то, что инструмент делает, так что модель об этом не узнает.

## Реестр инструментов

Реестр инструментов (tool registry) — это компонент харнесса, который сопоставляет имена инструментов с их схемами и реализациями:

```python
class ToolRegistry:
    def __init__(self):
        self._tools: dict[str, Tool] = {}

    def register(self, name: str, schema: dict, handler: Callable):
        self._tools[name] = Tool(name=name, schema=schema, handler=handler)

    def get_schemas(self) -> list[dict]:
        """Return schemas for the LLM API call."""
        return [t.schema for t in self._tools.values()]

    def dispatch(self, name: str, arguments: dict) -> str:
        """Execute a tool call and return the result as a string."""
        tool = self._tools.get(name)
        if not tool:
            return f"Error: Unknown tool '{name}'"
        try:
            result = tool.handler(**arguments)
            return str(result)
        except Exception as e:
            return f"Error: {type(e).__name__}: {e}"
```

Обратите внимание, что `dispatch` всегда возвращает строку — даже при ошибках. Это сделано намеренно: модели нужно видеть сообщения об ошибках, чтобы скорректировать подход, а не молча падать.

## Статические и динамические инструменты

**Статические инструменты** загружаются при старте и доступны всегда. Это работает для небольших наборов инструментов (5–15 штук), но не масштабируется: 100 инструментов означают 100 схем в каждом вызове API — они расходуют токены и сбивают модель с толку.

**Динамические инструменты** (их также называют **загрузкой навыков**, skill loading) решают эту проблему: модели показывают меню доступных категорий инструментов и подгружают конкретные инструменты только по запросу:

```python
# Instead of loading all 100 tools, show a menu
SKILL_MENU = """
Available skills (use load_skill to activate):
- file_ops: Read, write, search files
- git: Git operations (status, diff, commit, push)
- web: HTTP requests, web search
- database: SQL queries, schema inspection
"""

# The model calls load_skill("git") and then gets git-specific tools
def load_skill(skill_name: str) -> str:
    tools = skill_registry.load(skill_name)
    active_tools.extend(tools)
    return f"Loaded {len(tools)} tools: {[t.name for t in tools]}"
```

Экономия токенов получается значительной. Меню навыков может стоить 200 токенов, тогда как загрузка всех инструментов сразу обойдётся в 5000+.

## Качество описаний инструментов

Способность модели правильно пользоваться инструментами почти полностью зависит от качества их описаний. Расплывчатое описание ведёт к неверному применению, точное — направляет к корректному поведению:

```python
# Bad — the model will guess at behavior
{"name": "search", "description": "Search for things"}

# Good — unambiguous, includes format and constraints
{
    "name": "search_files",
    "description": "Search for files matching a glob pattern in the workspace. "
                   "Returns a list of relative file paths, one per line. "
                   "Max 100 results. Use '**/*.py' for recursive Python file search.",
    "parameters": {
        "type": "object",
        "properties": {
            "pattern": {
                "type": "string",
                "description": "Glob pattern (e.g., '*.md', 'src/**/*.ts')"
            }
        },
        "required": ["pattern"]
    }
}
```

Ключевые принципы для описаний инструментов:
- Описывайте, что инструмент **делает**, а не что он **есть**
- Указывайте **формат вывода** (JSON, простой текст, по одной записи на строку)
- Включайте **ограничения** (максимум результатов, лимиты на размер файла)
- Добавляйте **примеры** для неочевидных параметров

## Паттерны композиции инструментов

Сложные возможности агента чаще возникают из композиции простых инструментов, а не из построения сложных:

| Паттерн | Пример |
|---------|---------|
| **Последовательный** | `read_file` → `edit_file` → `run_tests` |
| **Веерный (fan-out)** | прочитать 5 файлов параллельно, затем синтезировать |
| **Условный** | `list_files` → решить, какие файлы передать в `read_file` |
| **Итеративный** | `run_tests` → `edit_file` → `run_tests` (пока не пройдут) |

Харнессу не нужно реализовывать эти паттерны — модель находит их сама в ходе агентного цикла. Ваша задача — предоставить правильные атомарные инструменты и дать модели их компоновать.

## MCP: Model Context Protocol

[MCP](https://modelcontextprotocol.io/) — это открытый стандарт для предоставления инструментов агентам через транспортный уровень (stdio, HTTP SSE). Вместо того чтобы зашивать инструменты в харнесс, MCP позволяет подключаться к внешним серверам инструментов:

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/workspace"]
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"]
    }
  }
}
```

MCP важен тем, что он развязывает реализацию инструментов и харнесс. Инструмент, написанный для одного харнесса, работает в любом MCP-совместимом харнессе — Claude Desktop, OpenClaw, Cursor и других.

## Типичные ошибки

- **Слишком много инструментов сразу** — более ~20 активных инструментов снижают качество работы модели. Используйте динамическую загрузку.
- **Молчаливые сбои** — инструменты, возвращающие пустые строки при ошибке, оставляют модель в догадках. Всегда возвращайте явные сообщения об ошибках.
- **Пропущенные результаты инструментов** — если вы забудете добавить результат инструмента в историю сообщений, вызов API завершится ошибкой. У каждого вызова инструмента должен быть соответствующий результат.
- **Несогласованные типы возврата** — если `read_file` иногда возвращает содержимое, а иногда словарь с ошибкой, модель не сможет надёжно разобрать вывод. Стандартизируйте формат результата.

## Дополнительное чтение

- [Anthropic: Tool Use Guide](https://docs.anthropic.com/en/docs/build-with-claude/tool-use) — продакшен-паттерны использования инструментов
- [Model Context Protocol](https://modelcontextprotocol.io/) — открытый стандарт для инструментов агентов

---
author: Nexu
---

# Ваш первый харнесс

> **Ключевая мысль:** харнесс (harness) — это всего лишь цикл: вызвать модель, выполнить запрошенные вызовы инструментов, вернуть результаты обратно, повторить. Рабочую версию можно написать менее чем за 50 строк на Python. Понимание этого цикла снимает завесу тайны с любого агентного фреймворка, который вам встретится.

Большинство руководств по агентам начинаются с фреймворка — LangChain, CrewAI, AutoGen. Но фреймворки прячут механику. Сборка харнесса с нуля показывает, что именно происходит: агентный цикл, сборку контекста и процесс принятия решений моделью. Стоит это понять — и любой фреймворк становится прозрачным.

## Полноценный харнесс

Перед вами полностью рабочий харнесс с двумя инструментами (чтение файла и запись файла). Скопируйте, вставьте и запустите.

### Предварительные требования

```bash
pip install openai
export OPENAI_API_KEY="sk-your-key-here"
```

### Код

```python
#!/usr/bin/env python3
"""A complete agent harness in ~50 lines. Run: python harness.py"""

import json
import os
from openai import OpenAI

client = OpenAI()
MODEL = "gpt-4o-mini"  # Cheap and fast for learning
MAX_TURNS = 15

# --- System prompt ---
SYSTEM = """You are a helpful file assistant. You can read and write files.
When asked to work with files, use the tools provided.
Always confirm what you did after completing a task."""

# --- Tool definitions ---
TOOLS = [
    {
        "type": "function",
        "function": {
            "name": "read_file",
            "description": "Read the contents of a file at the given path",
            "parameters": {
                "type": "object",
                "properties": {
                    "path": {"type": "string", "description": "Path to the file"}
                },
                "required": ["path"]
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "write_file",
            "description": "Write content to a file (creates or overwrites)",
            "parameters": {
                "type": "object",
                "properties": {
                    "path": {"type": "string", "description": "Path to the file"},
                    "content": {"type": "string", "description": "Content to write"}
                },
                "required": ["path", "content"]
            }
        }
    }
]

# --- Tool execution ---
def execute_tool(name: str, args: dict) -> str:
    try:
        if name == "read_file":
            with open(args["path"], "r") as f:
                return f.read()
        elif name == "write_file":
            os.makedirs(os.path.dirname(args["path"]) or ".", exist_ok=True)
            with open(args["path"], "w") as f:
                f.write(args["content"])
            return f"Wrote {len(args['content'])} chars to {args['path']}"
        else:
            return f"Error: Unknown tool '{name}'"
    except Exception as e:
        return f"Error: {e}"

# --- The tool loop ---
def run(user_message: str) -> str:
    messages = [
        {"role": "system", "content": SYSTEM},
        {"role": "user", "content": user_message}
    ]

    for turn in range(MAX_TURNS):
        response = client.chat.completions.create(
            model=MODEL, messages=messages, tools=TOOLS
        )
        msg = response.choices[0].message
        messages.append(msg)

        # No tool calls → model is done
        if not msg.tool_calls:
            return msg.content

        # Execute each tool call
        for tc in msg.tool_calls:
            args = json.loads(tc.function.arguments)
            print(f"  🔧 {tc.function.name}({args})")
            result = execute_tool(tc.function.name, args)
            messages.append({
                "role": "tool",
                "tool_call_id": tc.id,
                "content": result
            })

    return "Max turns reached."

# --- Main ---
if __name__ == "__main__":
    print("🤖 File Agent (type 'quit' to exit)")
    while True:
        user_input = input("\nYou: ").strip()
        if user_input.lower() in ("quit", "exit"):
            break
        response = run(user_input)
        print(f"\nAgent: {response}")
```

### Попробуйте

```bash
python harness.py
```

```
🤖 File Agent (type 'quit' to exit)

You: Create a file called hello.txt with a haiku about programming

  🔧 write_file({'path': 'hello.txt', 'content': 'Semicolons fall\nLike rain upon the server\nCompile error: none'})

Agent: I've created hello.txt with a programming haiku!

You: Read it back to me

  🔧 read_file({'path': 'hello.txt'})

Agent: Here's the content of hello.txt:
"Semicolons fall / Like rain upon the server / Compile error: none"
```

## Анатомия харнесса

Весь харнесс состоит из четырёх компонентов:

```
┌────────────────────────────────┐
│         System Prompt          │  ← Who the agent is
├────────────────────────────────┤
│        Tool Definitions        │  ← What it can do (JSON schema)
├────────────────────────────────┤
│        Tool Execution          │  ← How tools actually run
├────────────────────────────────┤
│          Tool Loop             │  ← The cycle: think → act → observe
└────────────────────────────────┘
```

**Системный промпт (system prompt)**: задаёт характер агента и его ограничения. Это самая дешёвая и при этом самая «рычажная» часть — изменение одного предложения здесь способно полностью поменять поведение.

**Определения инструментов (tool definitions)**: JSON-схемы, которые модель читает, чтобы понять, какие инструменты существуют. Модель никогда не видит ваш Python-код — только описания и схемы параметров.

**Выполнение инструментов (tool execution)**: ваш код, который собственно совершает действия. Модель выдаёт структурированный JSON; вы его разбираете и делаете настоящую работу.

**Цикл инструментов (tool loop)**: оркестратор. Вызвать модель, проверить наличие вызовов инструментов, выполнить их, вернуть результаты обратно. Повторять, пока модель не ответит обычным текстом.

## Добавляем третий инструмент

Хотите добавить выполнение shell-команд? Просто добавьте определение инструмента и обработчик:

```python
# Add to TOOLS list:
{
    "type": "function",
    "function": {
        "name": "run_shell",
        "description": "Run a shell command and return stdout/stderr",
        "parameters": {
            "type": "object",
            "properties": {
                "command": {"type": "string", "description": "Shell command to run"}
            },
            "required": ["command"]
        }
    }
}

# Add to execute_tool():
elif name == "run_shell":
    import subprocess
    r = subprocess.run(args["command"], shell=True, capture_output=True, text=True, timeout=30)
    return r.stdout + r.stderr
```

Цикл при этом не меняется. Модель автоматически обнаруживает и начинает использовать новый инструмент.

## Замена модели

Харнесс не привязан к конкретной модели. Чтобы переключиться на Claude от Anthropic, поменяйте клиента:

```python
from anthropic import Anthropic

client = Anthropic()

response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=4096,
    system=SYSTEM,
    messages=messages,
    tools=[{
        "name": t["function"]["name"],
        "description": t["function"]["description"],
        "input_schema": t["function"]["parameters"]
    } for t in TOOLS]
)

# Parse tool calls from response.content blocks
for block in response.content:
    if block.type == "tool_use":
        result = execute_tool(block.name, block.input)
```

Тот же цикл. Те же инструменты. Другая модель.

## Чего здесь не хватает (и что будет дальше)

Этот харнесс работает, но продакшен-агентам нужно больше:

| Возможность | Этот харнесс | Продакшен-харнесс |
|---------|-------------|-------------------|
| Память | Нет (без состояния) | MEMORY.md + ежедневные логи |
| Управление контекстом | Вся история целиком | Окно по приоритетам |
| Восстановление после ошибок | Базовый try/catch | Повтор (retry) + эскалация |
| Безопасность | Нет | Изолированное выполнение в песочнице |
| Загрузка инструментов | Все сразу | Навыки (skills) по запросу |

Каждой из этих тем посвящены отдельные разделы остальной части руководства.

## Типичные подводные камни

- **Забыть добавить сообщение ассистента** — если не добавить `msg` в `messages` до результатов инструментов, модель теряет нить того, что сама запросила. Всегда сначала добавляйте полный ответ ассистента.
- **Неправильное приведение результатов инструментов к строке** — результаты инструментов должны быть строками. Если ваш инструмент возвращает словарь, прогоните его через `json.dumps()`. Возврат «сырого» Python-объекта приведёт к падению.
- **Отсутствие ограничения на число итераций** — без `MAX_TURNS` запутавшаяся модель может крутиться вечно, сжигая токены. Всегда ставьте предел.

## Что почитать дальше

- [OpenAI Function Calling Guide](https://platform.openai.com/docs/guides/function-calling) — официальная документация по определениям инструментов
- [Anthropic Tool Use Guide](https://docs.anthropic.com/en/docs/build-with-claude/tool-use) — её аналог для Claude
- [ReAct Paper](https://arxiv.org/abs/2210.03629) — академический фундамент циклов с инструментами

---
author: Nexu
---

# Харнесс против фреймворка

> **Ключевая мысль:** Фреймворк тащит за собой сотни зависимостей и слои абстракций ради задачи, которую можно решить в 50 строк на Python. Но и писать оркестрацию из множества агентов с нуля, когда CrewAI уже её решает, — значит впустую потратить недели. Суть в том, чтобы подбирать инструмент под задачу, а не по умолчанию хвататься за самый популярный вариант.

Харнесс (harness — обвязка, runtime-обёртка) — это код, который вы пишете с нуля, чтобы обернуть модель инструментами, памятью и контекстом. **Фреймворк** — это библиотека, дающая абстракции для построения агентов: LangChain, CrewAI, AutoGen и другие. Выбор между ними — не вопрос о том, что «лучше», а вопрос о том, когда каждый из них оправдывает себя.

## Дерево решений

```
Need an agent? 
│
├── Is it a single-model loop with < 5 tools?
│   └── YES → Write a raw harness (50-200 lines)
│
├── Do you need multi-agent orchestration out of the box?
│   └── YES → Consider CrewAI or AutoGen
│
├── Do you need complex RAG pipelines with vector stores?
│   └── YES → Consider LangChain
│
├── Is this a production product where you need full control?
│   └── YES → Write a raw harness (own every line)
│
├── Are you prototyping / exploring quickly?
│   └── YES → Framework is fine, expect to rewrite later
│
└── Do you need to understand what's actually happening?
    └── YES → Write a raw harness first, then decide
```

## Одна и та же задача: три способа

**Задача**: прочитать CSV-файл, проанализировать его и записать сводку в Markdown-файл.

### Голый харнесс (~60 строк)

```python
import json, csv, io
from openai import OpenAI

client = OpenAI()

TOOLS = [
    {
        "type": "function",
        "function": {
            "name": "read_file",
            "description": "Read a file's contents",
            "parameters": {
                "type": "object",
                "properties": {"path": {"type": "string"}},
                "required": ["path"]
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "write_file",
            "description": "Write content to a file",
            "parameters": {
                "type": "object",
                "properties": {
                    "path": {"type": "string"},
                    "content": {"type": "string"}
                },
                "required": ["path", "content"]
            }
        }
    }
]

def execute(name, args):
    if name == "read_file":
        return open(args["path"]).read()
    elif name == "write_file":
        open(args["path"], "w").write(args["content"])
        return f"Written to {args['path']}"

def run(task):
    messages = [
        {"role": "system", "content": "You analyze data files and write reports."},
        {"role": "user", "content": task}
    ]
    for _ in range(10):
        resp = client.chat.completions.create(
            model="gpt-4o-mini", messages=messages, tools=TOOLS
        )
        msg = resp.choices[0].message
        messages.append(msg)
        if not msg.tool_calls:
            return msg.content
        for tc in msg.tool_calls:
            result = execute(tc.function.name, json.loads(tc.function.arguments))
            messages.append({"role": "tool", "tool_call_id": tc.id, "content": result})
    return "Done"

run("Read data.csv, analyze the trends, and write a summary to report.md")
```

**Зависимости**: `openai` (1 пакет)
**Строк кода**: ~60
**Под вашим контролем**: всё

### LangChain (~40 строк, но…)

```python
from langchain_openai import ChatOpenAI
from langchain.agents import AgentExecutor, create_openai_tools_agent
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder
from langchain.tools import tool

@tool
def read_file(path: str) -> str:
    """Read a file's contents"""
    return open(path).read()

@tool
def write_file(path: str, content: str) -> str:
    """Write content to a file"""
    open(path, "w").write(content)
    return f"Written to {path}"

llm = ChatOpenAI(model="gpt-4o-mini")
prompt = ChatPromptTemplate.from_messages([
    ("system", "You analyze data files and write reports."),
    ("human", "{input}"),
    MessagesPlaceholder("agent_scratchpad"),
])

agent = create_openai_tools_agent(llm, [read_file, write_file], prompt)
executor = AgentExecutor(agent=agent, tools=[read_file, write_file], verbose=True)

executor.invoke({"input": "Read data.csv, analyze trends, write summary to report.md"})
```

**Зависимости**: `langchain`, `langchain-openai`, `langchain-core`, плюс их транзитивные зависимости (~50+ пакетов)
**Строк кода**: ~40 (но слои абстракций под ними — это тысячи строк)
**Под вашим контролем**: определения инструментов, шаблон промпта. Всё остальное принадлежит LangChain.

### CrewAI (~35 строк)

```python
from crewai import Agent, Task, Crew
from crewai_tools import FileReadTool, FileWriterTool

analyst = Agent(
    role="Data Analyst",
    goal="Analyze CSV data and produce insightful reports",
    backstory="You are an expert data analyst.",
    tools=[FileReadTool(), FileWriterTool()],
    verbose=True
)

task = Task(
    description="Read data.csv, analyze the trends, write a summary to report.md",
    expected_output="A markdown report with key findings",
    agent=analyst
)

crew = Crew(agents=[analyst], tasks=[task], verbose=True)
crew.kickoff()
```

**Зависимости**: `crewai`, `crewai-tools`, плюс их зависимости (~80+ пакетов)
**Строк кода**: ~35
**Под вашим контролем**: роли агентов, описания задач. Поток выполнения принадлежит CrewAI.

## Матрица компромиссов

| Параметр | Голый харнесс | LangChain | CrewAI |
|-----------|------------|-----------|--------|
| Строк кода | Больше | Меньше | Меньше всего |
| Зависимости | 1 | ~50 | ~80 |
| Отладка | Лёгкая (это ваш код) | Тяжёлая (глубокие стектрейсы) | Средняя |
| Гибкость | Полная | Ограничена абстракциями | Только на основе ролей |
| Несколько агентов | Делаете сами | Возможно, но сложно | Встроено |
| Кривая обучения | Понять API модели | Изучить концепции LangChain | Изучить концепции CrewAI |
| Путь обновления | Меняете что хотите | Ждёте обновлений LangChain | Ждёте обновлений CrewAI |
| Готовность к продакшену | Решаете вы | Зависит от стабильности версии | Новее, меньше проверена в бою |

## Скрытая цена фреймворков

### 1. Отладка чёрных ящиков

Когда что-то ломается в голом харнессе, вы смотрите на свои 60 строк. Когда что-то ломается в LangChain:

```
File "langchain/agents/openai_tools/base.py", line 147, in _plan
File "langchain_core/runnables/base.py", line 534, in invoke
File "langchain/chains/base.py", line 89, in __call__
File "langchain_core/callbacks/manager.py", line 442, in _handle_event
...
```

Вы отлаживаете чужую архитектуру.

### 2. Привязка к абстракциям

Хотите добавить стриминг? Свою память? Нестандартный паттерн вызова инструментов? В голом харнессе вы просто это пишете. Во фреймворке вы работаете в рамках его точек расширения — или форкаете библиотеку.

### 3. Текучка версий

LangChain пережил несколько крупных переработок API. Код, написанный полгода назад, сегодня может не запуститься. Голый харнесс на одном лишь пакете `openai` стабилен годами.

## Когда выигрывают фреймворки

Фреймворки не плохи. Они действительно помогают, когда:

- **Вы прототипируете** — собрать что-то работающее за один вечер, чтобы проверить идею. Потом перепишете.
- **Оркестрация из нескольких агентов** — модель «агент — задача» в CrewAI действительно хороша для сложных многоролевых рабочих процессов.
- **RAG-пайплайны** — загрузчики документов, сплиттеры и интеграции с векторными хранилищами в LangChain экономят реальную работу.
- **Вам неважна внутренняя кухня** — если агент составляет малую часть большого продукта и вам просто нужно, чтобы он работал.

## Гибридный подход

Многие продакшен-команды начинают с фреймворка и мигрируют на голый харнесс:

```
Week 1:  LangChain prototype → "It works!"
Week 4:  Hit a limitation → "Why can't I do X?"
Week 8:  Fork/override half the framework → "I'm fighting the framework"
Week 12: Rewrite as raw harness → "This is 200 lines and does exactly what I need"
```

Это нормально. Фреймворк научил вас тому, что вам нужно. Харнесс даёт вам контроль.

## Типичные ловушки

- **Начать с фреймворка, не разобравшись в основах** — нельзя отладить то, чего не понимаешь. Соберите голый харнесс хотя бы один раз, даже если никогда не используете его в продакшене.
- **Выбирать по звёздам на GitHub** — звёзды ≠ пригодность. Фреймворк с 80 тысячами звёзд, заточенный под RAG-пайплайны, не поможет вам построить кодинг-агента.
- **Страх «изобрести велосипед»** — велосипед здесь — это 50 строк на Python. Не такой уж это и велосипед.

## Что почитать дальше

- [Документация LangChain](https://python.langchain.com/) — самый популярный фреймворк
- [Документация CrewAI](https://docs.crewai.com/) — оркестрация из нескольких агентов
- [AutoGen](https://microsoft.github.io/autogen/) — фреймворк для нескольких агентов от Microsoft
- [Ваш первый харнесс](/guide/your-first-harness) — соберите голую версию сами

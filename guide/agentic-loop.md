---
author: Nexu
---

# Агентный цикл (Agentic Loop)

> **Ключевая идея:** Любой агент — это цикл: подумать, действовать, наблюдать, повторить. Сам по себе цикл тривиален. Промышленным его делает то, как вы обрабатываете крайние случаи: когда останавливаться, что делать при сбое инструментов и как не допустить бесконечного зацикливания.

## Паттерн

Агентный цикл (его также называют паттерном ReAct — Reason + Act, «рассуждай и действуй») — это базовый цикл исполнения любого ИИ-агента. Модель порождает ответ, при необходимости вызывает один или несколько инструментов, наблюдает результаты и повторяет цикл, пока задача не будет выполнена.

```
┌─────────────┐
│   Reason    │◄──────────────────┐
│  (LLM call) │                   │
└──────┬──────┘                   │
       │                          │
       ▼                          │
  ┌─────────┐    No tools    ┌────┴─────┐
  │  Tools? ├───────────────►│  Output  │
  └────┬────┘                └──────────┘
       │ Yes
       ▼
  ┌─────────┐
  │ Execute │
  │  tools  │
  └────┬────┘
       │
       ▼
  ┌─────────┐
  │ Observe │
  │ results ├─────────────────────┘
  └─────────┘
```

Это не то же самое, что простой API вызова инструментов. Одиночный вызов инструмента — это однократное действие: модель говорит «вызови эту функцию», вы возвращаете результат. **Агентный цикл** же прогоняет этот процесс многократно: модель видит результат, решает, что ей нужно больше информации, вызывает следующий инструмент, видит *его* результат и продолжает, пока не наберёт достаточно контекста, чтобы выдать окончательный ответ.

## Реализация

Минимальный агентный цикл на Python:

```python
def agentic_loop(messages: list, tools: list, max_turns: int = 25) -> str:
    """Run the agentic loop until the model produces a final text response."""
    for turn in range(max_turns):
        response = llm.chat(messages=messages, tools=tools)
        assistant_msg = response.choices[0].message
        messages.append(assistant_msg)

        # Exit condition: no tool calls means the model is done
        if not assistant_msg.tool_calls:
            return assistant_msg.content

        # Execute each tool call and append results
        for tool_call in assistant_msg.tool_calls:
            result = dispatch_tool(tool_call)
            messages.append({
                "role": "tool",
                "tool_call_id": tool_call.id,
                "content": str(result)
            })

    raise AgentLoopError(f"Agent did not complete within {max_turns} turns")
```

Параметр `max_turns` критически важен. Без него запутавшаяся модель будет крутиться бесконечно: вызывать один и тот же инструмент снова и снова, получать одну и ту же ошибку и жечь токены. Это простейший ограничитель (guardrail), и он должен присутствовать всегда.

## Параллельные вызовы инструментов

Современные API поддерживают **параллельные вызовы инструментов** (parallel tool calls) — модель может запросить сразу несколько инструментов в одном ответе. Это не просто оптимизация; это меняет поведение агента. Модель, которой нужно прочитать три файла, запросит все три одновременно, а не последовательно:

```python
# A single assistant message might contain:
# tool_calls = [read_file("a.py"), read_file("b.py"), read_file("c.py")]

for tool_call in assistant_msg.tool_calls:
    result = dispatch_tool(tool_call)
    messages.append({
        "role": "tool",
        "tool_call_id": tool_call.id,
        "content": str(result)
    })
# All three results are appended, then the model sees them all at once
```

## Бюджет ходов и условия выхода

Циклу нужны чёткие условия выхода помимо `max_turns`:

| Условие | Действие |
|-----------|--------|
| В ответе нет вызовов инструментов | Вернуть текст — агент завершил работу |
| Достигнут лимит ходов | Бросить ошибку или принудительно запросить итоговое резюме |
| Превышен бюджет токенов | Запустить сжатие контекста, затем продолжить |
| Подряд идущие одинаковые вызовы инструментов | Вероятно, агент застрял — эскалировать или прервать |
| Сигнал прерывания от человека | Поставить цикл на паузу, показать текущее состояние |

```python
def detect_loop(messages: list, window: int = 3) -> bool:
    """Detect if the agent is stuck calling the same tool repeatedly."""
    recent_calls = []
    for msg in messages[-window * 2:]:
        if hasattr(msg, 'tool_calls') and msg.tool_calls:
            recent_calls.extend(
                (tc.function.name, tc.function.arguments) for tc in msg.tool_calls
            )
    if len(recent_calls) >= window:
        return len(set(recent_calls[-window:])) == 1
    return False
```

## Стриминг внутри цикла

Промышленные харнессы (harness) стримят вывод модели токен за токеном прямо во время работы цикла. Это важно для пользовательского опыта: человек видит, как агент «думает» в реальном времени, а не смотрит на пустой экран:

```python
for turn in range(max_turns):
    stream = llm.chat(messages=messages, tools=tools, stream=True)

    tool_calls = []
    text_chunks = []

    for chunk in stream:
        delta = chunk.choices[0].delta
        if delta.content:
            text_chunks.append(delta.content)
            emit_to_user(delta.content)  # Real-time streaming
        if delta.tool_calls:
            accumulate_tool_calls(tool_calls, delta.tool_calls)

    if not tool_calls:
        return "".join(text_chunks)

    # Execute tools and continue loop
    ...
```

## Типичные ошибки

- **Отсутствие лимита ходов** — самый частый баг харнесса. Всегда задавайте максимум.
- **Проглатывание ошибок инструментов** — если инструмент падает молча, модель будет повторять попытку или галлюцинировать успех. Всегда возвращайте сообщения об ошибках как результаты инструментов, чтобы модель могла подстроиться.
- **Добавление сырых результатов** — большие выводы инструментов (целые файлы, ответы API) раздувают окно контекста. Обрезайте или резюмируйте их перед добавлением.
- **Игнорирование параллельных вызовов** — если ваш цикл обрабатывает вызовы инструментов последовательно, а модель выдала их параллельно, вы можете создать зависимости по порядку, которых на самом деле нет.

## Что почитать дальше

- [Yao et al., "ReAct: Synergizing Reasoning and Acting"](https://arxiv.org/abs/2210.03629) — оригинальная статья, формализующая паттерн Reason + Act
- [Anthropic: Building Effective Agents](https://www.anthropic.com/research/building-effective-agents) — практические паттерны для промышленных циклов

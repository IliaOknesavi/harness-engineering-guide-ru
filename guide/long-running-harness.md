---
author: Nexu
---

# Проектирование харнесса для долгоживущих агентов

> **Ключевая мысль:** агенты для коротких задач отказывают аккуратно — они либо доводят дело до конца, либо упираются в тайм-аут. Долгоживущие агенты отказывают исподтишка. Они раздувают контекст, незаметно деградируют и при этом убеждают себя, что отлично справляются, постепенно сбиваясь с курса. Проектировать харнесс (harness, обвязка/runtime-обёртка) для долгоживущих агентов — значит проектировать против этих режимов отказа.

## Почему долгоживущие агенты — это трудно

Агент для «короткой задачи» — ответить на вопрос, написать функцию, кратко изложить документ — живёт и умирает в пределах одного окна контекста. Он либо доводит работу до конца, либо отказывает заметно.

«Долгоживущий» агент работает на протяжении часов или дней: рефакторит кодовую базу, пишет отчёт на 50 страниц, прогоняет многоступенчатый конвейер. Такие агенты сталкиваются с проблемами, которых агенты коротких задач никогда не встречают:

1. **Контекст накапливается.** Каждый вызов инструмента (tool), каждый промежуточный результат, каждый шаг рассуждения добавляют токены. Окно в 200K заполняется быстрее, чем кажется.
2. **Качество незаметно деградирует.** Агент не падает — он просто становится хуже. Ответы расплываются, инструкции забываются, ранний контекст вытесняется наружу.
3. **Самооценка лжёт.** Спросите агента «хороша ли твоя работа?» — и он ответит «да». Всегда. Для 30-секундной задачи, которую можно окинуть взглядом, это нормально. Для 4-часового конвейера, за которым вы не следите, это катастрофа.

## Режим отказа №1: тревога из-за контекста

По мере того как долгоживущий агент заполняет своё окно контекста, происходит нечто контринтуитивное: модель начинает торопиться. Она сворачивает работу преждевременно, срезает углы и объявляет «готово» ещё до того, как работа действительно завершена.

Это **тревога из-за контекста** (context anxiety) — неявное осознание моделью того, что место заканчивается. Проявляется она так:

- Модель пропускает шаги, которые обычно выполняла бы
- Выдаёт более короткие, менее тщательные результаты
- Рано объявляет о завершении со словами «я охватил основные моменты»
- Избегает вызовов инструментов, которые добавили бы ещё контекста

Тревога из-за контекста возникает эмерджентно в самых разных архитектурах. Модель усвоила, что диалоги заканчиваются, и по мере того как место сокращается, её тянет к завершению.

**Окно побольше откладывает проблему, но не решает её.** Решение архитектурное: управляйте жизненным циклом контекста явно.

## Режим отказа №2: смещение самооценки

Попросите генератор оценить собственный результат. Он поставит себе 8/10 или выше — стабильно, независимо от реального качества. Это **смещение самооценки** (self-evaluation bias), и это второй тихий убийца долгоживущих агентов.

Почему? У модели есть полный контекст её собственных рассуждений — каждый выбор кажется ей оправданным. Признать провал означает противоречить предыдущим выводам, а LLM этому сопротивляются. К тому же обучающие данные вознаграждают уверенность сильнее, чем самокритику.

В короткой задаче проблемы ловит человек. В долгоживущей задаче агент работает автономно. Если он оценивает собственные результаты и всякий раз говорит «выглядит хорошо», ошибки накапливаются бесконтрольно.

```
Short task:   Agent produces → Human reviews → Feedback
Long-running: Agent produces → Agent reviews → "Looks great!" → Errors compound
```

Вывод из проектирования состязательных сетей применим и здесь: **никогда не позволяйте генератору самому проверять свой экзамен.**

## Управление контекстом: сброс или уплотнение

Когда контекст заполняется, у вас есть два варианта. У каждого свои реальные компромиссы.

### Сброс контекста

Стереть диалог и начать с чистого листа. Передать в новый контекст краткую сводку предыдущей работы в качестве «брифинга».

```
Turn 1-50:  [full conversation history]
            ↓ context 80% full
Turn 51:    [system prompt + summary of turns 1-50 + current task]
            ↓ fresh start, ~10% context used
Turn 51-100: [continues from summary]
```

**Плюсы:** чистый лист, предсказуемый бюджет токенов, устранение тревоги из-за контекста для нового сегмента.

**Минусы:** потери при сжатии — сводки упускают нюансы и неудачные подходы. «Сводка сводки» деградирует от сброса к сбросу. Агент может снова забрести в тупики.

### Уплотнение контекста

Выборочно сжать более старые ходы, оставив недавние нетронутыми. Свернуть многоходовые рассуждения в сводки, отбросить многословные выводы инструментов.

```
Turn 1-20:  [compressed: 3-line summary of early exploration]
Turn 21-40: [compressed: key decisions and outcomes]
Turn 41-50: [full detail: recent work in progress]
```

**Плюсы:** сохраняется непрерывность. Постепенность — недавние ходы остаются подробными, старые сжимаются. Агент сохраняет осведомлённость о том, что он уже пробовал.

**Минусы:** качество сжатия неравномерно. Реализовать сложнее. Уплотнённый контекст может запутать модель, если сводки противоречат недавнему состоянию.

### Что выбрать?

| Сценарий | Предпочесть |
|----------|----------|
| Задачи с чёткими фазами (исследование → написание → проверка) | Сброс между фазами |
| Непрерывная итерация над одним артефактом | Уплотнение |
| Агент часто возвращается к ранее принятым решениям | Уплотнение (сохраняет историю решений) |
| В контексте накопилось много выводов инструментов | Сброс (выводы инструментов сжимаются плохо) |

На практике многие харнессы используют гибрид: уплотнение внутри фазы, сброс между фазами.

## Архитектура «генератор — оценщик»

Заимствуем из GAN: генератор создаёт, дискриминатор судит — отдельные сети с противоположными целями. Применим тот же принцип к агентам:

```
┌─────────────┐         ┌──────────────┐
│  Generator  │────────►│  Evaluator   │
│  (Agent A)  │         │  (Agent B)   │
│             │◄────────│              │
│  Produces   │ feedback│  Judges      │
│  output     │         │  output      │
└─────────────┘         └──────────────┘
        │                       │
        │    Separate context   │
        │    Separate prompt    │
        │    Separate criteria  │
```

**Ключевые правила проектирования:**

1. **Раздельные контексты.** Оценщик (evaluator) видит только результат, а не рассуждения генератора. Это предотвращает смещение из сочувствия.
2. **Явный критериальный лист (rubric).** Оценивайте по чек-листу, а не «на глаз». «Обрабатывает ли код граничный случай X?» лучше, чем «Хорош ли код?».
3. **Действенная обратная связь.** Возвращайте конкретные проблемы, а не баллы. «Функция `parse_input` не обрабатывает пустые строки» полезно. «7/10» — нет.
4. **Бюджет итераций.** Ограничьте цикл. Без лимита перфекционист-оценщик плюс рьяный генератор дают бесконечный цикл.

```python
def generator_evaluator_loop(task, max_iterations=3):
    output = None
    for i in range(max_iterations):
        # Generator: produce or revise
        if output is None:
            output = generator.run(task)
        else:
            output = generator.revise(task, output, feedback)

        # Evaluator: judge with fresh eyes
        evaluation = evaluator.judge(task, output)  # no generator context!

        if evaluation.passes:
            return output

        feedback = evaluation.issues

    return output  # best effort after max iterations
```

## Трёхагентная архитектура: планировщик → генератор → оценщик

Для сложных долгоживущих задач добавьте **планировщик** (Planner) для декомпозиции, исполнения и контроля качества.

```
                    ┌─────────────┐
                    │   Planner   │
                    │             │
                    │ Decomposes  │
                    │ task into   │
                    │ subtasks    │
                    └──────┬──────┘
                           │
                           ▼
              ┌─── subtask list ───┐
              │                    │
              ▼                    ▼
     ┌─────────────┐      ┌─────────────┐
     │  Generator  │      │  Generator  │   (parallel or sequential)
     │  subtask 1  │      │  subtask 2  │
     └──────┬──────┘      └──────┬──────┘
            │                    │
            ▼                    ▼
     ┌─────────────┐      ┌─────────────┐
     │  Evaluator  │      │  Evaluator  │
     │  subtask 1  │      │  subtask 2  │
     └──────┬──────┘      └──────┬──────┘
            │                    │
            └────────┬───────────┘
                     ▼
              ┌─────────────┐
              │   Planner   │
              │  (reviews   │
              │   results,  │
              │   re-plans  │
              │   if needed)│
              └─────────────┘
```

**Планировщик** — разбивает цель на суб-задачи с критериями успеха. Перепланирует, когда оценщики сигнализируют о проблемах. Держит общее видение, но сам не исполняет.

**Генератор** — выполняет по одной суб-задаче за раз в свежем контексте. Имеет инструменты, файлы, среды исполнения. Не оценивает собственную работу.

**Оценщик** — видит только результат генератора (а не его рассуждения). Оценивает по критериям планировщика. Возвращает «прошло/не прошло» плюс конкретные проблемы.

Критически важное свойство: **каждый агент работает в собственном окне контекста.** Генератор может забить свои 200K окна исследованием кода и всё равно выдать чистый результат. Оценщик стартует с чистого листа. Планировщик удерживает высокоуровневую картину без деталей реализации.

```python
def three_agent_pipeline(goal, max_replans=2):
    plan = planner.decompose(goal)

    for replan in range(max_replans + 1):
        results = {}
        for subtask in plan.subtasks:
            # Generator: fresh context per subtask
            output = generator.execute(subtask)

            # Evaluator: fresh context, only sees output + criteria
            evaluation = evaluator.judge(
                subtask=subtask,
                output=output,
                criteria=subtask.success_criteria
            )

            results[subtask.id] = {
                "output": output,
                "evaluation": evaluation
            }

        # Check if all subtasks pass
        failures = [r for r in results.values() if not r["evaluation"].passes]
        if not failures:
            return assemble_results(results)

        # Re-plan: planner sees which subtasks failed and why
        plan = planner.replan(goal, results)

    return assemble_results(results)  # best effort
```

## Антипаттерны

### Антипаттерн №1: агент-монолит

Запихнуть планирование, исполнение, оценку и управление контекстом в одного агента.

```python
# DON'T DO THIS
response = llm.chat(
    system="""You are a planner, coder, reviewer, and project manager.
    First plan the work, then do the work, then review your own work.
    If the review finds issues, fix them and review again.""",
    messages=conversation  # 150K tokens of accumulated history
)
```

Это проваливается по всем перечисленным выше причинам: контекст заполняется, самооценка ненадёжна, нет разделения ответственности. Для простых задач работает, на сложных рассыпается.

### Антипаттерн №2: оценка без критериального листа

```python
# DON'T DO THIS
evaluation = evaluator.judge(
    prompt=f"Is this output good? Rate 1-10.\n\n{output}"
)
# Result: always 8/10. Always.
```

Оценщик без критериев — это просто генератор с синдромом самозванца. Всегда давайте критериальный лист:

```python
# DO THIS
evaluation = evaluator.judge(
    prompt=f"""Evaluate the following output against these criteria:
    1. Does every function have error handling for edge cases?
    2. Are all API calls wrapped in retry logic?
    3. Does the code match the spec in {spec_file}?
    4. Are there any hardcoded values that should be config?

    Output to evaluate:
    {output}

    For each criterion, answer PASS or FAIL with a one-line explanation."""
)
```

### Антипаттерн №3: бесконечное перепланирование

```python
# DON'T DO THIS
while not all_subtasks_pass:
    plan = planner.replan(goal, results)  # loops forever
    results = execute_plan(plan)
```

Всегда ограничивайте число итераций. Три неудачных перепланирования означают проблему в спецификации, а не в исполнении. Вынесите её на человека.

## Собираем всё вместе: минимальная реализация

```python
class LongRunningHarness:
    """Planner → Generator → Evaluator harness for long-running tasks."""

    def __init__(self, planner_model, generator_model, evaluator_model):
        self.planner = Agent(model=planner_model, role="planner")
        self.generator = Agent(model=generator_model, role="generator")
        self.evaluator = Agent(model=evaluator_model, role="evaluator")

    def run(self, goal, max_replans=2, max_gen_iterations=3):
        plan = self.planner.decompose(goal)

        for _ in range(max_replans + 1):
            results = {}

            for subtask in plan.subtasks:
                output = self._generate_with_eval(
                    subtask, max_iterations=max_gen_iterations
                )
                results[subtask.id] = output

            failures = {k: v for k, v in results.items() if not v["passed"]}
            if not failures:
                return self._assemble(results)

            plan = self.planner.replan(goal, plan, failures)

        return self._assemble(results, partial=True)  # best effort

    def _generate_with_eval(self, subtask, max_iterations):
        output = None
        for i in range(max_iterations):
            output = self.generator.execute(
                subtask=subtask,
                prior_feedback=output.get("feedback") if output else None
            )

            evaluation = self.evaluator.judge(
                output=output["result"],
                criteria=subtask.success_criteria
            )

            if evaluation["passes"]:
                return {"result": output["result"], "passed": True}

            output["feedback"] = evaluation["issues"]

        return {"result": output["result"], "passed": False,
                "feedback": evaluation["issues"]}

    def _assemble(self, results, partial=False):
        assembled = "\n\n".join(r["result"] for r in results.values())
        if partial:
            failed = [k for k, v in results.items() if not v["passed"]]
            assembled += f"\n\n⚠️ Incomplete subtasks: {failed}"
        return assembled
```

## Главные выводы

1. **Долгоживущий ≠ короткий, но с бóльшим запасом времени.** Режимы отказа качественно иные.
2. **Тревога из-за контекста реальна.** Управляйте жизненным циклом контекста через сбросы, уплотнение или и то и другое.
3. **Никогда не позволяйте генератору самому проверять свой экзамен.** Раздельные агенты, раздельные контексты, явные критериальные листы.
4. **Ограничивайте всё.** Максимум ходов, максимум перепланирований, максимум итераций. Неограниченные циклы сжигают токены.
5. **Сначала декомпозиция.** Куски под размер контекста предотвращают большинство проблем с контекстом ещё до их появления.

## Дальнейшее чтение

- [Context Engineering](context-engineering.md) — глубокое погружение в сборку, сжатие и бюджетирование контекста
- [Multi-Agent Orchestration](multi-agent-orchestration.md) — паттерны оркестрации за пределами трёхагентной архитектуры
- [Error Handling](error-handling.md) — обработка отказов, повторов и плавной деградации в циклах агента
- [Anthropic: Building effective agents](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/building-effective-agents) — руководство Anthropic по паттернам проектирования агентов

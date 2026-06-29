---
author: Nexu
---

# Суб-агент

> **Ключевая идея:** У одиночного агента есть единственное окно контекста. Когда задача перерастает это окно — или когда несколько независимых задач можно запустить параллельно — вам нужны суб-агенты. Паттерн простой: ведущий (leader) делегирует работу исполнителям (workers), каждый из которых работает в своём изолированном контексте, а затем сводит результаты воедино.

## Когда применять суб-агентов

Не каждой задаче нужна многоагентная оркестрация. Суб-агенты добавляют сложности — порождение процессов, управление коммуникацией, сведение результатов. Используйте их, когда:

| Признак | Пример |
|--------|--------|
| **Задача превышает один контекст** | «Отрефакторить все 50 файлов сервисов под новый паттерн обработки ошибок» |
| **Независимая параллельная работа** | «Написать тесты для модулей A, B и C» — между ними нет зависимостей |
| **Изоляция предметных областей** | «Исследовать конкурентов, затем написать маркетинговый текст» — разные навыки (skill), разный контекст |
| **Долгоиграющая фоновая работа** | «Следить за этим CI-пайплайном и чинить падения по мере их появления» |

**Не** используйте суб-агентов для задач, требующих тесной координации по общему состоянию. Два агента, одновременно правящие один и тот же файл, породят конфликты. Последовательные вызовы инструментов внутри одного контекста проще и надёжнее.

## Паттерн «ведущий — исполнитель» (leader-worker)

У самого практичного многоагентного паттерна три фазы:

```
Phase 1: Plan            Phase 2: Execute           Phase 3: Merge
┌────────────┐           ┌──────────┐               ┌────────────┐
│   Leader   │──spawn──► │ Worker A │──result──┐    │   Leader   │
│  (plans,   │           └──────────┘          │    │  (reviews, │
│  delegates)│──spawn──► ┌──────────┐          ├──► │   merges,  │
│            │           │ Worker B │──result──┘    │   reports) │
│            │──spawn──► ┌──────────┐          │    │            │
│            │           │ Worker C │──result──┘    └────────────┘
└────────────┘           └──────────┘
```

Ведущий:
1. Анализирует задачу и разбивает её на независимые подзадачи
2. Порождает по одному исполнителю на каждую подзадачу с ясной, самодостаточной инструкцией
3. Дожидается завершения всех исполнителей
4. Просматривает и сводит результаты
5. Отчитывается пользователю

```python
import subprocess
import json
import os
import tempfile
from concurrent.futures import ThreadPoolExecutor, as_completed
from dataclasses import dataclass

@dataclass
class SubTask:
    name: str
    instruction: str
    working_dir: str | None = None

@dataclass
class SubResult:
    name: str
    success: bool
    output: str
    artifacts: list[str]  # Paths to files produced

class SubAgentSpawner:
    """Spawn and manage sub-agents as isolated processes."""

    def __init__(
        self,
        agent_command: str = "python -m agent",
        max_workers: int = 4,
        timeout: int = 300,
    ):
        self.agent_command = agent_command
        self.max_workers = max_workers
        self.timeout = timeout

    def spawn(self, tasks: list[SubTask]) -> list[SubResult]:
        """Spawn sub-agents for each task and collect results."""
        results = []
        with ThreadPoolExecutor(max_workers=self.max_workers) as pool:
            futures = {
                pool.submit(self._run_agent, task): task
                for task in tasks
            }
            for future in as_completed(futures):
                task = futures[future]
                try:
                    result = future.result(timeout=self.timeout)
                    results.append(result)
                except Exception as e:
                    results.append(SubResult(
                        name=task.name,
                        success=False,
                        output=f"Agent failed: {type(e).__name__}: {e}",
                        artifacts=[],
                    ))
        return results

    def _run_agent(self, task: SubTask) -> SubResult:
        """Run a single sub-agent in an isolated process."""
        # Each sub-agent gets its own working directory
        work_dir = task.working_dir or tempfile.mkdtemp(prefix=f"agent-{task.name}-")

        # Write the task instruction to a file the sub-agent reads
        task_file = os.path.join(work_dir, "TASK.md")
        with open(task_file, "w") as f:
            f.write(task.instruction)

        # Write a result file path for the sub-agent to populate
        result_file = os.path.join(work_dir, "RESULT.json")

        env = os.environ.copy()
        env["AGENT_TASK_FILE"] = task_file
        env["AGENT_RESULT_FILE"] = result_file
        env["AGENT_WORK_DIR"] = work_dir

        proc = subprocess.run(
            self.agent_command.split(),
            cwd=work_dir,
            env=env,
            capture_output=True,
            text=True,
            timeout=self.timeout,
        )

        # Read the result file if it exists
        if os.path.exists(result_file):
            with open(result_file) as f:
                result_data = json.load(f)
            return SubResult(
                name=task.name,
                success=result_data.get("success", True),
                output=result_data.get("output", ""),
                artifacts=result_data.get("artifacts", []),
            )

        return SubResult(
            name=task.name,
            success=proc.returncode == 0,
            output=proc.stdout[-5000:] or proc.stderr[-5000:],
            artifacts=[],
        )
```

## Коммуникация через файлы

Суб-агенты не могут разделять память — они работают в изолированных процессах со своими собственными окнами контекста. Коммуникация происходит через файловую систему:

```python
import json
import time
from pathlib import Path

class FileInbox:
    """File-based message passing between agents."""

    def __init__(self, base_dir: str):
        self.base_dir = Path(base_dir)
        self.base_dir.mkdir(parents=True, exist_ok=True)

    def send(self, recipient: str, message: dict):
        """Write a message to a recipient's inbox."""
        inbox = self.base_dir / recipient / "inbox"
        inbox.mkdir(parents=True, exist_ok=True)
        msg_file = inbox / f"{int(time.time() * 1000)}.json"
        msg_file.write_text(json.dumps(message, indent=2))

    def receive(self, agent_id: str) -> list[dict]:
        """Read and consume messages from this agent's inbox."""
        inbox = self.base_dir / agent_id / "inbox"
        if not inbox.exists():
            return []
        messages = []
        for msg_file in sorted(inbox.glob("*.json")):
            messages.append(json.loads(msg_file.read_text()))
            msg_file.unlink()  # Consume after reading
        return messages

    def claim(self, agent_id: str, task_id: str) -> bool:
        """Atomic claim — prevents two agents from working the same task."""
        claim_file = self.base_dir / "claims" / f"{task_id}.claimed"
        claim_file.parent.mkdir(parents=True, exist_ok=True)
        try:
            # O_CREAT | O_EXCL = atomic create-if-not-exists
            fd = os.open(str(claim_file), os.O_CREAT | os.O_EXCL | os.O_WRONLY)
            os.write(fd, agent_id.encode())
            os.close(fd)
            return True
        except FileExistsError:
            return False  # Another agent already claimed this task
```

Паттерн с claim-файлом важен: когда несколько исполнителей могут взяться за одну и ту же задачу, атомарное создание файла служит распределённой блокировкой без какой-либо базы данных.

## Изоляция сессий

Каждый суб-агент получает полностью независимое окно контекста. Это означает:

- **Нет общей памяти** — результаты инструментов одного агента невидимы для других
- **Независимое состояние инструментов** — каждый агент загружает свои собственные навыки (skill)
- **Раздельные бюджеты токенов** — суб-агент может использовать всё своё окно в 128K под свою конкретную задачу

```python
def prepare_sub_agent_context(task: SubTask, shared_context: dict) -> list[dict]:
    """Build an isolated context for a sub-agent."""
    return [
        {
            "role": "system",
            "content": (
                "You are a sub-agent executing a specific task. "
                "Complete the task and write results to RESULT.json.\n\n"
                f"Task: {task.instruction}"
            ),
        },
        {
            "role": "system",
            "content": f"[Shared context]\n{json.dumps(shared_context)}",
        },
    ]
```

Словарь общего контекста (shared context) передаёт только то, что нужно суб-агенту — соглашения проекта, расположение файлов, ограничения. Не передавайте весь разговор ведущего; это сводит на нет саму цель изоляции.

## Git worktrees для параллельных изменений кода

Когда суб-агентам нужно править код одновременно, git worktrees предотвращают конфликты веток:

```python
import subprocess

def create_worktree(repo_path: str, branch_name: str) -> str:
    """Create a git worktree for a sub-agent to work in."""
    worktree_path = f"/tmp/worktrees/{branch_name}"
    subprocess.run(
        ["git", "worktree", "add", "-b", branch_name, worktree_path],
        cwd=repo_path,
        check=True,
    )
    return worktree_path

def merge_worktrees(repo_path: str, branches: list[str], target: str = "main"):
    """Merge all sub-agent branches back into the target branch."""
    subprocess.run(["git", "checkout", target], cwd=repo_path, check=True)
    for branch in branches:
        result = subprocess.run(
            ["git", "merge", "--no-ff", branch, "-m", f"Merge {branch}"],
            cwd=repo_path,
            capture_output=True,
            text=True,
        )
        if result.returncode != 0:
            print(f"Conflict merging {branch}: {result.stderr}")
            subprocess.run(["git", "merge", "--abort"], cwd=repo_path)

def cleanup_worktrees(repo_path: str, branches: list[str]):
    """Remove worktrees and branches after merge."""
    for branch in branches:
        worktree_path = f"/tmp/worktrees/{branch}"
        subprocess.run(
            ["git", "worktree", "remove", worktree_path],
            cwd=repo_path,
            check=True,
        )
        subprocess.run(
            ["git", "branch", "-d", branch],
            cwd=repo_path,
            check=True,
        )
```

Каждый суб-агент получает свой worktree (полную рабочую копию на уникальной ветке). Они могут свободно править файлы, не мешая друг другу. Ведущий затем сводит ветки воедино.

## Типичные ошибки

- **Чрезмерная декомпозиция** — разбиение пятиминутной задачи на 3 суб-агента добавляет больше накладных расходов, чем экономит. Применяйте суб-агентов для задач, которые занимают 10+ минут или действительно выигрывают от параллельного выполнения.
- **Общее изменяемое состояние** — два суб-агента, правящие один и тот же файл, гарантируют конфликты. Проектируйте задачи так, чтобы каждый агент работал с отдельными файлами или отдельными секциями.
- **Неограниченное порождение** — ведущий, который порождает суб-агентов, а те порождают своих собственных суб-агентов, создаёт неуправляемое дерево. Ограничивайте глубину максимум 1–2 уровнями.
- **Отсутствие тайм-аута у исполнителей** — застрявший суб-агент подвесит весь пайплайн. Всегда задавайте тайм-ауты и обрабатывайте случай, когда исполнитель падает или выходит за тайм-аут.
- **Передача избыточного контекста** — сваливание всего разговора ведущего в каждого суб-агента тратит токены и сбивает исполнителя с толку. Давайте каждому суб-агенту только то, что нужно для его конкретной задачи.

## Что почитать дальше

- [Anthropic: Building Effective Agents — Multi-Agent](https://www.anthropic.com/research/building-effective-agents) — паттерны оркестрации для продакшн-систем с несколькими агентами
- [OpenAI: Agents SDK — Handoffs](https://openai.github.io/openai-agents-python/) — паттерны делегирования и передачи (handoff) между агентами
- [Git Worktrees Documentation](https://git-scm.com/docs/git-worktree) — параллельные рабочие директории для конкурентной разработки

# Harness Engineering Guide — русский перевод

> **Неофициальный русский перевод** руководства [**Harness Engineering Guide**](https://github.com/nexu-io/harness-engineering-guide) от **Nexu**.
> Это перевод, а не первоисточник; одобрения авторов оригинала он не подразумевает.

-  Оригинал: https://github.com/nexu-io/harness-engineering-guide  ·  сайт: https://harness-guide.com
-  Лицензия: **MIT** (© 2026 Nexu) — см. [LICENSE](LICENSE). Перевод распространяется на тех же условиях.
-  Перевод выполнен **Claude Code** (Anthropic).

Практическое руководство по инженерии агентных **харнессов** — runtime-обёрток, которые превращают языковую модель в агента: исполняют инструменты, ведут память, собирают контекст и держат границы безопасности. С реальными примерами кода.

## С чего начать

- [Что такое харнесс?](guide/what-is-harness.md)
- [Ваш первый харнесс](guide/your-first-harness.md)
- [Харнесс против фреймворка](guide/harness-vs-framework.md)

## Базовые понятия

- [Агентный цикл (Agentic Loop)](guide/agentic-loop.md)
- [Система инструментов](guide/tool-system.md)
- [Память и контекст](guide/memory-and-context.md)
- [Ограничители (Guardrails)](guide/guardrails.md)

## Практика

- [Инженерия контекста](guide/context-engineering.md)
- [Песочница](guide/sandbox.md)
- [Система навыков (Skill System)](guide/skill-system.md)
- [Суб-агент](guide/sub-agent.md)
- [Обработка ошибок](guide/error-handling.md)
- [Оркестрация мультиагентных систем](guide/multi-agent-orchestration.md)
- [Планирование и автоматизация](guide/scheduling-and-automation.md)
- [Проектирование харнесса для долгоживущих агентов](guide/long-running-harness.md)
- [Управляемые агенты: разделяем мозг и руки](guide/managed-agents-architecture.md)
- [Инфраструктурный шум в оценке агентов](guide/eval-infrastructure.md)
- [Системы разрешений на основе классификатора (Auto Mode)](guide/classifier-permissions.md)
- [Осознание оценки — когда агенты понимают, что их тестируют](guide/eval-awareness.md)
- [Команды агентов: параллельные Claude строят настоящий софт](guide/agent-teams.md)
- [Initializer + Coding Agent — двухфазный паттерн харнесса](guide/initializer-coding-pattern.md)

## Справочник

- [Сравнение основных реализаций харнессов](guide/comparison.md)
- [Глоссарий](guide/glossary.md)

## Истории

- [Битва на миллиард токенов: как мы выпускали Windows-клиент OpenClaw](guide/nexu-windows-packaging.md)
- [Каждому AI-стартапу стоит быть начеку: 1000+ призрачных аккаунтов высосали ресурсы нашей платформы за 15 дней](guide/ghost-account-hunting.md)

---

*Перевод выполнен **Claude Code** (Anthropic). Нашли неточность — присылайте правку. Авторские права на исходный материал принадлежат Nexu; перевод доступен под лицензией MIT.*

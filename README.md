# missing-manuals

Коллекция заметок, описывающих окружающий мир так, как автору это понятно.

## Заметки

- [api-protocols.md](api-protocols.md) — HTTP, REST, RPC: от транспорта до архитектурных стилей.
- [mcp-protocol.md](mcp-protocol.md) — Model Context Protocol, как агенты подключают внешние инструменты.
- [tool-packaging.md](tool-packaging.md) — куда паковать тулы агента: in-process, CLI, локальный и remote MCP. Карта выбора архитектуры.
- [claude-code-internals.md](claude-code-internals.md) — внутренности Claude Code, Agent SDK, мультиагентные системы.
- [claude-code-plugins.md](claude-code-plugins.md) — skills, plugins и marketplaces в Claude Code.
- [learning-web-development.md](learning-web-development.md) — базовые понятия фронтенд-стека и сборки.
- [eta-and-ai-rollups.md](eta-and-ai-rollups.md) — финансовая модель ETA / search fund / AI-rollup и анатомия покупки сервисной компании.
- [agent-evals.md](agent-evals.md) — бизнес-эвалы для агентов: бенчмарк vs челлендж, грейдер, среда, контаминация и различение, ремесленный цикл сборки задач (на примере BitGN).
- [agentic-knowledge-base.md](agentic-knowledge-base.md) — база знаний для агента: три слоя (сырьё → граф фактов → синтез-вики), активный routing-индекс, провенанс и шов «факт/вывод». Karpathy → OKF → Abdullin в связке, на примерах справочной (налог самозанятого в Португалии) и исследовательской (рыночный ресёрч) баз.
- [reproducible-dev-environments.md](reproducible-dev-environments.md) — воспроизводимые dev-среды слоями: Homebrew vs Nix flakes, `flake.lock`, language lockfiles, Docker/VM, Railway/PaaS, Terraform и Hetzner. Карта выбора минимального достаточного набора и вопрос «кто владеет риском?».

## Как они написаны

Заметки оформляются через скилл [missing-manual-creator](https://github.com/dmial/dimalx-marketplace/tree/main/plugins/missing-manual-creator) — в нём лежат принципы missing manual и формат материала.

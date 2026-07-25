---
title: agent-team-design
type: note
permalink: tacticum/20-architecture/agent-team-design-1
---

# Дизайн Agent Team

## Роли (все на Opus)
- Team Lead (главная сессия, я взаимодействую с ним): держит план и контекст, раздаёт задачи воркерам С КОНТЕКСТОМ (что искать, зачем, почему), ревьюит планы и выводы, отправляет на переделку, единолично пишет каноническую память (session-state, decisions, daily). Репо выбирает сам под задачу.
- Explorer (read-only + Serena): разведка — карта символов, все call-sites, зависимости. Ничего не правит.
- Implementer (правит код в git-worktree): перенос/правки через символьные операции Serena. Работает в отдельном worktree репо, активирует Serena на worktree-путь.
- Verifier: прогон eval-harness (#86) + тест изоляции тенантов, дельты метрик vs baseline, вердикт acceptance (#32).

## Протокол воркера (одинаковый для всех)
1. Получает задачу + контекст от лида.
2. Составляет план, шлёт лиду (SendMessage), пока не меняет.
3. Лид одобряет/дополняет (plan_approval).
4. Воркер выполняет.
5. Шлёт вывод лиду.
6. Лид принимает или возвращает на переделку.
7. Сырые находки — в 00-Board/ (черновики); каноническую запись делает только лид.

## Память
Только лид пишет в session-state/21-Decisions. Воркеры — черновики в 00-Board/ + SendMessage лиду. Task list эфемерен, vault durable.

## Worktree
Implementer пишет в git-worktree (~/tacticum-worktrees/<repo>-<задача>, ветка migration/<задача>), не в основное дерево. Лид/я ревьюим дифф ветки, мерджим когда ок, потом worktree remove.

## Отображение
Agent Teams запускать из iTerm2 (split-pane панели). Обычная работа — можно в Apple Terminal.

## Нюансы из тестового прогона
Зафиксировано на тестовом прогоне команды (explorer + verifier, 2026-06-30, repo rag_eval_service). Всё подтверждено по факту вызовов.

- **Serena-проект называется `codex`** (репо переименовано на GitHub; путь тот же — `/Users/bubblemac/tacticum/rag_eval_service`). Воркеры активируют через `activate_project` именно как `codex`, иначе символьные инструменты падают с «No active project». Первый `get_symbols_overview` до активации гарантированно падает.
- **MCP-тулы deferred:** перед первым вызовом инструмента MCP воркер обязан подгрузить его схему через `ToolSearch` (`select:<name>`), иначе `InputValidationError`. Это норма, не ошибка конфигурации.
- **Отдельной тулы `Grep` нет** — у воркеров «No such tool available: Grep». Греп идёт через `grep` внутри Bash. В наших ролях текстовый греп по коду не нужен — символьный путь Serena рабочий и предпочтителен.
- **Чтение `.env` заблокировано правами (deny)** — секреты воркерам недоступны; имена env видны через `rag/config.py` без значений. Это by design для read-only ролей.
- **`read_memory` в Serena ждёт параметр `memory_name`** (не `memory_file_name`) — иначе schema-ошибка.

## Relations
- part_of [[20-Architecture]]
- relates_to [[STACK-GUIDE]]
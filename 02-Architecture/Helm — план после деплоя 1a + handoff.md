---
title: Helm — план после деплоя 1a + handoff
type: plan
permalink: tacticum/02-architecture/helm-plan-posle-deploia-1a-handoff
tags:
- helm
- plan
- handoff
- control-tower
- wave-1a
- wave-1b
---

# Helm — план после деплоя 1a + handoff

**Дата:** 2026-07-03. Точка: 1a-backend готов, развёрнут и проверен на dev-сервере (курированные + реальные адаптеры + живой Gateway). Дальше — два трека: что двигаем сами и что ждёт внешних входов.

## Где мы (одним абзацем)
Монитор ГД (Волна 1a) по канону `control-tower-v02` §3–6: модели, расчёты (дедлайн/план-финиш/светофор/срочно×важно), снапшот+diff, недельный бриф, 5 из 6 разрывов §6.1. Развёрнут контейнерами (`docker-compose.prod.yml`: postgres+helm) на dev-сервере, Gateway-инференс идёт из контейнера. Реальные адаптеры ИВА парсят на сервере (Jira 3016/100, git 435/145). Скелеты 1b (git-ingestion, identity) собраны. ADR субстрата принят (PG-ядро + подключение к платформе, не self-host). 210 тестов зелёные, ruff+mypy strict чисто. **Не запушено** (локальная ветка `wave-1a-backend`).

## Трек A — двигаем БЕЗ внешних входов (можно завтра)

- **A1 · Follow-up `Signal.text` (§3).** `RawJiraIssue` +поле `summary` → пробросить в `Signal.text`. Мелкая правка контракта (`ingest/contract.py`) + все адаптеры (synthetic/csv/jira) + тест. Разблокирует текстовый контекст сигналов под LLM/RAG (1b). Быстро, изолированно.
- **A2 · 1b git-ingestion на РЕАЛЬНЫХ данных (валидация Т1 на масштабе).** Реальный git (435 коммитов/145 merge, 85 с jira-ключом) ↔ реальный Jira (3016/100) сшивается по ключам: IVAONE/VCSWEB/P8 — одна реальная Jira, ключи резолвятся (в отличие от курированного набора, где ONE/MAIL ≠ IVAONE). Собрать реальный граф: эпик→инициатива (genesis=jira_epic) + git-сигналы Т1 commit→task→initiative + M3 work→product через `repos_manifest` (8 репо). **Честная граница:** это НЕ полный «Монитор ГД» (нет блоков/ЕОЛ/целей/sales → нет роллапа в блоки, нет дедлайнов) — это валидация Т1-сшивки кода на живом масштабе, территория 1b. Даёт демо «реальный код ↔ реальные задачи» пока ждём кураторскую нарезку.
- **A3 · Прод-обвязка (инфра, по желанию).** reverse-proxy/TLS перед :8000 (на dev-сервере уже есть Traefik). Сейчас приложение — голый HTTP без auth. OIDC project-hub — когда придёт service-key (см. трек B).

## Трек B — ЖДЁТ внешних входов (добываешь ты)

| Что разблокирует | Нужно | От кого |
|---|---|---|
| **Полный реальный граф 1a** (дерево ответственности, роллап в блоки, дедлайны) | кураторская нарезка **блоков + ЕОЛ + целей** + реальный **CRM/sales** | CEO / CPO / Sales Ops («всё кроме зп и cv» — обещано) |
| OIDC-auth + коннекторы memory/knowledge_rag | **project-hub service-key** | инфра / платформа |
| 1b RE-обогащение (work→product, дубли, поколение) | **repomix** (код-снимки) + прогон RE | DevOps |
| 1b полный PR/MR (title/reviewers/approvals) | **GitLab PAT** | DevOps |
| 1b скоринг ФИО | 🔴 **штатка/зарплаты/CV** + решение по хостингу (§12.7) | HR + governance |

## Рекомендация приоритета
Завтра: **A1** (быстро, разблокирует RAG-контекст) → **A2** (самое ценное самостоятельное: реальный срез на живых данных, следующая волна по канону). **A3** — когда дойдут руки до прод-доступа. Трек B — по мере поступления входов от тебя; как придёт кураторская нарезка, сразу собираем полный реальный 1a-граф.

## Процесс
Работаем командой агентов (тимлид + explorer + implementer + verifier, Opus/High), воркеры в изолированных worktree, независимый verifier на живом Postgres (ловит то, что unit-тесты пропускают — так поймали 2 FK-бага). Любое отступление от канона — явным ADR в памяти, не тихим дрейфом.

---

## Handoff для новой сессии (старт завтра)

**Первым делом:** прочитать эту заметку + [[Helm — деплой 1a на dev-сервер (D1–D4 закрыты)]] + канон `control-tower-v02` (03-Decisions).

**Состояние кода:** `~/tacticum/helm`, ветка `wave-1a-backend` (локально, не запушено). 210 тестов, ruff+mypy strict чисто.

**Поднять/проверить локально:**
```
cd ~/tacticum/helm
uv run pytest -q
docker compose up -d postgres && uv run alembic upgrade head
uv run python scripts/seed_db.py --source csv    # курированный
uv run uvicorn helm.main:app --reload            # → :8000/docs
```

**Сервер (через ssh-manager MCP, алиас `helm`):** контейнеры `helm-postgres-1` + `helm-helm-1` в `/opt/helm`. Пере-сид/проверка:
```
docker exec -u root helm-helm-1 chmod -R a+rX /app/data          # права данных (cp кладёт как root)
docker compose -f docker-compose.prod.yml exec -T helm uv run --no-sync alembic downgrade base
docker compose -f docker-compose.prod.yml exec -T helm uv run --no-sync alembic upgrade head
docker compose -f docker-compose.prod.yml exec -T helm uv run --no-sync python scripts/seed_db.py --source csv
curl -sS localhost:8000/api/gaps
```

**Грабли (важно):** persist_graph аддитивен → чистить схему перед сменой датасета; `docker cp` данных → потом `chmod -R a+rX` под юзера `helm`; `RealGitAdapter` ждёт пути к CSV, не каталог (`commits_path=data/repos/commits.csv`).

**Данные:** весь `data/` вне git (реальные ИВА + курированные), на сервере лежат в контейнере `/app/data`. 🔴 зп/CV НЕ поставляются.

**Рекомендованный первый шаг завтра:** A1 (Signal.text), затем A2 (git-ingestion на реальных). Распределить на команду агентов.

Связано: [[Helm — деплой 1a на dev-сервер (D1–D4 закрыты)]], [[control-tower-v02 — чеклист выравнивания]].

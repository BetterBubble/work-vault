---
title: Helm — деплой 1a на dev-сервер (D1–D4 закрыты)
type: note
permalink: tacticum/01-sessions/helm-deploi-1a-na-dev-server-d1-d4-zakryty
tags:
- helm
- deploy
- control-tower
- wave-1a
---

# Helm — деплой 1a на dev-сервер (D1–D4 закрыты)

**Дата:** 2026-07-03. Контейнерный деплой Монитора ГД (Волна 1a) поднят и полностью проверен на dev-сервере.

## Что сделано

- **D1–D3:** инфра на сервере (docker + uv), воспроизводимые артефакты (`Dockerfile`, `docker-compose.prod.yml`, `scripts/deploy.sh`, `.dockerignore`), перенос кода rsync-ом, подъём стека, смоук на синтетике (75 инициатив, API 200).
- **D4:** реальные данные + Gateway + проверка на сервере.

## Развёрнутая топология

- `docker-compose.prod.yml`: `helm-postgres-1` (postgres:16, healthy, volume `helm_pg_prod`) + `helm-helm-1` (образ из локального Dockerfile, юзер `helm` uid 10001, порт 8000:8000).
- Приложение реально ходит в **платформенный LLM Gateway** из контейнера: `GatewayLeadTimeEstimator`, живой инференс (lead_time=15 дней). Креды в `.env` на сервере (не в git).
- Пока HTTP :8000 без auth — прод-обвязка (OIDC project-hub + reverse-proxy/TLS) отдельным шагом.

## Проверено на сервере (D4)

- **Курированный сид чистый** (после `alembic downgrade base`+`upgrade head`+`seed --source csv`): 13 инициатив, 7 продаж, 18 сигналов (git 8/6). `/api/gaps` = goals_without_work `[G1,G3,G4]`, promises_without_work `[S5,S6]`, work_without_goal `[CS-401,BE-501,git×2]`, work_without_money `[MAIL-210,MCU-300,CORE2-400]`. `/api/portfolio` = 6 блоков (B-ONE/MAIL/MCU/TEL/SHARED/CORE2) с ЕОЛ. `/api/gantt` goal:G2 «ГОСТ-шифрование почты» deadline 2026-10-01.
- **Реальные адаптеры парсят на сервере:** RealJiraAdapter — 3016 задач + 100 эпиков; RealGitAdapter — 435 коммитов / 145 merge / 85 коммитов с jira-ключом; repo→product манифест — 8 репозиториев.

## Грабли (для будущих деплоев)

- **persist_graph аддитивен (upsert, не replace):** сид synthetic, потом csv поверх → в БД смесь датасетов. Для чистого среза сначала пересоздать схему (`alembic downgrade base` + `upgrade head`), потом сидить.
- **Права на `docker cp` данных:** cp кладёт файлы как root, контейнер бежит юзером `helm` → `PermissionError` при чтении. Фикс: `docker exec -u root <container> chmod -R a+rX /app/data`.
- **`RealGitAdapter` ждёт пути к CSV**, не каталог: `RealGitAdapter(commits_path=data/repos/commits.csv, merges_path=data/repos/merges.csv)`.

## Дальше (ждёт внешних входов)

- Реальный граф 1a — кураторская нарезка блоков/ЕОЛ/целей (CEO/CPO) + реальный CRM. «Всё кроме зп и cv» — обещаны.
- Прод-обвязка: OIDC project-hub + TLS/reverse-proxy.
- Follow-up: `RawJiraIssue` +summary → `Signal.text` (под LLM/RAG-контекст).

Связано: [[control-tower-v02 — чеклист выравнивания (канон = спека)]], ADR субстрата (PG-ядро + подключение к платформе).

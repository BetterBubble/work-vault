---
title: Helm — runbook работы с dev-сервером (SFTP-доставка, не rsync)
type: guide
permalink: tacticum/20-architecture/helm-runbook-raboty-s-dev-serverom-sftp-dostavka-ne-rsync
tags:
- helm
- control-tower
- server
- deploy
- runbook
- ssh
- sftp
---

Как работать с dev-сервером Helm. Установлено на практике 2026-07-04 (тимлид-сессия). Дополняет [[Helm — деплой 1a на dev-сервер (D1–D4 закрыты)]].

## Сервер
- [reference] Алиас ssh-manager: **`helm`** (159.194.233.33, root, key-auth). Есть ещё `zu_demo` — не путать. Код+стек в **`/opt/helm`**. Контейнеры: `helm-helm-1` (:8000) + `helm-postgres-1` (healthy). Диск /: 29G (16% занято). #server

## ГЛАВНОЕ ПРАВИЛО ДОСТАВКИ: SFTP, НЕ rsync
- [decision] **`rsync` (ssh_sync) НЕ работает** — отдельное SSH-соединение rsync таймаутит на banner exchange («Connection timed out during banner exchange», exit 255). Пулированный `ssh_execute` при этом жив. Причина — сервер не принимает свежие ssh-сессии rsync'а (лимит/фильтр). #gotcha
- [decision] **Доставка файлов на сервер = tar + `ssh_upload` (SFTP) + распаковка через `ssh_execute`.** Паттерн: локально `tar czf /tmp/x.tgz ...` → `mcp__ssh-manager__ssh_upload(localPath=/tmp/x.tgz, remotePath=/opt/helm/_x.tgz)` → `ssh_execute("cd /opt/helm && tar xzf _x.tgz && rm _x.tgz")`. `ssh_upload` — один файл по SFTP, перезаписывает. Для деревьев — тарбол. #howto
- [decision] Всё тестируем/деплоим на сервере `helm` (операторское правило). Локально — только быстрый прогон перед выкаткой. #process

## Что НЕ перезаписывать
- [warning] **`/opt/helm/.env` НЕ трогать** — там секреты (`HELM_PROJECTHUB_TOKEN` = phk, `HELM_TENANT`, `HELM_GATEWAY_BASE_URL/API_KEY`). Тарбол кода собирать с `--exclude='.env'`. `.env` gitignored, локально read заблокирован (deny). #security
- [warning] `data/` — вне git, 564M (из них `real/git` 552M). На сервер лить **селективно**: для 1a нужны только мелкие CSV (`data/real/{manual,jira,monitor-gd,identity,crm}`), НЕ `real/git`. #data

## Деплой-цикл (docker-compose.prod)
- [howto] На сервере: `cd /opt/helm && docker compose -f docker-compose.prod.yml up -d --build` → ждать postgres healthy → `... exec -T helm uv run --no-sync alembic upgrade head` → `... exec -T helm uv run --no-sync python scripts/seed_db.py --source real`. Есть `scripts/deploy.sh [synthetic|csv|real]` (up --build → wait → migrate → seed). #howto
- [howto] Смена датасета: `docker compose -f docker-compose.prod.yml down -v` (чистит том `helm_pg_prod`) перед пере-seed (persist аддитивен). После `docker cp` данных → `docker exec -u root helm-helm-1 chmod -R a+rX /app/data`. #howto
- [howto] Проверка: `curl -sS localhost:8000/api/gaps` (и /api/portfolio, /api/gantt); Swagger :8000/docs. #howto

## Отношения
- relates_to [[Helm — деплой 1a на dev-сервер (D1–D4 закрыты)]]
- relates_to [[2026-07-04 — Helm: реорг data + план 1a/1b (backend-only, Agent Team)]]

## Грабли: macOS AppleDouble `._*` ломают alembic

- [gotcha] macOS `tar czf` кладёт в архив **AppleDouble-файлы `._*`** (несут xattr `com.apple.provenance` — те самые tar-warning'и `LIBARCHIVE.xattr`). На сервере они распаковываются как `._485d55...py` (163 байта, бинарь). Alembic сканирует `alembic/versions/*.py`, цепляет `._*.py` → **`SyntaxError: source code string cannot contain null bytes`** → миграции не применяются → seed падает «relation does not exist». #gotcha
- [howto] **Профилактика:** тарить с `COPYFILE_DISABLE=1 tar czf ...` (macOS) или `--no-xattrs`. **Лечение на сервере:** `find /opt/helm -name '._*' -delete` (и внутри контейнера `docker exec <c> sh -c "find /app -name '._*' -delete"`), потом повторить `alembic upgrade`. Проверено 2026-07-04 (было 186 шт.). #howto
- [reference] Данные в контейнер: `docker cp /opt/helm/data/real helm-helm-1:/app/data/real` + `docker exec -u root helm-helm-1 chmod -R a+rX /app/data` (data/ в `.dockerignore`, в образ не входит; Postgres на volume `helm_pg_prod`). #howto

## Грабли: write-permission данных в контейнере
- [gotcha] Скрипты, которые ПИШУТ в `data/` (напр. `derive_real_manual.py` → `manual/teams.csv`), падают `PermissionError` в контейнере: `docker cp` кладёт данные от root, `chmod -R a+rX` даёт read+exec без write. Фикс: `docker exec -u root helm-helm-1 chmod -R a+rwX /app/data/real/manual` (или нужный подкаталог) перед regen. Read-only потребление (seed/score) работает и с `a+rX`. #server #gotcha
- [reference] Флаги живого субстрата: `project_to_memory.py --write`, `ingest_knowledge.py --write` (не `--live`). memory-запись медленная (LLM-экстракция) — гонять фоново. #substrate


## Пере-сид: НЕ гонять полный форс-сид на каждый деплой (2026-07-07)

- [decision] Витрины (`sd_request`, `client_deal_money`, `product_economics`, `sd_theme`…) — **материализованы**: посчитаны из CSV один раз, лежат таблицами. Пере-сид (ingest) нужен ТОЛЬКО когда меняется: (1) **данные** (новый дроп CSV), (2) **логика ingest** (новый резолвер/маппинг — применить к строкам), (3) **схема** (новая колонка → backfill). Если ничего из этого — витрина уже готова, **прогон не нужен**.
- [gotcha] Мы ошибочно делали **полный форс-пере-сид (`delete ingest_run where source like 'sd_%'` → весь чейн) на КАЖДЫЙ деплой**. Это избыточно и **ломает sha-skip** (идемпотентность ingest: он умеет пропускать неизменённые файлы по хешу). Форс заставляет всё перегонять зря → отсюда медленный пере-прогон тем (Gateway-эмбеддинги ~5мин, ~100 запросов).
- [howto] **Правильно:**
  - Код-деплой без смены данных/логики → **БЕЗ пере-сида** (только rebuild+restart).
  - Пере-сид **только затронутого источника** (поменяли company-логику → `refresh_companies`, НЕ весь чейн; не трогаем servicedesk/themes).
  - Backfill новых колонок — **в миграции** (тогда форс-сид не нужен). Если нет — форс только конкретного `sd_*`-provenance, не всё.
  - Тяжёлый пере-прогон тем (`refresh_sd_themes`) — **только когда реально меняются обращения**.
  - Кнопка «Пересчитать» (Справочник v2) = **точечный** ручной пересчёт после правки refdata, не «всё заново».
- [reference] В проде: данные обновляются по расписанию (новый дроп) → плановый refresh → sha-skip обрабатывает только изменившееся → свежесть по `ingest_run.as_of`. Ручной полный пере-сид — не норма.


## Git-flow деплоя (2026-07-08, указание руководителя Dmitrij Lebedev)
- [decision] **Деплой — только через main.** Поток: **ветка → коммит → PR → merge в main → заливаешь** (деплой из main). Не работать через `git stash` (криво/ненадёжно).
- [context] CI/CD выстроим позже — сейчас не критично, но **точка деплоя = main**. Feature-ветки не деплоим напрямую; сначала merge в main, потом заливка на сервер.
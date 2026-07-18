---
title: 2026-07-06-verify-merge-livedb-wave-1a-backend
type: note
permalink: tacticum/00-inbox/2026-07-06-verify-merge-livedb-wave-1a-backend
---

# Verify: merge origin/main на живой БД (wave-1a-backend) — сырой лог

Дата: 2026-07-06 · Verifier · ветка wave-1a-backend @ b047ced (merge origin/main, +47 коммитов, 16 миграций)
Baseline: unit-тесты зелёные на тимлиде (587 passed). Здесь — независимая проверка на живом Postgres.

## Среда
- docker: доступен.
- Порт 5432 занят ЧУЖИМ контейнером `helm-wave1a-polish-postgres-1` (up 2 дня, healthy) — БД другого воркера/worktree. НЕ трогал.
- Поднял изолированный контейнер `helm-verify-pg` (postgres:16) на порту **5433**, свежий, без volume.
- URL переопределён через env: `HELM_DATABASE_URL=postgresql+asyncpg://helm:helm@localhost:5433/helm`.
  config.py читает `HELM_*` из окружения (env-var перекрывает .env); alembic/env.py берёт URL из `Settings()`.

## 1. Postgres с нуля — PASS
Чистая БД, 0 таблиц в public на старте.

## 2. alembic upgrade head (16 миграций линеаризуются) — PASS
- `alembic heads` = единственный head `d2e3f4a5b6c7` (нет multiple heads).
- upgrade с нуля: все 16 ревизий прошли по цепочке 485d55ee6aaa → … → d2e3f4a5b6c7, EXIT=0.
- Нет ошибок «multiple heads» / «revision not found» / битого down_revision.
- `alembic current` = d2e3f4a5b6c7 (head).

## 3. Обратимость downgrade base → upgrade head — PASS
- downgrade base: все 16 downgrade дошли до base, осталась только `alembic_version` (норма).
- re-upgrade head: чисто, current снова d2e3f4a5b6c7.
- Вывод: ни одна из 16 миграций (наши 3 + их 13) не необратима, down_revision-цепочка цела.

## 4. seed_db.py (source=synthetic) — PASS
«Снимок персистирован (источник: synthetic)». Без FK-ошибок (2 прошлых FK-бага не воспроизвелись).
Контроль строк: initiative=75, goal=20, sales_link=52, блоки=6, цели=20, продажные=25, зависимости=12, назначения=394, person_email=1165.

## 5. seed_vitrines.py (новый код Diaret) — ЧАСТИЧНО / БЛОКЕР ДАННЫХ
Раннер отработал GRACEFULLY (корректный skip, не упал), НО запись витрин на живой БД НЕ проверена:
`data/vitrines` отсутствует → все источники скипнуты (conformance/repos/commits/merge_requests/commit_effort/adp_adoption/meetings).
Контроль: conformance_row=0, repo_row=0, adp_adoption=0 — путь персистентности витрин НЕ исполнен.
Готового датасета с нужными именами/схемой в репо нет (есть только repos_manifest.csv/repos_registry.csv — иная схема).
Данные не выдумывал. Чтобы реально проверить запись витрин на живой БД — нужен датасет data/vitrines (или указание пути).
NB: unit-тесты (587) путь витрин, вероятно, покрывают на фикстурах — но live-DB запись именно нового кода не подтверждена.

## 6. Смоук FastAPI — PASS
- `helm.main:app` импортируется (FastAPI), lifespan стартует чисто (коннект к БД 5433 OK).
- 9 включённых роутеров, 37 эндпоинтов под `/api` в OpenAPI, включая новые: `/api/cio/*` (conformance/repos/effort/adp-impact/…), `/api/meetings/*`, `/api/auth/*`, `/api/snapshot*`.

## Артефакты
- Контейнер `helm-verify-pg` на 5433 оставлен запущенным (для доп. проверок). Teardown: `docker rm -f helm-verify-pg`.
- Чужой `helm-wave1a-polish-postgres-1` (5432) не тронут.
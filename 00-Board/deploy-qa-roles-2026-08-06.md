---
title: deploy-qa-roles-2026-08-06 — выкатка четырёх QA-ролей на прод
type: note
status: current
created: 2026-08-06 15:30
updated: 2026-08-06 15:30
permalink: tacticum/00-board/deploy-qa-roles-2026-08-06
repo: tacticum-dev
project: tacticum-dev / профили / QA
lead: lead-qa
tags:
- board
- runbook
- qa
- прод
---

# Выкатка QA-ролей 06.08 — сид, деплой, перепин

**Основание:** мерж PR #236 Президентом + его ОК на деплой и перепин в чате.
Прод-коммит: **`c972c6f0`**.

## ✅ ВЫПОЛНЕНО (факты, не план)

| Шаг | Факт |
|---|---|
| бэкап каталога | `/root/backups/tacticum_catalog_pre-qa-kit_20260806_0820.dump` — 21 733 290 Б, **231 объект**, читаемость проверена `pg_restore -l` |
| снимок пинов | `/root/backups/repin_before_20260806_qakit.txt` — **29 живых установок** |
| SQL отката пина | `/root/backups/repin_rollback_20260806_qakit.sql` — 3 строки |
| тег отката образа | `tacticum-catalog-mcp:rollback-20260806` → `b2161d73eeee`, поставлен **до** сборки |
| `git pull` | `cd0ddfc7` → `c972c6f0`, fast-forward, рабочее дерево чистое |
| сборка образа | `docker compose build catalog-mcp` — успешно |
| **проверка образа ДО замены** | отдельный контейнер `tmp-newimage-check`: healthy, `/healthz` 200, `/readyz` 200, `/mcp` → **401** (транспорт и авторизация живы), **0 ошибок** в логе |
| деплой | `up -d catalog-mcp` — healthy, `/readyz` 200, **0 ошибок** в логе |
| **сид** | **9 профилей `seed_created`**, выборочный — только наши; провенанс `seeded_from_commit=c972c6f0` у всех девяти |
| перепин | `UPDATE 2` + `UPDATE 1`; сортированный диф со снимком «до» — **ровно 3 изменённые установки**, 29 до и 29 после |
| битые пины | **0** |

## Что теперь в каталоге

| Профиль | было | стало |
|---|---|---|
| `iva-role-qa` | 0.8.1 | **0.12.0** |
| `iva-role-qa-web` | 0.4.1 | **0.8.0** |
| `iva-role-qa-mobile` | 0.1.1 | **0.6.1** |
| `iva-role-qa-desktop` | 0.1.1 | **0.5.1** |
| `tacticum-autotest-core` | 0.5.0 | **0.7.1** |
| `iva-pytest-appium-autotest-base` | 0.1.0 | **0.4.2** |
| `iva-qa-delivery-base` | 0.4.0 | **0.5.0** |
| `iva-qa-tms-base` | 0.2.0 | **0.3.1** |
| `iva-squish-desktop-autotest-base` | 0.1.0 | **0.2.1** |

Установки перепинены: `iva-role-qa` ×2 → 0.12.0, `iva-role-qa-web` → 0.8.0.

## Проверено свойством на проде, а не отчётом

- **Канал записи цел**: рёбра зависимостей показывают `iva-write-base` 0.1.0 на позиции 6 у
  `iva-role-qa` 0.12.0 и `iva-role-qa-web` 0.8.0. Это была главная опасность мержа.
- **Предупреждение о перезаписи доехало до людей**: в ингредиентах `iva-role-qa-mobile` 0.6.1
  оба фрагмента точки входа (`claude-md-fragment`, `codex-agents-md`) содержат
  «перезаписывает без спроса», «оставьте свой» и замер «3236» — то есть текст приезжает в
  репозиторий, а не остаётся в манифесте.
- Тесты на прод-коммите `c972c6f0`: **1319 passed · 42 skipped · 0 failed · 0 error**;
  гейты версий (64 профиля) и зеркал (75 в 7 парах) — OK.

## Грабли, на которые наступил

**Проверка нового образа сначала дала `readyz` 500 — и это была моя ошибка, не образа.**
Временный контейнер я поднял с `--env-file .env`, а `DATABASE_URL` в `.env` отсутствует
(он приходит из compose) — контейнер не видел БД. Правильно: брать окружение из рабочего
контейнера (`docker inspect ... .Config.Env`), а не из `.env`. С правильным окружением образ
дал 200/200/401 и ноль ошибок.

**Диф пинов шумит на ровном месте.** `ORDER BY profile_id` при равных ключах даёт нестабильный
порядок строк, и наивный `diff` показал 12 расхождений вместо 3. Сортировать перед сравнением
обязательно, иначе легко прочитать чужие строки как изменённые.

## Откат, если понадобится

1. образ: `docker tag tacticum-catalog-mcp:rollback-20260806 tacticum-catalog-mcp:latest` +
   `docker compose -f docker-compose.prod.yml up -d catalog-mcp`;
2. пины: `psql -f /root/backups/repin_rollback_20260806_qakit.sql`;
3. каталог целиком: `pg_restore` из `tacticum_catalog_pre-qa-kit_20260806_0820.dump`.

Сид иммутабельный — прежние версии остались на месте и активны, откат пина возвращает людей
на них без восстановления БД.

## Связано

[[merge-qa-mobile-kit-2026-08-06]] · [[verify-qa-mobile-kit-merge-2026-08-06]] ·
[[runbook-sid-pin-iva-write-base-2026-08-05]]

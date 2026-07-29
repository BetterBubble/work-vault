---
title: report-verify-3-roli-iva-write-base
type: report
permalink: tacticum/00-board/report-verify-3-roli-iva-write-base
status: draft
role: verifier
repo: tacticum-dev
created: 2026-07-22 10:30
tags:
- verify
- role-presets
- iva-write
- acceptance
- draft
archived-at: 2026-07-29 18:12
---

# report — верификация 3 роли-пресета + iva-write-base (Этап 3)

## Сигналы
- [ ] to:director from:verifier Этап 3 СДЕЛАН как договаривались (3 роли + iva-write-base на ветке feat/iva-write-base, тест 35 passed); QA собран по Треку B (iva-qa-autotest-base, ГД-санкц.) — не смержено в main, флаг: в QA-роли больше нет requirement_tests.

Проверка read-only (код не правил). Baseline: approved-план `plan-iva-tri-deliveravla`. Работа НЕ на main — ветка `feat/iva-write-base`, worktree `/Users/bubblemac/tacticum/tacticum-dev-iva-write` (4 коммита cf8dd61→8a7e854, ещё не смержено в main).

## Вердикт: СДЕЛАНО как договаривались (с одной ГД-санкционированной эволюцией по QA)

## Дельта по ролям (ЕСТЬ во всех случаях, worktree feat/iva-write-base)
| Роль | план (baseline) | факт | вердикт |
|---|---|---|---|
| iva-role-architect | [core, iva-analysis-base, iva-write-base] | = точно | OK (Вариант A: реюз iva-analysis-base, отд. arch-лейн не делали) |
| iva-role-qa | [core, iva-analysis-base, iva-write-base] | [core, **iva-qa-autotest-base**, iva-write-base] | ГД-санкц. отклонение (Трек B) |
| tacticum-role-techwriter | [core, tacticum-documentation-base, iva-write-base] | = точно | OK |

Все 3 роли: тонкий leaf `ingredients: []` ✓, persona.role (architect/qa/techwriter) ✓, README+CHANGELOG+manifest по образцу iva-role-analyst ✓, строки в ROLE_LANES теста ✓. Отдельного quickstart-файла нет — но его нет и у эталона iva-role-analyst (README = quickstart), конвенция соблюдена.

## iva-write-base — ЕСТЬ
Leaf-лейн, 1 ингредиент `iva-write` (mcp_server_spec, url `https://mcp.tacticum.ru/iva-write/mcp`, bearer). Без depends_on (depth-1 ok). Все 3 роли композят его.

## Отклонение QA (Трек B) — НЕ дефект, ГД принял
Вместо плейсхолдера `iva-analysis-base` собран НОВЫЙ лейн `iva-qa-autotest-base` (9 реальных QA-скиллов автотест-команды one-web: генерация/прогон/починка/публикация pytest). Это ровно созвон-договорённость «собрать из уже существующего QA-контента tacticum-dev». Подтверждено принятыми заметками (`qa-profil-trek-b-gotovo`, `plan-qa-profile`).
- ⚠️ Побочный флаг (не блокер): при переходе на Трек B QA-роль потеряла `iva-analysis-base` → тул `requirement_tests` (helm-analyst) больше НЕ в составе QA-роли. Созвон явно упоминал requirement_tests для QA. Allure-интеграция присутствует в другой форме (чтение TC из Allure TestOps URL как вход для write-autotest). Стоит подтвердить у ГД, что requirement_tests для QA больше не нужен в роли.

## Acceptance-тест — ЗЕЛЁНЫЙ
`apps/backend/tests/catalog/test_iva_role_presets.py` → `.venv/bin/python -m pytest` → **35 passed in 0.42s**. Покрывает: pure-composition-leaf, ingredients==[], depends_on==ROLE_LANES, depth-1 bases, single-owner pairwise-disjoint, schema-v2 validation. Целевые роли присутствуют в ROLE_LANES.
- persona.role enum: в `manifest.v2.schema.json` НЕ ограничен (top-level допускает доп.поля, валидация проходит); enum-контроль живёт в тесте ROLE_PERSONA, не в схеме. Функционально ок.

## Item 4 (лёгкая проверка) — my_todo ЕСТЬ и СМЕРЖЕН
`helm/src/helm/interface/mcp/analyst_server.py:1480` `async def my_todo(...)`, тонкая обёртка на `task_mgmt_router._my_todo`. Коммит `9468d9d`, смержен в helm **main** (PR #82, ea295f8). Всего 19 `@mcp.tool()`.

## Оговорка честности
Прогон = pytest на структуре манифестов (не на реальном источнике: провижн на CLI + дым-прогон architect→arch_map / QA→автотест на реальном требовании из плана НЕ выполнял — вне зоны read-only без окружения). Acceptance по коду/структуре зелёный; «на реальных данных» дым-прогон остаётся за implementer/controller-гейтом.
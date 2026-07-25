---
title: helm-v12-e2e-verify-raw-2026-07-03
type: note
permalink: tacticum/00-board/helm-v12-e2e-verify-raw-2026-07-03
status: archived
updated: 2026-07-18
---

# Helm #12 — E2E-верификация внедрения данных (S1–S3): сырой прогон

Дата: 2026-07-03. Ветка `wave-1a-backend`. Дерево `/Users/bubblemac/tacticum/helm`.
Verifier — только измерение, код не правил.

## 1. Регрессия
- `uv run pytest -q` → **146 passed in 1.38s** ✅ (ожид 146)
- `uv run ruff check .` → **All checks passed!** ✅
- `uv run --with mypy mypy` → **Success: no issues found in 76 source files** ✅

## 2. Inline-пайплайн (без БД) — ВСЁ СОШЛОСЬ ✅
- `len(g.initiatives) == 13` ✅ (17−4)
- Поглощены (нет в id): `sales:S1`, `jira_epic:MAIL-200`, `sales:S2`, `jira_epic:ONE-100` — все True ✅
- `goal:G2`: merged_from=`('sales:S1','S1','jira_epic:MAIL-200','MAIL-200')` (содержит S1+MAIL-200 ✅); deadline=2026-10-01 ✅; important=8_750_000.0 ✅
- `goal:G5`: merged_from=`('sales:S2','S2','jira_epic:ONE-100','ONE-100')` (содержит S2+ONE-100 ✅); deadline=2026-07-15 ✅; important=4_800_000.0 ✅
- Три списка разрывов (факт):
  - promises_without_work = `['S3','S4','S5','S6','S7']` (S5,S6 есть ✅; S1,S2 нет ✅)
  - goals_without_work = `['G1','G3','G4']` (G2,G5 нет ✅)
  - urgent_important_without_owner = `[]`

## 3. E2E Postgres+API — БЛОКЕР ❌
- `docker compose up -d postgres` → Running; `pg_isready -U helm` → ready.
- `alembic downgrade base` ✅ → `alembic upgrade head` ✅.
- `python scripts/seed_db.py --source csv` → **ПАДЕНИЕ**:
  `ForeignKeyViolationError: sales_depends_on_initiative_id_fkey` —
  `Key (initiative_id)=(MAIL-200) is not present in table "initiative"`.
  (repository.py:385 `_upsert_sales` → `persist_graph`)
- API (/gaps, /gantt) НЕ поднимал — БД не засеяна, шаг недостижим.

### Корень (диагностика, не правка)
`sales.depends_on` содержит id, которых нет среди `initiative_id`:
```
S1 -> MAIL-200      (поглощён merge в goal:G2 — ссылка не переписана/не снята)
S2 -> ONE-100       (поглощён merge в goal:G5 — то же)
S3 -> MAIL-210      (существует как jira_epic:MAIL-210 — НЕ namespaced)
S4 -> MCU-300       (существует как jira_epic:MCU-300 — НЕ namespaced)
S7 -> CORE2-400     (существует как jira_epic:CORE2-400 — НЕ namespaced)
```
Два независимых дефекта DB-пути:
1. **Namespace-mismatch (S2 depends_on resolution):** `sales.depends_on` хранит голый ключ эпика (`MAIL-210`) вместо namespaced `jira_epic:MAIL-210`. 3 из 5 ссылок ведут на реально существующие инициативы, но FK не матчит из-за префикса.
2. **Merge не чистит depends_on на поглощённые id:** `MAIL-200`/`ONE-100` ушли в goal:G2/goal:G5 при genesis-merge; ссылки `sales_depends_on` на них не переписаны на приёмник и не сняты → висячий FK.

Почему шаг 2 зелёный, а шаг 3 нет: `build_graph`/`build_portfolio` не проверяют ссылочную целостность — битые depends_on проносятся молча; FK ловит их только на persist в Postgres.

## ИТОГ: REJECT
S1 (CSV-адаптер) и genesis-мержи (S3, in-memory) корректны. Но заявленный acceptance #12 шаг 3 («seed → инициативы 13» + API на реальных данных) не проходит: DB-сид падает на FK. Требуется правка резолюции `sales.depends_on` (namespaced id + обработка поглощённых merge-таргетов) — вне зоны verifier.
---

## RE-VERIFY после фикса #15 (+#14) — раздел 3 — REJECT (новый FK)

`alembic downgrade base && upgrade head` ✅. `seed_db --source csv` → снова падение, но на ДРУГОЙ таблице:

`ForeignKeyViolationError dependency_from_initiative_fkey`:
`Key (from_initiative)=(sales:S1) is not present in table "initiative"`
SQL: `INSERT INTO dependency (from_initiative, to_kind, to_ref, ...) VALUES ('sales:S1','initiative','goal:G2','structural',...)`

Диагностика висячих dependency (из 13 инициатив):
```
sales:S1 -> initiative:goal:G2   origin=structural   [FROM-нет: sales:S1 поглощён merge в goal:G2]
sales:S2 -> initiative:goal:G5   origin=structural   [FROM-нет: sales:S2 поглощён merge в goal:G5]
```

Корень: #15 переписал TO-конец depends_on (MAIL-200 → goal:G2) и перенёс связь в таблицу `dependency`, но FROM-конец — сам `sales:S1`/`sales:S2` — поглощён genesis-merge и в `initiative` отсутствует → FK-violation. Нужно ремапить и FROM на приёмник merge и снимать самопетли (goal:G2 → goal:G2).

Шаг 3 недостижим: БД не засеяна, uvicorn не поднимал. **ИТОГ: REJECT.**

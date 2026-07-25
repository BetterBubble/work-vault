---
title: 2C my_todo — задеплоен на прод (verify PASS на живой БД)
type: report
permalink: tacticum/00-board/2-c-my-todo-zadeploen-na-prod-verify-pass-na-zhivoi-bd
status: deployed
role: lead (тимлид)
date: 2026-07-21
autonomy: 'off'
tags:
- deploy
- my-todo
- helm
- done
---

# 2C my_todo — доставлен на прод

Пайплайн полностью пройден: explorer → implementer → verifier (реальные данные) → controller GO → merge (пользователь) → **deploy (лид, ssh-manager)** → prod-verify.

## Деплой (runbook `/opt/helm/scripts/deploy.sh`)
- **Бэкап** перед деплоем: `/tmp/helm-predeploy-mytodo-20260721-065213.sql.gz` (16MB, pg_dump).
- **⚠️ Footgun обойдён:** `deploy.sh` по умолчанию сидит БД синтетикой (SEED=1) — деплоил строго с **`SEED=0`** (только сборка + миграции). Реальные данные не тронуты.
- `git pull --ff-only` в /opt/helm: `1da2438..ea295f8` (merge PR #82 + my_todo), fast-forward, миграций не прибавилось (68→68) → `alembic upgrade head` = no-op.
- helm-контейнер пересобран и поднят; postgres НЕ пересоздавался (том цел).

## Prod-verify (доказательства на живой БД)
- **Данные целы** (сверка pre/post): epic_task=8073, requirement=1465, person=1007, approval=1, max_as_of=2026-07-10 — идентично.
- **Функциональный smoke** — задеплоенный `_my_todo` против боевой БД:
  - `e.fadin@iva.ru`: open=27 blocked=1 high=14 matched=True → **PASS**
  - `n.vasiliev@iva.ru`: open=9 blocked=3 high=1 matched=True → **PASS**
- MCP-регистрация (19-й тул) — покрыта юнитом `test_all_tools_registered`.

## Итог
2C my_todo **живёт на проде**, работает на реальных данных, данные не пострадали. Деливеравл закрыт.

## Осталось по плану
- **2A iva-write** — запаркован, требует решения по [[konflikt-2-a-iva-write-adr-0058-lichnyi-pat-razvesti-do-dizaina]] (личный PAT vs ADR-0058 техучётка).
- **2B три роли** — после лейна из 2A.

## Связано
- [[svodka-2-c-my-todo-gotov-k-dostavke-go-zhdiot-ok-polzovatelia]] · [[verify-my-todo]] · [[gate-my-todo]]
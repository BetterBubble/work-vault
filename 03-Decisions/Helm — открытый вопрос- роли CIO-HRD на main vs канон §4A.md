---
title: 'Helm — открытый вопрос: роли CIO/HRD на main vs канон §4A'
type: decision
permalink: tacticum/03-decisions/helm-otkrytyi-vopros-roli-cio-hrd-na-main-vs-kanon-ss4-a
tags:
- helm
- control-tower
- v03
- roles
- conformance
- open-question
---

# Helm — открытый вопрос: роли CIO/HRD на main vs канон §4A

Зафиксировано 2026-07-06 (тимлид-сессия, после merge origin/main в wave-1a-backend).

- [decision] **Статус: ОТЛОЖЕНО, решить позже** (решение оператора). Тихий дрейф не оставлять — этот якорь фиксирует его.
- [reference] На `origin/main` (автор Diaret) веб-фронт реализует **6 ролей: ceo·cpo·coo·cio·cco·hrd** (`web/src/screens/roles.tsx`), плюс backend-роутер `interface/api/routers/cio.py` (10 эндпоинтов: conformance/repos/effort/contributors/stack/adp-impact) и роль `hrd` (gated 🔒). #remote
- [reference] Канон `control-tower-v03.md` **§4A** явно: роли **CEO·CPO·COO·CCO**, «**нет отдельного CIO-дэша**: conformance/поколения → CPO; инженерное исполнение/зависимости/дубли → COO». То есть CIO и HRD как отдельные дэши каноном не предусмотрены. #canon
- [followup] Развилка при возврате к вопросу: (а) **ADR — обновить канон под main** (признать CIO/HRD полезными, §4A расширяется); (б) **выровнять под канон** — свернуть CIO→CPO/COO, HRD→red-контур COO (откат части кода Diaret). Оператор выбрал НЕ трогать сейчас.
- [outcome] На Шаг 0 (реконсиляция + PR в main) дрейф НЕ влияет — код Diaret сохраняется как есть, merge чистый.

- relates_to [[Helm — план: реконсиляция с main + грань CPO/conformance (v03)]]
- implements [[control-tower-v02]]
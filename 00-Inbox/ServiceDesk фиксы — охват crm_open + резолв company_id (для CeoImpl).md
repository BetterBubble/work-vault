---
title: ServiceDesk фиксы — охват crm_open + резолв company_id (для CeoImpl)
type: note
permalink: tacticum/00-inbox/service-desk-fiksy-okhvat-crm-open-rezolv-company-id-dlia-ceo-impl
tags:
- helm
- servicedesk
- ceo
- fix
- crm_open
- company_id
- implementer
---

Две правки в витрину ServiceDesk. Ветка `feature/servicedesk-ceo` (worktree `~/tacticum-worktrees/helm-ceo`) — продолжай там. Репо `/Users/bubblemac/tacticum/helm`. Только правки, вид не ломать.

## Fix 1 — охват вида → crm_open (без naumen)
Решение оператора: ServiceDesk = product-support обращения **crm_open (782)**, БЕЗ naumen (`sd_tasks`). Причина: у naumen нет полей продукт/клиент/тариф/тип, приоритет = дефолт «1» (12402 → забивал разрез), 90% «Завершена» (закрытая история). Проверено на данных.
- В `routers/servicedesk.py`: базовый фильтр (`_base_conds`/`_signal_cond`) → **scope к `SdRequest.source == ROW_CRM_OPEN`** ('crm_open') для ВСЕХ разрезов вида (overview/stuck/timeline/facets/clients-money/requests). Тогда: total **782**, by_priority **P3 663 / P2 6 / P1 3 / Не указан 110**, by_status по обращениям.
- Нюанс: `by_category` (это поле naumen) для crm_open пустое → убери «По категории» из вида ИЛИ оставь пустым («нет данных»). Осмысленные разрезы crm_open: статус/группа статуса/приоритет/тип/продукт/тариф/команда.
- `signal_only`-параметр можно оставить, но он станет moot (crm_open и так «сигнал»). by_type/product/tariff уже crm_open — не сломай.

## Fix 2 — резолв sd_request.company_id (by_client / clients-money сейчас ПУСТЫ)
Дефект (нашёл Verify на сервере): `sd_request.company_id` не проставлен ни у одной из 782. Причина: `sd_request.client` = crm_open код-префикс («AKR - Авиакомпания Россия»), а company-алиасы в БД = crm_deals формат («транснефть (пао), ИНН») → exact-резолв матчит 0.
- Фикс: резолвить `company_id` **ФАЗЗИ** — по образцу `ingest/client_money.py` (`norm_client`/`_CompanyResolver`: стрип орг-форм + **код-префикса crm_open** типа «AKR - » + подстрочный матч имени к канон-компании), НЕ exact-alias. Адаптируй под формат crm_open client.
- Порядок сидов (Verify): companies-canon → servicedesk → резолв `company_id` ПОСЛЕ servicedesk. Убедись, что refresh-скрипт/пайплайн проставляет company_id после наличия и обращений, и канона.
- Приёмка: сколько из 782 резолвится (ждём заметную долю), `by_client` непустой, `clients-money` непустой (ЦБ/Транснефть/Газпромбанк и т.д.).

## Приёмка/отчёт
- `uv run pytest -q` зелёно, ruff+mypy+web tsc чисто. Коммить в `feature/servicedesk-ceo`, НЕ пушь.
- Отчёт ТЕКСТОМ to:main (НЕ idle): что менял по Fix1/Fix2, как резолвишь company_id, **число passed**, ожидаемое покрытие резолва.

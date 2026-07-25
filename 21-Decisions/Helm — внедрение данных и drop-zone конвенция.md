---
title: Helm — внедрение данных и drop-zone конвенция
type: decision
permalink: tacticum/21-decisions/helm-vnedrenie-dannykh-i-drop-zone-konventsiia
tags:
- helm
- control-tower
- data
- ingestion
- wave-1a
- decision
---

Решения по внедрению первых реальных данных в Helm (backend Волны 1a). Продолжение [[2026-07-03 — Helm backend Волны 1a собран]].

## Инвентарь данных (в `helm/data/`, весь каталог в .gitignore)

- [material] `Обезличенные данные/` — курированный обезличенный демо-набор под конвейер: goals(5)/blocks(6)/teams(10)/sales(7)/jira(5 эпиков+10 задач)/reference. Заложены genesis-мержи (ГОСТ: G2∪S1∪MAIL-200; One 1.5: G5∪S2∪ONE-100) и разрывы §6.1 (S5/S6, BE-501/CS-401, CS/Largo убыточны). Безопасен. #dataset
- [material] `repos/` — ⚠ РЕАЛЬНЫЙ git-снимок ИВА, НЕ обезличено: ~435 коммитов, ~145 merge, 8 репо. Настоящие email @iva.ru, реальные Jira-проекты IVAONE/VCSWEB/P8. Конфиденциальность уровня data room. Волна 1b (M1/M2/M3). НЕ джойнится с обезличенным (разные Jira). #red #wave-1b
- [material] `confluence/` — 3 страницы (IVA ADP, ВКС 1.5, Профиль 1.5), контекст §7 требование↔Jira↔клиент. #wave-1b
- [material] repomix (код-снимки) — придут позже, лягут в `data/`, питают RE-обогащение 1b. #wave-1b

## Решения

- [decision] Всё преобразование формы источника — в адаптере слоя `ingest/` (`csv_source.py`), домен/валидаторы НЕ гнём под данные («придут реальные данные → меняется только адаптер»). #adapter
- [decision] Курированный датасет = дефолтный демо-сид, случайный генератор остаётся для масштаба (сосуществуют, `seed_db.py --source csv|synthetic`). #scope
- [decision] Внедряем только 1a-срез; git/crm/economics/sensitive/repos/confluence — парковка 1b. #scope
- [decision] Расхождения курированного sales vs наш код: стадии RU→EN (адаптер нормализует синонимы), `owner`→`owner_email`, `depends_on`=эпик-ключи→initiative_id (резолвит loader через genesis-lookup). Наш EN-энум стадий — реконструкция; реальные данные по-русски → адаптер маппит. #reconcile
- [decision] Разрывы §6.1 на курированных данных осмысленны ТОЛЬКО после применения genesis-merge (без merge goal/sales/epic отдельны, сигналов нет → всё флагается). После merge связка держится на `merged_from`. #calc
- [decision] Дедлайн/важность после merge: `_bound_sales` расширен на sales-ссылки из `merged_from` (`sales_refs_of`), иначе смёрженная инициатива теряла дедлайн продажи. #calc

## Drop-zone конвенция (куда класть материалы)

- [convention] Данные для работы (сырьё источников) → `<репо>/data/<источник>/`, весь `data/` в .gitignore (кроме README). Рядом с пайплайном, не коммитятся. #convention
- [convention] Осмысление (решения/планы/итоги) → basic-memory vault (`02/03/04`). #convention
- [convention] Секреты/реальные PII → вне git, 🔴-контур. #convention

## Отношения
- relates_to [[2026-07-03 — Helm backend Волны 1a собран]]
- implements [[control-tower-v02]]

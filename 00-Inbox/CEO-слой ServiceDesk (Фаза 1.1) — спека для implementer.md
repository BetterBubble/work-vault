---
title: CEO-слой ServiceDesk (Фаза 1.1) — спека для implementer
type: note
permalink: tacticum/00-inbox/ceo-sloi-service-desk-faza-1.1-speka-dlia-implementer
tags:
- helm
- servicedesk
- ceo
- phase-1
- spec
- implementer
---

Спека инкремента «CEO-слой» витрины ServiceDesk (`sd_request`). Стратегический взгляд ГД IVA поверх операционного вида. Ветка: `feature/servicedesk-ceo` от `main` (worktree `~/tacticum-worktrees/helm-ceo`). Только ДОБАВЛЕНИЯ, существующее не удалять. Репо `/Users/bubblemac/tacticum/helm`.

## Задачи

### 1. По клиентам + сортировка (приоритет оператора)
- В `/api/servicedesk/overview` добавить `by_client: list[Bucket]` (топ клиентов по числу обращений; резолв через `Company` как в `/stuck`).
- Web: секция «По клиентам» + **сортируемые таблицы** клиентов (по имени / числу обращений / деньгам). Клиенты должны быть на виду. Существующий `clients-money` (E) уже есть — его НЕ переделывать, но сделать его таблицу тоже сортируемой.

### 2. A — Продукт × ЭКОНОМИКА (портфельный оверлей)
- Новая витрина-таблица `product_economics` (канон-продукт, фот, выручка, прибыль, маржа, решение) — сид из `data/real/data-room/economics/product_economics_37m.csv` через новый ingest (образец — `ingest/client_money.py` + существующий `ingest/economics.py::load_product_economics`, там уже парсинг). Миграция (down_revision = текущий head).
- Endpoint `/api/servicedesk/product-economics`: агрегат застрявших по продукту (сырой `sd_request.product`) → маппинг raw→канон через `domain.product_catalog.ProductCatalog.canonical` → джойн с `product_economics`. Верни на продукт: stuck-count, tickets, маржа, выручка, **портфельное Решение**.
- Web: панель «Продукт: боль поддержки × экономика» (боль + маржа/выручка + бейдж Решения Инвестировать/Оптимизировать/Сократить).

### 3. C — Тренд «обращений на пользователя»
- `/api/servicedesk/timeline` уже возвращает `count` + `users` (накопительная база). Добавить в ответ `per_user` (count/users, guard деление на 0) ИЛИ считать в web.
- Web: линия «обращений на пользователя» во времени (рядом с абсолютным трендом).

### 4. D — Концентрация (Pareto)
- Web (данные из существующего `/stuck` by_product/by_client): накопительная доля топ-N + KPI концентрации («топ-1 продукт = 66% висяков»). Роутер менять не обязательно, если данных хватает.

### 5. Фикс приоритетов P1/P2/P3
- Web: метки бакета `by_priority`: `1→P1`, `2→P2`, `3→P3`, пусто/`—`→«Не указан». В данных только 1/2/3/пусто (подтверждено). Не выдумывать другие.

### E (деньги под риском) — НЕ трогать, уже есть (clients-money).

## Приёмка
- `uv run pytest -q` зелёно (юниты: by_client bucket, product-economics джойн на фикстуре, timeline per_user). ruff+mypy+web tsc чисто. Миграция линейна (один head).
- Существующий вид цел (все прежние секции на месте).
- Коммить в `feature/servicedesk-ceo`, НЕ пушить.

## Отчёт (текстом лиду, НЕ idle)
Изменённые/новые файлы, id миграции+down_revision, как реализовал product_economics-витрину и raw→канон, результат pytest (число passed). После — верификация на dev-сервере отдельным шагом (deploy + сид product_economics + сверка).
